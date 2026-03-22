# ts-tools

Two small macOS CLI tools for downloading and merging `.ts` video segment files.

| Tool | What it does |
|---|---|
| `download-ts` | Downloads a numbered sequence of `.ts` files from a URL |
| `combine-ts` | Merges `.ts` files into a single `.mp4`, grouped by filename prefix |
| `download-combine-ts` | Does both in one step — just provide the URL |

---

## Requirements

- macOS with Bash 3.2+ (the default on every Mac)
- [ffmpeg](https://formulae.brew.sh/formula/ffmpeg) — only required for `combine-ts`
- `curl` — pre-installed on macOS

Install ffmpeg via Homebrew if you don't have it:

```bash
brew install ffmpeg
```

---

## Installation

Download both scripts and place them somewhere on your `$PATH`:

```bash
# Move scripts to /usr/local/bin (or any directory on your PATH)
mv download-ts combine-ts download-combine-ts /usr/local/bin/

# Make them executable
chmod +x /usr/local/bin/download-ts /usr/local/bin/combine-ts /usr/local/bin/download-combine-ts
```

Verify the installation:

```bash
download-ts --help        # shows usage hint
combine-ts                # starts interactive mode
download-combine-ts       # starts the one-step workflow
```

---

## `download-ts`

Downloads a consecutive sequence of numbered `.ts` files from a web server.

### Usage

```
download-ts [url-of-first-ts-file] [output-dir]
```

Both arguments are optional — if omitted you will be prompted interactively.

### Examples

```bash
# Pass the URL directly
download-ts https://cdn.example.com/show/episode_001.ts

# Specify a custom output directory
download-ts https://cdn.example.com/show/episode_001.ts ~/Movies

# Run interactively (no arguments)
download-ts
```

### Interactive flow

```
⬇️   download-ts
─────────────────────────────────────────
▸ First file : episode_001.ts
▸ Base URL   : https://cdn.example.com/show/
▸ Name prefix: episode_  number width: 3

Last file number [Enter to auto-detect]:
```

**Entering a number** downloads files up to and including that number:

```
Last file number [Enter to auto-detect]: 24
```

**Pressing Enter** (leaving it blank) probes the server with `HEAD` requests to find the last available file automatically:

```
▸ Auto-detecting last file...
  Probing episode_001.ts ... ✔  (200)
  Probing episode_002.ts ... ✔  (200)
  Probing episode_003.ts ... ✖  (404) — stopped
✔ Detected last file: 002  (2 file(s))
```

After confirming the plan, files are downloaded one by one:

```
⬇️   Downloading
─────────────────────────────────────────
⬇  [1/3] episode_001.ts ...  ✔  12M
⬇  [2/3] episode_002.ts ...  ✔  11M
⬇  [3/3] episode_003.ts ...  ✔  13M

📋  Done
─────────────────────────────────────────
✔ 3 file(s) downloaded
▸ Files are in: /Users/you/Downloads
▸ Run combine-ts to merge them into an MP4.
```

### Notes

- Numbers can be entered with or without leading zeros (`007` and `7` both work)
- Files that already exist in the output directory are skipped — safe to re-run after interruption
- Downloads retry up to 3 times on failure
- Supports URLs with any zero-padded numeric suffix: `segment003.ts`, `show_S01E02_001.ts`, `042.ts`, etc.
- Pass `--auto` to skip all prompts (auto-detect file count, use `~/Downloads`, skip confirmation) — used internally by `download-combine-ts`

---

## `combine-ts`

Scans a directory for `.ts` files, automatically groups them by filename prefix, and merges each group into a single `.mp4` using ffmpeg's stream copy (no re-encoding).

### Usage

```
combine-ts [directory]
```

If no directory is given it defaults to `~/Downloads`.

### Examples

```bash
# Scan ~/Downloads (default)
combine-ts

# Scan a specific folder
combine-ts ~/Movies/raw
```

### Interactive flow

The tool scans the directory and displays the detected groups:

```
📂  Detected Groups
─────────────────────────────────────────
  [1] MyShow_S01E02  →  MyShow_S01E02.mp4
      MyShow_S01E02_001.ts
      MyShow_S01E02_002.ts
      MyShow_S01E02_003.ts
  [2] Documentary.2024  →  Documentary.2024.mp4
      Documentary.2024.001.ts
      Documentary.2024.002.ts
  [3] standalone  →  standalone.mp4
      standalone.ts
─────────────────────────────────────────

Which groups to combine?
  a  All groups
  1-3  Specific group(s), comma-separated (e.g. 1,3)
  q  Quit
```

Select groups, confirm the output directory, and the merge runs:

```
⚙️   Combining
─────────────────────────────────────────

▶  MyShow_S01E02  (3 file(s) → MyShow_S01E02.mp4)
   + MyShow_S01E02_001.ts
   + MyShow_S01E02_002.ts
   + MyShow_S01E02_003.ts

✔ Saved: MyShow_S01E02.mp4  (340M)
```

### How grouping works

The tool strips the trailing sequence number from each filename to determine the group prefix:

| Filename | Group prefix | Output |
|---|---|---|
| `show_001.ts`, `show_002.ts` | `show` | `show.mp4` |
| `MyShow_S01E02_001.ts`, `_002.ts` | `MyShow_S01E02` | `MyShow_S01E02.mp4` |
| `Documentary.2024.001.ts`, `.002.ts` | `Documentary.2024` | `Documentary.2024.mp4` |
| `standalone.ts` | `standalone` | `standalone.mp4` |

### Notes

- Uses `ffmpeg -c copy` — pure stream copy, no re-encoding, so merging is very fast
- Output files get `-movflags +faststart` for web-friendly MP4s
- If an output file already exists it is skipped — delete it to re-merge
- Files within each group are sorted alphabetically before merging
- Source `.ts` files are deleted after a successful merge
- Pass `--auto` to skip all prompts (select all groups, use the scanned directory as output) — used internally by `download-combine-ts`

---

## `download-combine-ts`

Downloads a `.ts` sequence and merges it into an `.mp4` in one fully automated step.
All files are saved to `~/Downloads` and the source `.ts` files are removed after a successful merge.

### Usage

```
download-combine-ts [url-of-first-ts-file]
```

### Examples

```bash
# Pass the URL directly
download-combine-ts https://cdn.example.com/show/episode_001.ts

# Run interactively (will prompt for URL)
download-combine-ts
```

### Flow

```
🎬  download-combine-ts
─────────────────────────────────────────
▸ Saving to : /Users/you/Downloads
▸ Mode      : auto-detect files → download → combine → clean up

Step 1 / 2 — Download
─────────────────────────────────────────
▸ Auto-detecting last file...
  Probing episode_001.ts ... ✔  (200)
  Probing episode_002.ts ... ✔  (200)
  Probing episode_003.ts ... ✖  (404) — stopped
✔ Detected last file: 002  (2 file(s))
⬇  [1/2] episode_001.ts ...  ✔  12M
⬇  [2/2] episode_002.ts ...  ✔  11M

Step 2 / 2 — Combine
─────────────────────────────────────────
▶  episode  (2 file(s) → episode.mp4)
✔ Saved: episode.mp4  (23M)
   deleted episode_001.ts
   deleted episode_002.ts

✅  All done
─────────────────────────────────────────
✔ MP4 file(s) saved in /Users/you/Downloads
```

### Notes

- `download-combine-ts` looks for `download-ts` and `combine-ts` next to itself first, then falls back to `PATH`
- All three scripts must be installed for this to work

---

## Typical workflow

**One step** — download and merge automatically:

```bash
download-combine-ts https://cdn.example.com/video/episode_001.ts
```

**Two steps** — with manual control over file count and output directory:

```bash
# 1. Download the segments
download-ts https://cdn.example.com/video/episode_001.ts ~/Movies

# 2. Merge them into an MP4
combine-ts ~/Movies
```

---

## License

MIT
