#!/usr/bin/env python3
# File: grid_planner.py
# Build a 3xN Instagram-like grid preview from a folder and (optionally) export a posting calendar CSV.

import argparse, os, glob, math
from datetime import date, timedelta
from typing import List, Tuple, Optional
from dateutil.parser import isoparse
from PIL import Image, ImageDraw, ImageFont

COLS = 3

def list_images(folder: str) -> List[str]:
    exts = ("*.jpg","*.jpeg","*.png","*.webp","*.JPG","*.JPEG","*.PNG","*.WEBP")
    files = []
    for e in exts:
        files += sorted(glob.glob(os.path.join(folder, e)))
    return files

def hex_to_rgb(h: str) -> Tuple[int,int,int]:
    h = h.strip().lstrip("#")
    if len(h)==3: h = "".join(ch*2 for ch in h)
    return tuple(int(h[i:i+2],16) for i in (0,2,4))

def cover_square(img: Image.Image, size: int) -> Image.Image:
    # center-crop to square, then resize
    w,h = img.size
    if w == h:
        return img.resize((size,size), Image.LANCZOS)
    if w > h:
        x = (w - h)//2
        img = img.crop((x,0,x+h,h))
    else:
        y = (h - w)//2
        img = img.crop((0,y,w,y+w))
    return img.resize((size,size), Image.LANCZOS)

def load_image(fp: str) -> Image.Image:
    im = Image.open(fp)
    if im.mode not in ("RGB","RGBA"):
        im = im.convert("RGB")
    return im

def place_logo(cell: Image.Image, logo_path: str, scale: float=0.12, margin: int=16, corner: str="bottom-right") -> Image.Image:
    if not logo_path or not os.path.exists(logo_path):
        return cell
    logo = Image.open(logo_path).convert("RGBA")
    max_w = int(cell.width * scale)
    r = max_w / logo.width
    logo = logo.resize((max(1,int(logo.width*r)), max(1,int(logo.height*r))), Image.LANCZOS)
    x,y = margin, margin
    if "right" in corner: x = cell.width - logo.width - margin
    if "bottom" in corner: y = cell.height - logo.height - margin
    out = cell.convert("RGBA")
    out.alpha_composite(logo, (x,y))
    return out

def draw_badge(cell: Image.Image, text: str, font_path: Optional[str], bg=(0,0,0,150), fg=(255,255,255,255)) -> Image.Image:
    draw = ImageDraw.Draw(cell)
    # choose a small bold-ish font
    font = None
    if font_path and os.path.exists(font_path):
        try: font = ImageFont.truetype(font_path, size=int(cell.width*0.08))
        except Exception: font = None
    if font is None:
        for cand in ["DejaVuSans.ttf","arial.ttf","Arial.ttf"]:
            try:
                font = ImageFont.truetype(cand, size=int(cell.width*0.08))
                break
            except Exception:
                continue
        if font is None:
            font = ImageFont.load_default()

    tw = draw.textlength(text, font=font)
    th = font.getbbox(text)[3] - font.getbbox(text)[1]
    pad_x, pad_y, radius = 14, 8, 14
    bw, bh = int(tw + 2*pad_x), int(th + 2*pad_y)
    x = 12
    y = 12
    # rounded rectangle
    badge = Image.new("RGBA", (bw, bh), (0,0,0,0))
    bd = ImageDraw.Draw(badge)
    bd.rounded_rectangle([0,0,bw,bh], radius=radius, fill=bg)
    cell = cell.convert("RGBA")
    cell.alpha_composite(badge, (x,y))
    draw = ImageDraw.Draw(cell)
    draw.text((x + pad_x, y + pad_y - 2), text, font=font, fill=fg)
    return cell

def ig_like_order(files: List[str]) -> List[str]:
    """
    Instagram grid shows newest first (top-left), filling rows left->right, then next rows.
    If you want to preview how your feed LOOKS AFTER posting the listed files (in order),
    you need to reverse the list so that the last file ends up top-left.
    """
    return list(reversed(files))

def build_grid(files: List[str], cell: int, gutter: int, bg_rgb: Tuple[int,int,int],
               logo: Optional[str], logo_scale: float, logo_corner: str, numbering: bool,
               font_path: Optional[str]) -> Image.Image:
    n = len(files)
    rows = math.ceil(n / COLS)
    W = COLS * cell + (COLS + 1) * gutter
    H = rows * cell + (rows + 1) * gutter
    canvas = Image.new("RGB", (W,H), bg_rgb)

    for idx, fp in enumerate(files):
        row = idx // COLS
        col = idx % COLS
        x = gutter + col * (cell + gutter)
        y = gutter + row * (cell + gutter)
        im = cover_square(load_image(fp), cell)
        if logo:
            im = place_logo(im, logo, scale=logo_scale, margin=max(10, gutter//2), corner=logo_corner)
        if numbering:
            im = draw_badge(im, f"#{idx+1}", font_path, bg=(0,0,0,150), fg=(255,255,255,255))
        canvas.paste(im.convert("RGB"), (x,y))
    return canvas

def make_calendar(csv_path: str, files: List[str], start_iso: str, per_week: int):
    if not csv_path: return
    start = isoparse(start_iso).date()
    step = max(1, 7 // max(1, per_week))  # spread across week
    cur = start
    with open(csv_path, "w", encoding="utf-8") as f:
        f.write("date,filename,notes\n")
        for i, fp in enumerate(files):
            f.write(f"{cur.isoformat()},{os.path.basename(fp)},\n")
            cur = cur + timedelta(days=step)

def main():
    ap = argparse.ArgumentParser(description="Create a 3-column Instagram grid preview + optional CSV calendar.")
    ap.add_argument("--images", required=True, help="Folder with images")
    ap.add_argument("--output", default="grid_preview.jpg")
    ap.add_argument("--cell", type=int, default=1080, help="Cell size in pixels (square)")
    ap.add_argument("--gutter", type=int, default=16, help="Gutter between cells (px)")
    ap.add_argument("--bg", default="#111827", help="Background color (hex)")
    ap.add_argument("--logo", default=None, help="Optional logo PNG (with alpha)")
    ap.add_argument("--logo-scale", type=float, default=0.12, help="Logo width relative to cell width")
    ap.add_argument("--logo-corner", choices=["top-left","top-right","bottom-left","bottom-right"], default="bottom-right")
    ap.add_argument("--numbering", action="store_true", help="Show #index badge on cells")
    ap.add_argument("--font", default=None, help="Font path for numbering badge")

    ap.add_argument("--ig-order", action="store_true", help="Reverse list to simulate newest at top-left")
    ap.add_argument("--limit", type=int, default=0, help="Use only first N files after ordering (0 = all)")

    # calendar
    ap.add_argument("--calendar", default=None, help="Output CSV path for posting plan (optional)")
    ap.add_argument("--start", default=None, help="Start date YYYY-MM-DD for calendar")
    ap.add_argument("--per-week", type=int, default=4, help="Posts per week (calendar)")

    args = ap.parse_args()

    files = list_images(args.images)
    if not files:
        raise SystemExit("No images found in --images")

    if args.ig_order:
        files = ig_like_order(files)

    if args.limit and args.limit > 0:
        files = files[:args.limit]

    bg_rgb = hex_to_rgb(args.bg)
    grid = build_grid(
        files, cell=args.cell, gutter=args.gutter, bg_rgb=bg_rgb,
        logo=args.logo, logo_scale=args.logo_scale, logo_corner=args.logo_corner,
        numbering=args.numbering, font_path=args.font
    )
    grid.save(args.output, quality=95, subsampling="4:2:0", optimize=True)
    print(f"✅ Grid saved to {args.output} ({len(files)} images, {math.ceil(len(files)/COLS)} rows)")

    if args.calendar:
        if not args.start:
            raise SystemExit("--start YYYY-MM-DD is required when using --calendar")
        make_calendar(args.calendar, files, args.start, args.per_week)
        print(f"📅 Calendar saved to {args.calendar}")

if __name__ == "__main__":
    main()
