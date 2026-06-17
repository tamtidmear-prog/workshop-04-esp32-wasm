# desk-pet GIF character pack สูตรโกง

> สร้าง→verify→flash→PR pack ลง jc3248-pet (workshop-04-esp32-wasm) — ทุกคำสั่งจาก session จริง 2026-06-17

---

## 🎨 สร้าง GIF เอง (Pillow → MIT, no sprite ใคร)

```python
# 96×100 GIF89a, supersample 3x แล้ว downscale = ขอบเนียน
from PIL import Image, ImageDraw, ImageFilter
W, H, SS = 96, 100, 3
img = Image.new("RGBA", (W*SS, H*SS), (13,17,23,255))   # bg #0D1117
# ...วาดด้วย ImageDraw บน *SS แล้ว resize ตอนท้าย...
frame = img.convert("RGB").resize((W, H), Image.LANCZOS)
```

```python
# save: shared palette/state → ไฟล์เล็ก + loop เนียน (อย่า dither)
big = Image.new("RGB", (W, H*len(frames)))
for i, f in enumerate(frames): big.paste(f, (0, i*H))
pal = big.quantize(colors=64, method=Image.MEDIANCUT)
qf = [f.quantize(palette=pal, dither=Image.NONE) for f in frames]
qf[0].save("idle.gif", save_all=True, append_images=qf[1:],
           duration=delays, loop=0, disposal=2, optimize=True)
```

States desk-pet (7): `sleep idle busy attention celebrate dizzy heart`

## 🖼️ แปลง sprite ที่มีอยู่ → pack (scale 2x pixel-art)

```python
im = Image.open("Sprite-1.png").convert("RGBA").resize((96,96), Image.NEAREST)  # crisp
canvas = Image.new("RGBA", (96,100), (13,17,23,255)); canvas.alpha_composite(im,(0,2))
```

## 📦 build LittleFS storage เอง — ไม่ต้อง ESP-IDF

```bash
python3 -m pip install --user littlefs-python
```
```python
from littlefs import LittleFS                       # app auto-discovers pack
fs = LittleFS(block_size=4096, block_count=0x300000//4096)   # 3MB @ 0x290000
fs.makedirs("/characters/vialumen-pet", exist_ok=True)
for fn in os.listdir(PACK):                          # .gif + manifest.json only
    with fs.open(f"/characters/vialumen-pet/{fn}","wb") as o: o.write(open(f"{PACK}/{fn}","rb").read())
open("docs/vialumen-storage.bin","wb").write(bytes(fs.context.buffer))
```

## 🔍 verify (3 ชั้น — อย่าเดา)

```bash
# 1) GIF decode ใน decoder จริง (ตัวเดียวกับ device)
node -e 'const G=require("./gifdec.js");const fs=require("fs");(async()=>{const M=await G();
  const b=new Uint8Array(fs.readFileSync("gifs/x/idle.gif"));const p=M._malloc(b.length);M.HEAPU8.set(b,p);
  console.log(M._gif_open(p,b.length),M._gif_width()+"x"+M._gif_height());})()'
```
```python
# 2) storage mount กลับ — ไฟล์ครบ?
from littlefs import LittleFS
buf=open("docs/vialumen-storage.bin","rb").read()
fs=LittleFS(block_size=4096,block_count=len(buf)//4096,mount=False)
fs.context.buffer[:]=buf; fs.mount(); print(fs.listdir("/characters/vialumen-pet"))
```
```bash
# 3) flasher-check CI: offset-0 part ของทุก manifest ต้อง 0xE9
head -c1 docs/bootloader.bin | xxd -p          # = e9 ผ่าน · 0xff/02 = brick
```

## 🌐 download asset folder (Google Drive public)

```bash
python3 -m pip install --user gdown
gdown --folder "https://drive.google.com/drive/folders/<ID>" -O /tmp/out
```

## 📸 capture screen (headless, ไม่มี hardware)

```bash
python3 -m http.server 8109 --directory docs &      # serve
google-chrome --headless=new --no-sandbox --disable-gpu --hide-scrollbars \
  --screenshot=/tmp/shot.png --window-size=470,470 --virtual-time-budget=7000 \
  "http://localhost:8109/preview/index.html?embed=1&pack=vialumen-pet"
```

## 🔁 fork + PR workflow (gh)

```bash
git remote -v                                        # origin=upstream, fork=ของเรา
git checkout -b my-fix origin/main                   # branch จาก upstream ล่าสุดเสมอ
git push fork my-fix
gh pr create --repo OWNER/REPO --head USER:my-fix --base main --title "..." --body "..."
gh pr view <N> --repo OWNER/REPO --json state,mergedAt   # เช็คสถานะหลังสร้าง!
```

## ⚡ ลัด

| ทำอะไร | คำสั่ง |
|--------|--------|
| frame size เช็ค | `python3 -c "from PIL import Image;i=Image.open('x.gif');print(i.size,i.n_frames)"` |
| magic byte | `head -c1 f.bin \| xxd -p` (e9=app/boot, aa=parttable, 02=littlefs) |
| montage ดู sprites | `montage *.png -tile 7x1 -geometry 96x100+4+4 -background '#0D1117' out.png` |
| pack json format | `{id,name,kind:"pet",manifest,previewDir,states,blurb,submission}` |
| PR สถานะ | `gh pr view N --repo R --json state,mergedAt` |

## ⚠️ trap ที่เจอจริง

| trap | วิธีเลี่ยง |
|------|-----------|
| preview เป็นกบ (bufo) | pack id ต้องตรงทุกชั้น: `packs/<id>.json` + preview registration + `gifs/<id>/` + `/characters/<id>/` — ไม่ตรง = fallback เงียบ ไม่ error |
| structure เปลี่ยนกลางคัน | re-grep ว่าของจริงต่อสายยังไงก่อนแก้ — workshop เปลี่ยน static→data-driven 3 รอบ, PR ถูกปิดเพราะวิธีล้าสมัย |
| PR ปิดไม่รู้ตัว | เช็ค `gh pr view N --json state` หลัง push เสมอ — อย่ารายงาน "รอ merge" โดยไม่ verify |
| `rm -rf` ถูก hook block | ใช้ `mv <target> /tmp/` แทน |
| `git push --force` block | push branch ใหม่แทน force (golden rule ห้าม force) |
| gdown ดึงไม่ครบ | limit 50 ไฟล์/folder — folder ใหญ่/nested ต้อง target subfolder หรือ rclone |
| GIF ไฟล์ใหญ่เกิน | `quantize(colors=64)` shared palette + `dither=NONE` (glow gradient = สีบาน) |
| เดา ownership repo | git remote `origin` ≠ คน — verify `gh repo view`, อย่าผูกชื่อเจ้าของโดยไม่มีหลักฐาน |

---

🤖 ViaLumen — Oracle · จาก session desk-pet pack กับพี่นัท 🕯️
