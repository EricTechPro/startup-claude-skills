---
name: scoop
description: Competitor content gap analysis — extracts ALL comments from competitor and your own YouTube videos, identifies what audiences want but aren't getting, ranks gaps by engagement, and proposes video ideas. Use when the user says "scoop," "content gap," "analyze competitor videos," "what should my next video be about," "find content gaps," or "competitor analysis."
metadata:
  version: 3.0.0
---

# Scoop — Content Gap Analysis

Extracts ALL comments from competitor YouTube videos (and optionally your own), identifies content gaps ranked by audience engagement, and synthesizes a content strategy.

## Why this skill exists

Content-gap analysis done by hand is mostly dull categorization — reading hundreds of comments, clustering them, counting likes. It's the kind of work where the model is *great* at the qualitative part (grouping a comment under the right theme, choosing keywords for a gap) and *unreliable* at the quantitative part (adding up likes, merging counts across videos). An earlier version of this skill let the model do both, and the arithmetic came out wrong in ways the user couldn't audit. This version splits the work: **the model classifies, Python scores.** Every score in the final output is traceable back to the comment IDs that produced it.

## Setup (one-time)

The underlying toolkit eagerly imports `apify_client` from its package `__init__` even though this skill doesn't use Apify. If you haven't installed it:

```bash
pip3 install apify-client
```

Do this once per environment. It's a known toolkit quirk.

Transcription requires `mlx_whisper` (Apple Silicon):

```bash
pipx install mlx-whisper
```

The model (`mlx-community/whisper-medium-mlx`, ~1.5GB) downloads automatically on first run.

## Pipeline Overview

```
┌─────────┐   ┌─────────┐   ┌──────────┐   ┌──────────┐   ┌─────────┐   ┌──────────┐
│ 1.      │──>│ 2.      │──>│ 3.       │──>│ 3.5      │──>│ 4.      │──>│ 5.       │
│ Extract │   │ Classify│   │ Verify   │   │ Channel  │   │ Score   │   │ Narrate  │
│ videos  │   │ gaps    │   │ vs trans-│   │ gap      │   │ gaps    │   │ strategy │
│ (JSON)  │   │ (JSON)  │   │ cript    │   │ check    │   │ (md)    │   │ (prose)  │
└─────────┘   └─────────┘   └──────────┘   └──────────┘   └─────────┘   └──────────┘
  Python        Model         Python         Python         Python        Model
  side          side          side           side           side          side
```

Steps 1, 3, 3.5, 4 are deterministic Python. Steps 2 and 5 are the model's creative work. The separation is the whole point — it means scores are reproducible and the strategy narrative is grounded in facts Python verified.

**All intermediate files live in `/tmp/`:**

- `/tmp/scoop_videos.json` — extracted data for all videos
- `/tmp/scoop_gaps.json` — classified gaps (written by model, cross-check appends `covered_in_transcript`)
- `/tmp/scoop_scored.md` — rendered ranked tables (fed back into the narrative)

**Final narrative output** lives in `output/scoop/<topic>.md` — the user-facing deliverable produced in Step 5. `output/` is gitignored at the repo root, so these files stay out of version control by design.

---

## Step 1 — Extract Videos

### 1a. Collect URLs

Ask the user two things via `AskUserQuestion`:

> "What competitor video URLs do you want to analyze? (paste one or more YouTube URLs)"

Then:

> "Do you have your own video on this same topic? If yes, paste the URL. If not, type N/A."

### 1b. Extract and write a single combined JSON

Run ONE Python block that extracts every URL and writes one combined file. Role-tag each video (`competitor1`, `competitor2`, …, or `own`) so later steps can distinguish them.

```python
import json, sys, subprocess, os
from yt_toolkit.extractors.video import VideoExtractor

URLS = [
    ("competitor1", "https://www.youtube.com/watch?v=..."),
    ("competitor2", "https://youtu.be/..."),
    ("own",         "https://youtu.be/..."),   # omit tuple if user said N/A
]

extractor = VideoExtractor()
videos = []

for role, url in URLS:
    try:
        data = extractor.extract(url, all_comments=True, save_to_file=False)
    except Exception as e:
        print(f"[skip] {role} {url}: {e}", file=sys.stderr)
        continue

    # --- Local transcription via mlx_whisper ---
    vid = data.get("video_id", role)
    audio_path = f"/tmp/scoop_{vid}.mp3"
    transcript = {"available": False, "text": "", "segments": [], "language": ""}

    try:
        # Download audio
        subprocess.run(
            ["yt-dlp", "-x", "--audio-format", "mp3", "--audio-quality", "0",
             "-o", audio_path, "--no-playlist", url],
            check=True, capture_output=True, text=True,
        )
        # Transcribe locally (auto-detect language)
        subprocess.run(
            ["mlx_whisper", audio_path,
             "--model", "mlx-community/whisper-medium-mlx",
             "--output-format", "json", "--output-dir", "/tmp"],
            check=True, capture_output=True, text=True,
        )
        json_path = audio_path.rsplit(".", 1)[0] + ".json"
        if os.path.exists(json_path):
            with open(json_path) as jf:
                w = json.load(jf)
            transcript = {
                "available": True,
                "text": w.get("text", ""),
                "segments": [
                    {"text": s["text"], "start": s["start"],
                     "duration": round(s["end"] - s["start"], 3)}
                    for s in w.get("segments", [])
                ],
                "language": w.get("language", "en"),
            }
            os.remove(json_path)
        else:
            print(f"[warn] {role}: whisper JSON not found at {json_path}", file=sys.stderr)
    except subprocess.CalledProcessError as e:
        print(f"[warn] {role}: transcription failed: {e}", file=sys.stderr)
    except Exception as e:
        print(f"[warn] {role}: transcription error: {e}", file=sys.stderr)
    finally:
        if os.path.exists(audio_path):
            os.remove(audio_path)

    data["transcript"] = transcript
    # --- End local transcription ---

    data["role"] = role
    videos.append(data)

    # Truncation guard — YouTube API pagination can silently stop short.
    got = len(data.get("comments", []))
    expected = data.get("statistics", {}).get("comments", 0)
    if expected and abs(got - expected) / expected > 0.05:
        print(f"[warn] {role}: extracted {got} comments, metadata says {expected} — possible truncation", file=sys.stderr)

    stats = data.get("statistics", {})
    tr = data.get("transcript", {})
    print(f"{role}: {data['title'][:60]}  |  views {stats.get('views','?'):,}  |  "
          f"comments got={got} meta={expected}  |  transcript={'yes' if tr.get('available') else 'no'}")

with open("/tmp/scoop_videos.json", "w") as f:
    json.dump(videos, f)

print(f"Saved {len(videos)} videos to /tmp/scoop_videos.json")
```

**Important:** always pass `all_comments=True`. The default 20 is never enough — the whole value of Scoop is reading the long tail.

If extraction fails for one URL, continue with the rest. Don't abort. If transcription fails for one video, the video is still included (with `transcript.available = False`) — comments are the primary data, transcripts are for cross-checking.

### 1c. Present the extraction summary

After running the block, present a table to the user:

```
| # | Video Title                              | Channel        | Views   | Comments | Transcript | Role        |
|---|------------------------------------------|----------------|---------|----------|------------|-------------|
| 1 | GSD vs Superpowers vs Claude Code        | Chase AI       | 26,453  | 96       | yes        | competitor1 |
| 2 | This One Plugin Just 10x'd Claude Code   | Nate Herk      | 60,029  | 102      | yes        | competitor2 |
| 3 | Claude Code + SUPERPOWERS = ...          | Eric Tech      | 27,409  | 10       | yes        | own         |
```

If a truncation warning fired, surface it here. If any transcript is missing, flag it — the cross-check in Step 3 will skip that video.

---

## Step 2 — Classify Comments into Gaps

This is where the model earns its keep. Read `/tmp/scoop_videos.json` (or the summary you already produced in context) and cluster comments into **content gaps** — themes the audience keeps bringing up that represent unaddressed (or poorly addressed) demand.

### What counts as a gap

1. **Content gap** — a topic the commenter wants covered.
2. **Frustration** — criticism of what the video did/didn't do (needs a different tone of response video, so we tag it).
3. **Follow-up request** — explicit "please make a video on X."
4. **Alternative tool mentioned** — a tool the audience keeps naming that the video ignored.
5. **Methodology criticism** — what the audience thinks was done wrong.

These categories are lenses, not output buckets. Every gap gets represented in `gaps.json` with the same schema below.

### Rules for classification

- **Global, not per-video.** Classify across all videos in a single pass. If the same theme shows up in V1 and V2, it's ONE gap with two `video_appearances` entries — not two separate gaps.
- **Cite comment IDs.** For each gap, list every comment you're using as evidence. This is how scoring stays honest. Split `comment_ids` (top-level) from `reply_comment_ids` (nested replies) — the scoring script weights them differently.
- **Exclude creator self-comments.** The YouTube API returns author display names (e.g. `@Chase-H-AI`) which don't exact-match the channel display title (e.g. `Chase AI`), so naive string matching fails. Instead, identify the creator's handle per video by looking for pinned/promo comments ("Full courses + unlimited support: ...", "Get the Claude Code Masterclass: ..."), and record them in `gaps.json` as `creator_handles_by_video: {video_id: ["@handle"]}`. The scoring script reads this map and drops any cited comment whose author matches. Creators replying to their own comments aren't audience demand.
- **Keywords are search terms, not labels.** Pick 3–6 distinct phrases that a transcript would contain if the video addressed this gap. Include synonyms. Bad: `["quality"]`. Good: `["code quality", "maintainability", "architecture", "test coverage", "production-ready"]`. Step 3 uses these for a literal transcript search — vague keywords produce vague results.
- **Frustration flag.** Set `frustration: true` when the gap is driven by criticism of the existing video ("you missed X," "this is wrong"). Set `false` when it's demand or curiosity ("please make a video on X," "how do you do Y?"). One boolean, nothing more nuanced — boundary cases go to `false`.
- **One comment can cite multiple gaps.** That's fine. The scoring script will log it so you can spot over-categorization.

### Write the JSON

Write `/tmp/scoop_gaps.json` with the following schema:

```json
{
  "creator_handles_by_video": {
    "celLbDMGy8w": ["@Chase-H-AI"],
    "4XqVR6xI6Kw": ["@nateherk"]
  },
  "gaps": [
    {
      "gap_id": "g1",
      "name": "Complex brownfield project comparison",
      "description": "Audience wants testing on existing production codebases, not trivial greenfields.",
      "frustration": true,
      "keywords": ["brownfield", "complex project", "existing codebase", "production", "legacy"],
      "video_appearances": [
        {
          "video_id": "celLbDMGy8w",
          "comment_ids": ["UgxAbc123...", "UgyDef456..."],
          "reply_comment_ids": ["UgzGhi789..."]
        }
      ]
    }
  ]
}
```

Aim for 6–12 gaps. Fewer than 6 usually means you're under-categorizing; more than 12 means you're splitting themes that belong together.

---

## Step 3 — Transcript Cross-Check

This step catches the single biggest accuracy failure in gap analysis: ranking a "gap" as top missing content when the competitor actually covered it. A comment complaining "you never talked about code quality" is only useful evidence of a gap if the transcript confirms it. If the video dedicated 90 seconds to code quality and commenters still asked for more, that's a different kind of opportunity — "covered but shallow" — and it needs different framing in the final narrative.

This pass is a keyword search, not a depth judgment. It produces a boolean per gap per video.

Run this inline:

```python
import json, re

with open("/tmp/scoop_videos.json") as f:
    videos = json.load(f)
with open("/tmp/scoop_gaps.json") as f:
    data = json.load(f)

by_id = {v["video_id"]: v for v in videos}

for gap in data["gaps"]:
    coverage = []
    for app in gap["video_appearances"]:
        vid = app["video_id"]
        v = by_id.get(vid)
        if not v:
            continue
        tr = v.get("transcript") or {}
        if not tr.get("available"):
            coverage.append({"video_id": vid, "status": "NO_TRANSCRIPT", "hits": []})
            continue
        text = (tr.get("text") or "").lower()
        segments = tr.get("segments") or []
        hits = []
        for kw in gap["keywords"]:
            needle = kw.lower().strip()
            if not needle:
                continue
            if needle in text:
                # find approximate timestamp of first hit
                for seg in segments:
                    if needle in (seg.get("text") or "").lower():
                        hits.append({"keyword": kw, "start": seg.get("start", 0)})
                        break
                else:
                    hits.append({"keyword": kw, "start": None})
        status = "COVERED" if hits else "NOT_COVERED"
        coverage.append({"video_id": vid, "status": status, "hits": hits})
    gap["coverage"] = coverage

with open("/tmp/scoop_gaps.json", "w") as f:
    json.dump(data, f, indent=2)

# Human-readable summary
for gap in data["gaps"]:
    statuses = [c["status"] for c in gap["coverage"]]
    print(f"{gap['gap_id']} {gap['name'][:50]:50s}  {statuses}")
```

**How to read the output:**

- `NOT_COVERED` in every video → genuine unaddressed gap. Top candidate for a new video.
- `COVERED` in at least one video → the topic was touched on. Audience is asking for *more depth*, not introduction. Frame the recommendation accordingly ("deeper than X's 90-second mention").
- `NO_TRANSCRIPT` → can't verify. Flag in the final output rather than silently treating it as NOT_COVERED.

Do NOT drop COVERED gaps from the output. Commenters asking for more depth on a covered topic is still real demand; it just needs different framing.

---

## Step 3.5 — Channel Gap Check

Step 3 checks whether the *analyzed video* covered a gap. But the competitor might have a whole separate video about it on their channel. Without this check, Scoop cheerfully recommends "make a video about X" when the competitor already has one — just not the one you analyzed.

This step fetches the full video catalog for each competitor channel and keyword-searches titles and descriptions against the gap keywords.

Run this inline:

```python
import json
from yt_toolkit.utils.youtube_api import YouTubeAPI

with open("/tmp/scoop_videos.json") as f:
    videos = json.load(f)
with open("/tmp/scoop_gaps.json") as f:
    data = json.load(f)

api = YouTubeAPI()

# Collect unique competitor channels (skip "own")
channels = {}
for v in videos:
    if v.get("role", "").startswith("competitor") and v.get("channel_id"):
        cid = v["channel_id"]
        if cid not in channels:
            channels[cid] = {
                "channel_title": v.get("channel_title", ""),
                "videos": api.get_channel_videos(cid, max_results=None),
            }
            print(f"Fetched {len(channels[cid]['videos'])} videos from {channels[cid]['channel_title']}")

# For each gap, check if any channel video title/description matches the keywords
for gap in data["gaps"]:
    channel_coverage = []
    for cid, ch in channels.items():
        matching = []
        for cv in ch["videos"]:
            title = (cv.get("title") or "").lower()
            desc = (cv.get("description") or "").lower()
            searchable = title + " " + desc
            for kw in gap["keywords"]:
                if kw.lower().strip() in searchable:
                    matching.append({"video_id": cv["video_id"], "title": cv.get("title", "")})
                    break  # one keyword match is enough per video
        status = "COVERED_ON_CHANNEL" if matching else "NOT_COVERED_ON_CHANNEL"
        channel_coverage.append({
            "channel_id": cid,
            "channel_title": ch["channel_title"],
            "status": status,
            "matching_videos": matching[:5],  # cap at 5 most relevant
        })
    gap["channel_coverage"] = channel_coverage

with open("/tmp/scoop_gaps.json", "w") as f:
    json.dump(data, f, indent=2)

# Summary
for gap in data["gaps"]:
    statuses = [f"{cc['channel_title']}:{cc['status']}" for cc in gap["channel_coverage"]]
    matches = sum(len(cc["matching_videos"]) for cc in gap["channel_coverage"])
    print(f"{gap['gap_id']} {gap['name'][:50]:50s}  {statuses}  ({matches} matching videos)")
```

**How to read the output:**

- `NOT_COVERED_ON_CHANNEL` everywhere → genuine gap across the competitor's entire catalog. Strongest opportunity.
- `COVERED_ON_CHANNEL` → the competitor has a dedicated video about this topic. Your angle needs to be differentiation (better depth, different perspective, newer info), not "they never covered this."
- The `matching_videos` list shows which videos match, so the narrative in Step 5 can reference them specifically.

Do NOT drop COVERED_ON_CHANNEL gaps. They're still valid opportunities — the framing just changes from "nobody covers this" to "here's how to do it better."

---

## Step 4 — Score Gaps

Scoring is mechanical so it's consistent. **Do not type a Score column into any markdown table by hand.** Run this block and paste its output.

```python
import json

with open("/tmp/scoop_videos.json") as f:
    videos = json.load(f)
with open("/tmp/scoop_gaps.json") as f:
    data = json.load(f)

# Build lookup: comment_id -> (like_count, author, video_id, is_reply)
# Also capture video metadata for creator-self-comment filter.
comment_by_id = {}
video_meta = {}
for v in videos:
    vid = v["video_id"]
    video_meta[vid] = {
        "title": v.get("title", ""),
        "channel_title": v.get("channel_title", ""),
        "role": v.get("role", ""),
    }
    for c in v.get("comments", []):
        comment_by_id[c["comment_id"]] = {
            "likes": c.get("like_count", 0),
            "author": c.get("author", ""),
            "video_id": vid,
            "is_reply": False,
        }
        for r in c.get("replies", []):
            comment_by_id[r["comment_id"]] = {
                "likes": r.get("like_count", 0),
                "author": r.get("author", ""),
                "video_id": vid,
                "is_reply": True,
            }

creator_map = {vid: set(h.lower() for h in handles)
               for vid, handles in data.get("creator_handles_by_video", {}).items()}

def is_creator(author, vid):
    return author.strip().lower() in creator_map.get(vid, set())

def score_appearance(app):
    """Return dict of per-video score components, filtering creator comments."""
    vid = app["video_id"]
    top_ids = [cid for cid in app.get("comment_ids", [])
               if cid in comment_by_id and not is_creator(comment_by_id[cid]["author"], vid)]
    reply_ids = [cid for cid in app.get("reply_comment_ids", [])
                 if cid in comment_by_id and not is_creator(comment_by_id[cid]["author"], vid)]
    likes = sum(comment_by_id[cid]["likes"] for cid in top_ids + reply_ids)
    mentions = len(top_ids)           # distinct top-level commenters raising this gap
    reply_depth = len(reply_ids)       # replies = discussion depth
    score = likes + mentions * 2 + reply_depth * 3
    return {"likes": likes, "mentions": mentions, "replies": reply_depth, "score": score,
            "top_ids": top_ids, "reply_ids": reply_ids}

# Score each gap per-video, then total across videos
rows = []
citation_counts = {}
for gap in data["gaps"]:
    per_video = []
    total = {"likes": 0, "mentions": 0, "replies": 0, "score": 0}
    for app in gap["video_appearances"]:
        s = score_appearance(app)
        per_video.append({"video_id": app["video_id"], **s})
        for k in ("likes", "mentions", "replies", "score"):
            total[k] += s[k]
        for cid in s["top_ids"] + s["reply_ids"]:
            citation_counts[cid] = citation_counts.get(cid, 0) + 1
    gap["scoring"] = {"per_video": per_video, "total": total}
    rows.append(gap)

# Sort descending by total score
rows.sort(key=lambda g: g["scoring"]["total"]["score"], reverse=True)

# Render markdown
lines = []
lines.append("## Combined Gap Ranking\n")
lines.append("| # | Gap | Score | Likes | Mentions | Replies | Appears In | Coverage | Channel | Frustration |")
lines.append("|---|-----|-------|-------|----------|---------|------------|----------|---------|-------------|")
for i, g in enumerate(rows, 1):
    t = g["scoring"]["total"]
    appears = ",".join(video_meta[a["video_id"]]["role"] for a in g["video_appearances"])
    cov = ",".join(f"{video_meta[c['video_id']]['role']}:{c['status']}" for c in g.get("coverage", []))
    ch_cov = ",".join(f"{cc['channel_title']}:{cc['status'].replace('_ON_CHANNEL','')}" for cc in g.get("channel_coverage", []))
    fr = "yes" if g.get("frustration") else "no"
    lines.append(f"| {i} | {g['name']} | {t['score']} | {t['likes']} | {t['mentions']} | {t['replies']} | {appears} | {cov} | {ch_cov} | {fr} |")

lines.append("\n## Per-Video Breakdown\n")
for g in rows:
    lines.append(f"\n### {g['name']}  (total {g['scoring']['total']['score']})")
    lines.append("| Video | Score | Likes | Mentions | Replies |")
    lines.append("|-------|-------|-------|----------|---------|")
    for pv in g["scoring"]["per_video"]:
        role = video_meta[pv["video_id"]]["role"]
        lines.append(f"| {role} | {pv['score']} | {pv['likes']} | {pv['mentions']} | {pv['replies']} |")

# Diagnostics
multi_cited = sum(1 for n in citation_counts.values() if n > 1)
total_comments = sum(len(v.get("comments", [])) + sum(len(c.get("replies", [])) for c in v.get("comments", []))
                     for v in videos)
cited_any = len(citation_counts)
lines.append(f"\n---\n_Diagnostics: {cited_any}/{total_comments} comments cited ({100*cited_any/max(total_comments,1):.0f}%); "
             f"{multi_cited} cited in multiple gaps._")

out = "\n".join(lines)
with open("/tmp/scoop_scored.md", "w") as f:
    f.write(out)
print(out)
```

**Scoring formula (fixed, do not mutate):**

```
score = likes + mentions * 2 + replies * 3
where:
  likes    = sum of like_count across all cited comments (top + replies)
  mentions = count of distinct top-level comments cited
  replies  = count of cited reply comments
```

Rationale for the weights:
- Likes are the rawest engagement signal — pass through as-is.
- Each distinct mention represents a separate person raising the issue; the ×2 is a baseline value even if their comment has zero likes.
- Replies indicate active debate in threads — audiences argue about things they care about. The ×3 rewards discussion depth.

Combined across videos = sum of per-video scores. No re-invention. The script enforces this; the model must not override.

**Diagnostic line at the bottom** — if citation coverage is under 25% of total comments, you skimmed; if it's over 90%, you over-categorized. Expected range is 30–70%.

---

## Step 5 — Strategy Narrative

Now the qualitative work resumes. Use the tables from Step 4 as ground truth — do NOT recompute scores or adjust rankings in prose.

### 5a. Extraction summary

Show the table from Step 1c, including any truncation warnings.

### 5b. Top-level ranking

Paste the markdown from `/tmp/scoop_scored.md` verbatim (or at least the Combined Gap Ranking table and the top 5 per-video breakdowns).

### 5c. Video ideas

For each of the top 5 gaps, write:

```
### Video Idea #N: [Working Title]

**Gap it addresses:** [Name] — score X across N videos ([role list])
**Coverage status:** [NOT_COVERED everywhere | COVERED in roleX | NO_TRANSCRIPT for roleY]
**Channel coverage:** [NOT_COVERED_ON_CHANNEL everywhere = true gap | COVERED_ON_CHANNEL in channelX = they have a dedicated video, list title(s)]
**Why this gap exists:** [1–2 sentences, grounded in what the competitor DID or DIDN'T say.
                          If COVERED_ON_CHANNEL, acknowledge the existing video and explain why there's still an opportunity (outdated, shallow, different angle).]
**Audience demand evidence:** [likes, mentions, replies — paste from the scoring table]
**Competitive angle:** [What the user can do that competitors didn't. If COVERED_ON_CHANNEL, what depth/freshness they can add beyond the existing video.]
**Key differentiator:** [The one thing making this video stand out]
**Framing note:** [if frustration=true, rebuttal framing; else demand framing]
```

**Title guidelines:**
- Under 60 characters.
- Frustration gaps → rebuttal/audit tone ("I Audited...", "The Truth About...").
- Non-frustration → curiosity-gap or promise framing ("How to...", "The Missing...").
- Include the primary topic keyword naturally.

### 5d. Recommendation

Close with a clear pick:

```
## Recommendation

**Next video:** [Title of #1 recommendation]
**Why this one first:** [One paragraph. Must acknowledge:
  (a) audience demand (score + mentions),
  (b) coverage status (addressed this gap being genuinely missing vs. competitor did touch it),
  (c) channel coverage (whether the competitor has a dedicated video about it already),
  (d) the user's positioning advantage.]
```

If the #1 gap is marked COVERED everywhere, be explicit in the recommendation: "The competitor already covered this, so the angle is depth, not introduction." If COVERED_ON_CHANNEL, name the competitor's existing video and explain why there's still an opportunity.

### 5e. Save the narrative

Write the full Step 5 narrative (5a–5d, plus 5f if applicable) to a file in `output/scoop/`. Create the folder with `mkdir -p output/scoop` if it doesn't exist yet.

Filename convention:
- Short descriptor of the analysis topic, kebab-case, prefixed with `scoop-`.
- Examples: `scoop-bookkeeping-video-plan.md`, `scoop-brownfield-experiment-plan.md`.
- Avoid dates in the filename unless the user asks — the analysis topic is the useful identifier.

`output/` is already in `.gitignore`, so these files stay out of version control by design. They're deliverables for the user, not artifacts for the repo.

### 5f. If the user's own video was analyzed

Add two short sections based on comments on their own video:

**What the audience validated (positive signals):**
- Which aspects got praised (supportive comments with meaningful likes)
- What concrete promises the user made in replies that are worth delivering on

**What the audience criticized:**
- Direct pushback
- Skepticism about specific claims

Keep these tight — they're context for the user, not the primary deliverable.

---

## Reference: Python API

```python
from yt_toolkit.extractors.video import VideoExtractor
from yt_toolkit.utils.youtube_api import YouTubeAPI, extract_video_id

# Primary: full extraction with ALL comments
extractor = VideoExtractor()
data = extractor.extract(url, all_comments=True, save_to_file=False)
# Returns dict with keys: video_id, url, title, description, channel_id, channel_title,
#   published_at, tags, duration, thumbnail,
#   statistics: {views, likes, comments},
#   transcript: {available, text, segments[{text, start, duration}], language},
#     (transcript is overwritten by local mlx_whisper transcription in Step 1b)
#   comments: [{comment_id, video_id, author, text, like_count, published_at, replies: [...]}]

# Quick metadata only (no comments/transcript)
api = YouTubeAPI()
info = api.get_video_info(extract_video_id(url))

# Channel video catalog (used in Step 3.5)
channel_videos = api.get_channel_videos(channel_id, max_results=None)
# Returns list of dicts: [{video_id, title, description, published_at, thumbnail}, ...]
```

**Key correction from v1.0:** views live at `data["statistics"]["views"]`, not `data["views"]`. Same for likes and comments counts.

## Why the pipeline is structured this way

The earlier version of this skill asked the model to do everything in its head — read comments, group them, compute scores, write tables — and emit the result as free-form markdown. The failure mode was quiet arithmetic errors: a gap with 59 likes and 7 mentions would be scored as 26 instead of 73, with no way to audit what went wrong. The current split — model classifies into JSON with comment-ID citations, Python sums mechanically — exists specifically because that failure is unfixable by "just being more careful." It requires moving state out of the model's head.

The transcript cross-check exists for a similar reason: a complaint like "you never addressed code quality" is evidence of a gap only if the transcript confirms it. Without the check, the skill cheerfully ranks covered topics as top missing gaps.

Neither step is novel work — they're just work the v1.0 skill silently skipped.
