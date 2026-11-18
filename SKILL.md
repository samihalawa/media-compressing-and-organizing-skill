---
name: media-compressing-and-organizing-skill
description: Compress, deduplicate, rename, index and organize video and audio files (downloads, courses, lessons, recordings, podcasts, archives) with ffmpeg, safely and in bulk, and optionally attach them to Apple Notes. Use when asked to compress, shrink, re-encode, clean up, organize, rename, dedupe, catalogue or move videos or audios, free disk space taken by media, build a course or audio archive, or put media files into Notes.
---

# Media Compressing And Organizing

Work on the machine where the files live (local shell or a remote-shell tool),
with inline commands. Never delete an original before its replacement is
verified. Report `done X of N` at the end.

## 1. Inventory The Whole Folder First

```bash
DIR="$HOME/Downloads/SomeFolder"            # target folder
# folder shape: one line per directory with its file count (run again at the end)
find "$DIR" -type d | sort | while IFS= read -r d; do printf '%6d  %s\n' "$(find "$d" -maxdepth 1 -type f | wc -l)" "$d"; done
# extensions and total size
find "$DIR" -type f | sed 's/.*\.//' | sort | uniq -c | sort -rn
du -sh "$DIR"; df -h "$DIR" | tail -1
command -v ffmpeg ffprobe; ffmpeg -hide_banner -encoders | grep -E 'libx264|libx265|videotoolbox|libopus|aac'
```

- Separate real media from leftovers: `*.part`, `*.part-Frag*`, `*.ytdl`
  (unfinished downloads), hidden `.*.compressing-*` temp files and work
  folders from killed compression runs, `__pycache__`, `.DS_Store`. List them
  in the report; they are not library items.
- Count main-folder media, subfolder media (`_duplicates/`, `_incoming/`,
  work folders) and anything unreadable. That count is N.
- Check free disk space before any bulk re-encode: each running job needs room
  for one extra output file at the same time.

## 2. Choose The Encoding

| Content | Default | When to change |
|---|---|---|
| Video, any | H.264 `libx264 -crf 28 -preset veryfast`, AAC 128k, `+faststart` | Plays everywhere; good size/quality balance |
| Video, keep quality | `-crf 23 -preset medium` | User asks for best or near-original quality |
| Video, fastest on Apple Silicon | `hevc_videotoolbox -q:v 60 -tag:v hvc1` | Huge batches where time matters more than size |
| Speech audio (lessons, podcasts, calls) | `-vn -c:a aac -b:a 64k -ac 1` → `.m4a` | Opus `libopus -b:a 32k` → `.opus` for smallest size |
| Music | `-vn -c:a aac -b:a 160k` → `.m4a` | Keep lossless originals if the user collects them |

Use `-preset veryfast` unless the user asks for best quality; slower presets
take two to three times longer for little gain.

## 3. Compress Safely

How the pipeline works, per file:

1. Encode to a hidden temp file next to the source
   (`.name.compressing-<id>.mp4`) with `-n`, so nothing is ever overwritten
   mid-encode.
2. Keep the first video stream, every audio stream, metadata and chapters
   (`-map 0:v:0 -map 0:a? -map_metadata 0 -map_chapters 0`).
3. Pad odd dimensions to even (`pad=ceil(iw/2)*2:ceil(ih/2)*2`), which H.264
   requires, and move the index to the front (`+faststart`) so the file
   streams and seeks immediately.
4. Verify the result: ffprobe duration within 1.5 s of the source, and size at
   most 90% of the original. If either check fails, delete the temp and keep
   the original untouched (already-efficient files are not made worse).
5. Atomically move the temp into place as `<name>.mp4` and only then remove a
   different-extension original (`.mkv`, `.mov`, `.webm`, ...).
6. Run at most 2 encodes at once with 2 threads each, so the machine stays
   usable; run long batches in the background with a log and poll it.

```bash
DIR="$HOME/Downloads/SomeFolder"; CRF=28; LOG=/tmp/media-compress.log
cd "$DIR" || exit 1
# stale temps from killed runs (review, then remove)
find . -name '.*.compressing-*' -mmin +60 -print
nohup sh -c '
find . -maxdepth 1 -type f \( -iname "*.mp4" -o -iname "*.mkv" -o -iname "*.mov" -o -iname "*.webm" -o -iname "*.avi" -o -iname "*.m4v" -o -iname "*.flv" -o -iname "*.wmv" \) ! -name ".*" -print0 |
xargs -0 -P 2 -I{} sh -c '"'"'
src="$1"; crf="$2"; out="${src%.*}.mp4"; tmp="$(dirname "$src")/.$(basename "${src%.*}").compressing-$$.mp4"
ffmpeg -hide_banner -nostdin -loglevel error -n -threads 2 -i "$src" -map 0:v:0 -map 0:a? -map_metadata 0 -map_chapters 0 \
  -c:v libx264 -crf "$crf" -preset veryfast -threads 2 -filter_threads 1 -vf "pad=ceil(iw/2)*2:ceil(ih/2)*2" \
  -c:a aac -b:a 128k -movflags +faststart "$tmp" || { rm -f "$tmp"; echo "FAIL $src"; exit 0; }
d1=$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$src"); d2=$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$tmp")
s1=$(wc -c < "$src"); s2=$(wc -c < "$tmp")
if awk -v a="$d1" -v b="$d2" "BEGIN{exit !(a-b<1.5 && b-a<1.5)}" && [ "$s2" -lt $((s1/10*9)) ]; then
  mv -f "$tmp" "$out"; [ "$out" != "$src" ] && rm -f "$src"; echo "OK $s1 $s2 $out"
else rm -f "$tmp"; echo "KEEP $src"; fi
'"'"' _ {} '"$CRF"'
echo DONE' > "$LOG" 2>&1 &
# poll until finished, then summarize
until grep -q DONE "$LOG"; do sleep 30; grep -c . "$LOG"; done
grep -c '^OK' "$LOG"; grep -c '^KEEP' "$LOG"; grep '^FAIL' "$LOG"
awk '/^OK/{a+=$2;b+=$3} END{printf "saved %.1f GB\n",(a-b)/1e9}' "$LOG"
```

Variants (same verify-then-replace rule):

- Speech audio: replace the encode line with
  `ffmpeg ... -i "$src" -vn -map_metadata 0 -c:a aac -b:a 64k -ac 1 "$tmp"`,
  set `out="${src%.*}.m4a"` and a `.m4a` temp, and select `*.mp3 *.wav *.m4a *.aac *.flac *.ogg *.opus` files.
- Extract audio from lecture videos while keeping the video:
  `ffmpeg -n -i in.mp4 -vn -c:a aac -b:a 64k -ac 1 in.m4a`.
- If a downloader script already compresses each finished file (yt-dlp then
  ffmpeg), do not re-encode its outputs again; only process files that are
  not yet H.264/HEVC or whose bitrate is clearly high:
  `ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,bit_rate -of csv=p=0 file`.

## 4. Deduplicate Without Deleting

Byte-identical copies go to `_duplicates/`; nothing is deleted. Same-title
files with different bytes (different quality or cut) are listed for the user,
not moved.

```bash
cd "$DIR" && mkdir -p _duplicates
find . -maxdepth 1 -type f ! -name '.*' -print0 | xargs -0 shasum -a 256 | sort |
awk 'seen[substr($0,1,64)]++ {print substr($0,67)}' | while IFS= read -r f; do mv -n "$f" _duplicates/; echo "dup: $f"; done
# near-duplicates by title (ignore a leading id/hash prefix)
ls | sed -E 's/^[0-9a-f]{8,}-//' | sort | uniq -d
```

- A re-encoded copy (for example a leftover in a compression work folder) has
  different bytes but the same duration as its source: treat a duration match
  within 1.5 s as a copy candidate.
- Equal duration alone is not proof (sites often trim clips to the same
  length). Confirm by comparing small frames at 20%, 50% and 80%:
  `ffmpeg -v error -ss T -i f -frames:v 1 -vf scale=16:16,format=gray -f rawvideo -`;
  a mean absolute byte difference under about 12 means the same video, while
  different videos score 50 or more.
- For the user's working set (a Notes folder, playlist or index), keep one
  copy per video, the largest or best-named, without deleting files on disk.

## 5. Organize Into A Flat, Named Library

Default layout for courses, lessons and audio collections:

- one folder per source or platform, kept flat (no or few subfolders);
- slug file names keyed by platform, course and lesson:
  `platform-course-NN-lesson-title.ext`; keep an existing id or hash prefix
  when names would otherwise collide;
- one `INDEX.md` per folder listing original title, new file name, source
  URL, duration and size, so new courses, modules or seasons can be appended.

Always produce the rename mapping first, check it, then apply with `mv -n`
(never overwrite) and write the index from the same mapping.

```bash
slug() { printf '%s' "$1" | iconv -f UTF-8 -t ASCII//TRANSLIT 2>/dev/null | tr '[:upper:]' '[:lower:]' | sed -E 's/[^a-z0-9]+/-/g; s/^-+|-+$//g' | cut -c1-90; }
# mapping: old<TAB>new (edit PREFIX / numbering to the real platform, course and lesson order)
PREFIX="platform-course"; i=0
ls -tr | grep -iE '\.(mp4|m4a|mp3|mov|mkv|webm|opus|wav)$' | while IFS= read -r f; do
  i=$((i+1)); printf '%s\t%s-%02d-%s.%s\n' "$f" "$PREFIX" "$i" "$(slug "${f%.*}")" "${f##*.}"
done > /tmp/media-rename.tsv
column -t -s $'\t' /tmp/media-rename.tsv | head -40            # review
while IFS=$'\t' read -r old new; do mv -n "$old" "$new"; done < /tmp/media-rename.tsv
{ echo '| # | File | Original title | Duration | Size | Source |'; echo '|---|---|---|---|---|---|'
  n=0; while IFS=$'\t' read -r old new; do n=$((n+1))
    d=$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$new" 2>/dev/null | awk '{printf "%d:%02d:%02d",$1/3600,$1%3600/60,$1%60}')
    s=$(du -h "$new" | cut -f1); echo "| $n | $new | ${old%.*} | $d | $s | |"
  done < /tmp/media-rename.tsv; } > INDEX.md
```

Fill the Source column from the download log, playlist or course page when
known. When the folder grows later, append new rows instead of rebuilding.

## 6. Attach Media To Apple Notes (macOS, Optional)

AppleScript can create notes and attach files directly; this is faster and
more reliable than clicking through the Notes UI. Two Notes quirks must be
handled, or every note looks duplicated:

- `make new attachment` inserts the file twice. After each attach, delete the
  extra copies until the attachment count equals the number of files added.
- Setting both `name` and `body` prints the title twice. Set only
  `body:"<h1>Title</h1>"`; Notes takes the title from the first line.

Put the notes in their own folder, deduplicate the files first (step 4, plus
same-title copies and compressed copies of the same video), and test on one
file before the batch.

```bash
FOLDER="My videos"
osascript -e 'on run argv' -e 'tell application "Notes"' \
  -e 'if not (exists folder (item 1 of argv)) then make new folder with properties {name:(item 1 of argv)}' -e 'end tell' -e 'end run' "$FOLDER"
# create one note, return its id
NID=$(osascript -e 'on run argv' -e 'tell application "Notes" to return id of (make new note at folder (item 1 of argv) with properties {body:("<h1>" & (item 2 of argv) & "</h1>")})' -e 'end run' "$FOLDER" "Videos 01")
# attach file number K (1-based within this note) and remove the duplicate copy
osascript -e 'on run argv' -e 'tell application "Notes"' -e 'set n to note id (item 1 of argv)' \
  -e 'make new attachment at end of attachments of n with data (POSIX file (item 2 of argv))' \
  -e 'set c to count of attachments of n' -e 'set e to (item 3 of argv) as integer' \
  -e 'repeat while c > e' -e 'delete attachment c of n' -e 'set c to count of attachments of n' -e 'end repeat' \
  -e 'return c' -e 'end tell' -e 'end run' "$NID" "$FILE" "$K"     # prints K when correct
# verify per note: attachments must equal files added
osascript -e 'on run argv' -e 'tell application "Notes"' -e 'set out to ""' -e 'repeat with n in notes of folder (item 1 of argv)' \
  -e 'set out to out & (name of n) & ": " & (count of attachments of n) & linefeed' -e 'end repeat' -e 'return out' -e 'end tell' -e 'end run' "$FOLDER"
```

- Open the folder in Notes and check visually that each note says "N videos"
  and shows each video once.
- Deleted notes go to Recently Deleted (kept 30 days, still counted in iCloud
  storage until then).
- Use as few notes as possible: Notes allows at most 250 attachments per note
  (error "Maximum attachments reached", -10000), so fill notes "Videos 1",
  "Videos 2", ... up to 250 each, oldest first. Notes cannot move attachments
  between notes, so plan the split before attaching rather than merging later.
- Pass about 20 files per `osascript` call (loop over `argv` inside one
  AppleScript run, keeping the running expected count) to cut per-call
  overhead; run it in the background with a log of expected vs actual counts,
  and stop if free disk space drops below a safety floor
  (`df -g "$DIR" | tail -1 | awk '{print $4}'`).
- `set name of note` changes the list title only; the heading inside the note
  keeps the original body text, so set the final title in `body` when creating.
- On the same APFS volume the attach is near-instant and uses no extra local
  space (file clone), but an iCloud Notes account uploads every byte: check
  iCloud storage before attaching tens of gigabytes. Exclude temp, partial and
  `_duplicates/` files unless the user asks for them.
- The first run may show a macOS Automation prompt for the shell to control
  Notes; it must be allowed once.

## 7. Content Limits

Never compress, rename, index, attach or upload files whose names or content
indicate sexual material involving minors. Leave them out, tell the user they
were excluded (by file id, without describing the content), and continue with
the rest.

## 8. Verify And Report

- Re-run the folder-shape listing from step 1 and compare with the start.
- Report: media found N; compressed / kept original / failed; duplicates
  moved; renamed; attached; excluded; space before and after; leftovers
  (partial downloads, stale temps) and what was done with them.
- Spot-check one output per batch: `ffprobe -v error -show_entries format=duration:stream=codec_name,width,height -of compact file`
  and open it once in the default player (`open file` on macOS).
- Count check: library files + `_duplicates/` + excluded + failed = N.
