# YouTube-automation-
An agent that runs my entire YouTube channel for Free 
name: render-short

on:
  workflow_dispatch:
    inputs:
      scenes_b64:
        description: "Base64 scenes JSON"
        required: true
      tag:
        description: "Release tag"
        required: true

permissions:
  contents: write

jobs:
  render:
    runs-on: ubuntu-24.04
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Install tools
        run: |
          sudo apt-get update -y
          sudo apt-get install -y ffmpeg fonts-montserrat
          fc-cache -f
          pip install edge-tts requests pillow
      - name: Write renderer
        run: |
          cat > render.py <<'PYEOF'
          #!/usr/bin/env python3
          """Money Uncovered Shorts renderer (free): edge-tts voice + free AI images + ffmpeg."""
          import asyncio, base64, json, math, os, random, re, subprocess, sys, time, urllib.parse
          import requests
          from PIL import Image, ImageDraw, ImageFont

          W, H, FPS, XF = 1080, 1920, 30, 0.30
          VOICES = ["en-US-AndrewNeural", "en-US-GuyNeural", "en-US-AriaNeural"]
          RATE = os.environ.get("TTS_RATE", "+25%")
          TRANS = ["fade", "wipeleft", "slideleft", "circleopen", "smoothleft",
                   "dissolve", "wiperight", "slideup", "circleopen", "fade"]
          STYLE = ("flat 2D vector infographic illustration, bold clean outlines, bright saturated "
                   "colors, simple clean background, educational explainer style, minimalist, ")
          PALETTE = [(11, 31, 58), (22, 60, 110), (90, 30, 120), (10, 90, 80), (120, 50, 30)]


          def run(cmd):
              subprocess.run(cmd, check=True)


          def probe(path):
              out = subprocess.check_output(["ffprobe", "-v", "error", "-show_entries",
                                             "format=duration", "-of", "csv=p=0", path])
              return float(out.decode().strip())


          # ---------------------------------------------------------------- voice
          async def _tts_one(text, voice, out):
              import edge_tts
              try:
                  com = edge_tts.Communicate(text, voice, rate=RATE, boundary="WordBoundary")
              except TypeError:
                  com = edge_tts.Communicate(text, voice, rate=RATE)
              events = []
              with open(out, "wb") as f:
                  async for ch in com.stream():
                      if ch["type"] == "audio":
                          f.write(ch["data"])
                      elif ch["type"] == "WordBoundary":
                          st = ch["offset"] / 1e7
                          events.append((st, st + ch["duration"] / 1e7, ch["text"]))
              if os.path.getsize(out) < 2000:
                  raise RuntimeError("empty audio")
              return events


          def tts(text, out):
              last = None
              for voice in VOICES:
                  for _ in range(2):
                      try:
                          return asyncio.run(_tts_one(text, voice, out))
                      except Exception as e:  # noqa
                          last = e
                          time.sleep(2)
              raise RuntimeError(f"TTS failed: {last}")


          # --------------------------------------------------------------- images
          def fallback_image(path, idx, keyword):
              c = PALETTE[idx % len(PALETTE)]
              img = Image.new("RGB", (W, H), c)
              d = ImageDraw.Draw(img)
              for y in range(H):
                  k = y / H
                  d.line([(0, y), (W, y)], fill=(int(c[0] + 60 * k), int(c[1] + 60 * k), int(c[2] + 80 * k)))
              d.ellipse([W // 2 - 260, H // 2 - 260, W // 2 + 260, H // 2 + 260], outline=(255, 213, 74), width=18)
              img.save(path, "JPEG", quality=92)


          def fit_image(path):
              img = Image.open(path).convert("RGB")
              s = max(W / img.width, H / img.height)
              img = img.resize((math.ceil(img.width * s), math.ceil(img.height * s)))
              l, t = (img.width - W) // 2, (img.height - H) // 2
              img.crop((l, t, l + W, t + H)).save(path, "JPEG", quality=93)


          def get_image(prompt, path, idx, keyword):
              full = prompt if prompt.lower().startswith("flat 2d") else STYLE + prompt
              for attempt in range(4):
                  try:
                      seed = random.randint(1, 999999)
                      url = ("https://image.pollinations.ai/prompt/" + urllib.parse.quote(full[:700]) +
                             f"?width={W}&height={H}&model=flux&nologo=true&seed={seed}")
                      r = requests.get(url, timeout=150)
                      if r.status_code == 200 and r.headers.get("content-type", "").startswith("image") \
                              and len(r.content) > 15000:
                          with open(path, "wb") as f:
                              f.write(r.content)
                          fit_image(path)
                          return True
                      print(f"image {idx}: attempt {attempt + 1} status {r.status_code}")
                  except Exception as e:  # noqa
                      print(f"image {idx}: attempt {attempt + 1} error {e}")
                  time.sleep(6 + attempt * 6)
              print(f"image {idx}: using fallback background")
              fallback_image(path, idx, keyword)
              return False


          # ------------------------------------------------------------- captions
          def ass_time(t):
              t = max(t, 0)
              return f"{int(t // 3600)}:{int((t % 3600) // 60):02d}:{t % 60:05.2f}"


          def clean(s):
              return re.sub(r"[{}\\]", "", s)


          def build_ass(narr, keyword, dur, events, path):
              if events and len(events) >= 3:
                  words = [(a, b, clean(w).upper()) for a, b, w in events if clean(w).strip()]
              else:
                  ws = re.sub(r"\s+", " ", clean(narr)).strip().split(" ")
                  wt = [max(2, len(w)) for w in ws]
                  span, t, words = dur * 0.96, 0.03, []
                  for w, x in zip(ws, wt):
                      d = span * x / sum(wt)
                      words.append((t, t + d, w.upper()))
                      t += d
              lines = []
              for i in range(0, len(words), 3):
                  chunk = words[i:i + 3]
                  for k, (st, en, _) in enumerate(chunk):
                      nxt = chunk[k + 1][0] if k + 1 < len(chunk) else (words[i + 3][0] if i + 3 < len(words) else dur)
                      txt = " ".join(
                          ("{\\c&H00D5FF&}" + w + "{\\c&HFFFFFF&}") if j == k else w
                          for j, (_, _, w) in enumerate(chunk))
                      lines.append(f"Dialogue: 0,{ass_time(st)},{ass_time(max(nxt, en))},Cap,,0,0,0,,{txt}")
              key = clean(keyword).upper()
              if key:
                  lines.append(f"Dialogue: 1,{ass_time(0)},{ass_time(dur)},Key,,0,0,0,,{{\\fad(150,0)}}{key}")
              head = f"""[Script Info]
          ScriptType: v4.00+
          PlayResX: {W}
          PlayResY: {H}
          WrapStyle: 2

          [V4+ Styles]
          Format: Name,Fontname,Fontsize,PrimaryColour,SecondaryColour,OutlineColour,BackColour,Bold,Italic,Underline,StrikeOut,ScaleX,ScaleY,Spacing,Angle,BorderStyle,Outline,Shadow,Alignment,MarginL,MarginR,MarginV,Encoding
          Style: Cap,Montserrat,78,&H00FFFFFF,&H00FFFFFF,&H00000000,&H80000000,1,0,0,0,100,100,0,0,1,7,2,2,60,60,430,1
          Style: Key,Montserrat,100,&H00FFFFFF,&H00FFFFFF,&H00000000,&H80000000,1,0,0,0,100,100,0,0,1,8,3,8,60,60,260,1

          [Events]
          Format: Layer,Start,End,Style,Name,MarginL,MarginR,MarginV,Effect,Text
          """
              with open(path, "w", encoding="utf-8") as f:
                  f.write(head + "\n".join(lines) + "\n")


          # ---------------------------------------------------------------- video
          def zoom_filter(i, frames):
              mode = i % 4
              cx, cy = "iw/2-(iw/zoom/2)", "ih/2-(ih/zoom/2)"
              if mode == 0:
                  z, x, y = f"1+0.22*on/{frames}", cx, cy
              elif mode == 1:
                  z, x, y = f"1.22-0.22*on/{frames}", cx, cy
              elif mode == 2:
                  z, x, y = "1.18", f"(iw-iw/zoom)*(1-on/{frames})", cy
              else:
                  z, x, y = "1.18", f"(iw-iw/zoom)*on/{frames}", cy
              return f"zoompan=z='{z}':x='{x}':y='{y}':d={frames}:s={W}x{H}:fps={FPS}"


          def main():
              scenes = json.loads(base64.b64decode(os.environ["SCENES_B64"]).decode("utf-8"))
              if isinstance(scenes, dict):
                  scenes = scenes.get("scenes", [])
              scenes = [s for s in scenes if str(s.get("narration", "")).strip()]
              if not scenes:
                  sys.exit("no scenes")
              os.makedirs("work", exist_ok=True)
              n, durs, segs, audios = len(scenes), [], [], []
              for i, sc in enumerate(scenes):
                  narr = re.sub(r"\s+", " ", str(sc["narration"])).strip()
                  kw = str(sc.get("keyword", ""))
                  a = f"work/a{i}.mp3"
                  events = tts(narr, a)
                  dur = probe(a)
                  durs.append(dur)
                  audios.append(a)
                  img = f"work/i{i}.jpg"
                  get_image(str(sc.get("image_prompt", narr)), img, i, kw)
                  ass = f"work/s{i}.ass"
                  build_ass(narr, kw, dur, events, ass)
                  frames = int(round((dur + (XF if i < n - 1 else 0)) * FPS))
                  seg = f"work/v{i}.mp4"
                  vf = f"scale={W * 2}:{H * 2},{zoom_filter(i, frames)},ass={ass},format=yuv420p"
                  run(["ffmpeg", "-y", "-loglevel", "error", "-i", img, "-vf", vf, "-frames:v", str(frames),
                       "-c:v", "libx264", "-preset", "veryfast", "-crf", "19", "-r", str(FPS), seg])
                  segs.append(seg)
                  print(f"scene {i + 1}/{n} done ({dur:.1f}s)", flush=True)
              total = sum(durs)
              # video: chain of crossfades
              if n == 1:
                  vid = segs[0]
              else:
                  cmd = ["ffmpeg", "-y", "-loglevel", "error"]
                  for s in segs:
                      cmd += ["-i", s]
                  parts, prev, off = [], "0:v", 0.0
                  for i in range(1, n):
                      off += durs[i - 1]
                      lab = f"x{i}"
                      parts.append(f"[{prev}][{i}:v]xfade=transition={TRANS[(i - 1) % len(TRANS)]}:"
                                   f"duration={XF}:offset={off:.3f}[{lab}]")
                      prev = lab
                  vid = "work/video.mp4"
                  cmd += ["-filter_complex", ";".join(parts), "-map", f"[{prev}]", "-c:v", "libx264",
                          "-preset", "veryfast", "-crf", "19", "-pix_fmt", "yuv420p", "-r", str(FPS), vid]
                  run(cmd)
              # audio: voice (+ optional quiet music if music.mp3 exists in the repo)
              cmd = ["ffmpeg", "-y", "-loglevel", "error", "-i", vid]
              for a in audios:
                  cmd += ["-i", a]
              ins = "".join(f"[{k + 1}:a]" for k in range(n))
              flt = f"{ins}concat=n={n}:v=0:a=1,aresample=44100[voice]"
              out_label = "[voice]"
              if os.path.exists("music.mp3"):
                  cmd += ["-stream_loop", "-1", "-i", "music.mp3"]
                  flt += (f";[{n + 1}:a]volume=0.12,atrim=0:{total:.2f},afade=t=out:st={max(total - 1.5, 0):.2f}:d=1.5[mus];"
                          "[voice][mus]amix=inputs=2:duration=first:normalize=0[mix]")
                  out_label = "[mix]"
              cmd += ["-filter_complex", flt, "-map", "0:v", "-map", out_label, "-c:v", "copy",
                      "-c:a", "aac", "-b:a", "192k", "-t", f"{total:.2f}", "-movflags", "+faststart", "video.mp4"]
              run(cmd)
              print(f"DONE video.mp4 {total:.1f}s")


          if __name__ == "__main__":
              main()
          PYEOF
      - name: Render video
        env:
          SCENES_B64: ${{ inputs.scenes_b64 }}
        run: python render.py
      - name: Publish video
        env:
          GH_TOKEN: ${{ github.token }}
          TAG: ${{ inputs.tag }}
        run: |
          gh release create "$TAG" video.mp4 --title "$TAG" --notes "Rendered short"
