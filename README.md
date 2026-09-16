# oscar

- online version

Here's a minimal `app.py` that does what you want. It uses **GuessIt** for smart title parsing, **TMDB API** for posters, and **yt-dlp** for trailers.

```python
import os, re, requests, yt_dlp, shutil
from guessit import guessit
from pathlib import Path

TMDB_KEY = "YOUR_TMDB_API_KEY"  # get from themoviedb.org

def clean_title(raw):
    info = guessit(raw)
    # GuessIt handles S01E01, dots, brackets, quality tags etc.
    title = info.get('title', Path(raw).stem)
    year  = info.get('year')
    return title, year

def fetch_tmdb(title, year=None):
    q = title
    if year: q += f"&year={year}"
    r = requests.get(
        f"https://api.themoviedb.org/3/search/movie?api_key={TMDB_KEY}&query={q}"
    ).json()
    if not r.get('results'): return None
    m = r['results'][0]
    mid = m['id']
    # poster
    poster_url = f"https://image.tmdb.org/t/p/w500{m['poster_path']}"
    # trailer via videos endpoint
    vids = requests.get(
        f"https://api.themoviedb.org/3/movie/{mid}/videos?api_key={TMDB_KEY}"
    ).json().get('results', [])
    trailer = next((v['key'] for v in vids if v['type']=='Trailer' and v['site']=='YouTube'), None)
    return m['title'], m.get('vote_average', 0), poster_url, trailer

def download(url, path):
    with requests.get(url, stream=True) as r:
        r.raise_for_status()
        with open(path, 'wb') as f:
            for chunk in r.iter_content(8192): f.write(chunk)

def process(fname):
    title, year = clean_title(fname)
    data = fetch_tmdb(title, year)
    if not data: return print(f"✗ not found: {fname}")
    real_title, rating, poster_url, trailer_key = data
    folder = Path(real_title); folder.mkdir(exist_ok=True)
    # poster
    download(poster_url, folder / f"{real_title} poster-({rating:.1f}).jpg")
    # move video
    shutil.move(fname, folder / fname)
    # trailer
    if trailer_key:
        ydl_opts = {'outtmpl': str(folder / 'trailer.%(ext)s'), 'quiet': True}
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            ydl.download([f"https://youtu.be/{trailer_key}"])
    print(f"✓ {real_title} → {folder}/")

# run on all .mkv/.mp4 in cwd
for f in os.listdir('.'):
    if f.endswith(('.mkv','.mp4','.avi')):
        process(f)
```

Install deps: `pip install guessit requests yt-dlp`

Get a free TMDB API key at themoviedb.org. GuessIt handles `some.thing.S01E01.1080p.mkv` → extracts `some thing` as title automatically.

- offline version
```python
import os
import re
from pathlib import Path

def get_folder_name(filename):
    """
    Extract 'movie-name.S0XE0X' part from filename.
    Handles S01E01, s01e01, S1E1, etc.
    """
    stem = Path(filename).stem
    # match everything up to and including SxxExx
    m = re.match(r'^(.*?\.s\d+e\d+)', stem, re.IGNORECASE)
    if m:
        return m.group(1)
    return None

def organize():
    cwd = Path.cwd()
    for f in cwd.iterdir():
        if not f.is_file():
            continue
        if f.suffix.lower() not in ('.mkv', '.mp4', '.avi', '.mov', '.wmv', '.flv'):
            continue

        folder_name = get_folder_name(f.name)
        if not folder_name:
            print(f"✗ skip (no SxxExx): {f.name}")
            continue

        target_dir = cwd / folder_name
        target_dir.mkdir(exist_ok=True)

        new_path = target_dir / f.name
        if new_path.exists():
            print(f"✗ already exists: {new_path}")
            continue

        f.rename(new_path)
        print(f"✓ {f.name}  →  {folder_name}/")

if __name__ == "__main__":
    organize()
```

**How it works:**

- Scans current directory for video files (`.mkv`, `.mp4`, `.avi`, `.mov`, `.wmv`, `.flv`).
- Uses regex `^(.*?\.s\d+e\d+)` to grab everything **up to and including** the `SxxExx` part, ignoring whatever comes after (like `1080p.BluRay.x264.mkv`).
- Case-insensitive, so `S01E01`, `s01e01`, `S1E1` all work.
- Creates a folder with that exact name and moves the file inside.

**Example:**
```
some.thing.S01E02.1080p.mkv   →   some.thing.S01E02/some.thing.S01E02.1080p.mkv
Breaking.Bad.s05e14.HDTV.mkv  →   Breaking.Bad.s05e14/Breaking.Bad.s05e14.HDTV.mkv
```

Just drop it in the folder and run `python app.py`.
