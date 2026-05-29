---
name: vtake-cut
description: Turn a local video into metadata, transcript, and an AI-composed card-based video where the agent freely designs and writes HTML cards in conversation. Use when the user asks for VTake, vtake-cut, video takeaways, transcript cleanup, or AI-composed video repurposing.
---

# VTake Local Workflow

VTake converts a local input video into a card-based composition. The agent
designs the cards (timing + content) and **writes each card's HTML directly
in the conversation**, then assembles a single composition HTML and renders
it to MP4 via `hyperframes`. There is no fixed archetype list and no
prescribed card structure — the cards emerge from what the transcript
actually says.

Inspectable intermediate files in the work directory:

- `metadata.json` — duration / width / height / fps
- `audio.mp3` — extracted audio
- `transcript.json` — segments + words with timestamps
- `storyboard.json` — lightweight card outline (the agent's plan)
- `public/cards/card-XX.html` — one HTML fragment per card
- `public/index.html` — final assembled composition
- `output.mp4` — rendered video

## CLI Resolution

```bash
# vtake CLI — auto-downloaded from npm on first run
npx -y vtake@latest --help

# hyperframes — for rendering the assembled HTML to MP4
npx hyperframes render --help
```

> Every `vtake …` command below is shorthand for `npx -y vtake@latest …`.

## Workflow

### 1. Check Environment

```bash
npx -y vtake@latest doctor
# confirm bundled assets:
ls "<SKILL_DIR>/assets/fonts" "<SKILL_DIR>/assets/vendor/gsap.min.js"
```

Required:

- `ffmpeg` / `ffprobe` (system)
- `<SKILL_DIR>/assets/fonts/*.woff2`, `<SKILL_DIR>/assets/vendor/gsap.min.js` (bundled inside this skill, staged to work dir in Step 9)

Optional:

- `ELEVEN_API_KEY` — when set, `vtake transcribe` connects to ElevenLabs
  directly and bypasses the rate-limited proxy. When **not** set, it falls
  back to `https://vtake.app/api/transcribe`, which enforces **3 requests
  per minute per IP**. Override the proxy URL with
  `VTAKE_TRANSCRIBE_ENDPOINT` (e.g. for local Wrangler dev).

Strongly recommended on macOS for `hyperframes render`:

```bash
export PRODUCER_BROWSER_GPU_MODE=hardware
```

### 2. Create a Work Directory

```bash
VIDEO_PATH="/absolute/path/input.mp4"
WORK_DIR=".vtake-work/$(basename "$VIDEO_PATH" | sed 's/\.[^.]*$//')"
mkdir -p "$WORK_DIR"
```

### 3. Extract Audio and Metadata

```bash
npx -y vtake@latest extract "$VIDEO_PATH" --out-dir "$WORK_DIR"
```

Outputs: `metadata.json` (duration, width, height, fps) + `audio.mp3`.

### 4. Transcribe

```bash
npx -y vtake@latest transcribe "$WORK_DIR/audio.mp3" --out-dir "$WORK_DIR" --asr elevenlabs
```

Output: `transcript.json` with `{ segments, words, raw }`.

**Rate limiting (proxy mode only — no `ELEVEN_API_KEY`):** the server allows
3 requests per minute per IP. If you see an error starting with
`rate_limited:` or `service_busy:`, do **not** auto-retry — stop and tell
the user how many seconds to wait (the message includes the retry hint),
then resume from this step when they ask again.

### 5. Correct Transcript

Read `transcript.json` and fix obvious ASR errors:

- Homophones, product names, technical terms, punctuation
- Preserve all `start` / `end` timestamps
- Prefer editing `segments[].text` only
- Edit individual `words[].word` only for clear one-to-one replacements

### 6. Draft a Lightweight Storyboard (in chat)

**No CLI involved.** Read `transcript.json` + `metadata.json` and design
cards directly. `storyboard.json` is an agent-internal planning artifact
— no vtake CLI command consumes it; it exists so you can think clearly
about timing and content before writing each card's HTML. Keep the
shape consistent with the example below so the same outline can drive
the composition you author in Step 9:

```json
{
  "schemaVersion": 3,
  "composition": {
    "fps": 30,
    "width": 1080,
    "height": 1920,
    "durationSeconds": 121.2,
    "layout": "portrait",
    "themeId": "noir",
    "seed": 42
  },
  "videoTrack": {
    "sourcePath": "input-video.mp4",
    "startSec": 0,
    "endSec": 121.2,
    "bounds": { "x": 0, "y": 0, "width": 1080, "height": 1920 }
  },
  "subtitles": { "enabled": false },
  "cards": [
    {
      "id": "card-01",
      "intent": "Hook with the speaker's anxious midnight question",
      "startSec": 0.5,
      "endSec": 13.0,
      "accentIndex": 0,
      "zone": "fullscreen",
      "contentHints": {
        "kicker": "AN HONEST QUESTION",
        "title": "晚上 11 点的灵魂提问",
        "detail": "客户六十秒语音：「人民币会升值，我的美金保单是不是亏惨了？」"
      }
    }
  ]
}
```

**Required Card fields:**

| field | type | purpose |
|---|---|---|
| `id` | string | stable id used in card HTML & GSAP selectors |
| `intent` | string | natural-language description; fed to card synthesis |
| `startSec` / `endSec` | number | times in seconds (endSec > startSec) |
| `accentIndex` | 0 \| 1 \| 2 \| 3 \| 4 | which of the 5 theme accent colors this card pulls |
| `zone` | enum (see below) | where on the canvas the card lives |
| `contentHints` | object | free-form bag; agent puts kicker/title/detail/data/quote here |
| `archetype` (optional) | string | free-form label you may attach to remember a card's pattern; absent = free-form, which is the default |
| `transition` (optional) | enum: `cut` \| `fade` \| `slide` \| `wipe` | declarative card-to-card transition |

**Five `zone` values:**

| zone | resolved bounds | when to use |
|---|---|---|
| `fullscreen` | covers whole canvas | hero moments, big numbers, mantras |
| `whiteboard-area` | inset 40px margin (or 45% of portrait height) | dense data / annotated content |
| `lower-third` | bottom 30% band | annotation over visible video |
| `side-panel` | right 42% (landscape) or bottom 40% (portrait) | data side, video other side |
| `video-overlay` | full canvas, expects mostly-transparent card | annotation overlays on full-bleed video |

When you assemble the composition in Step 9, resolve each card's `zone`
into pixel bounds on the card-host wrapper following the table above.
Video bounds are set **once** at composition level (`videoTrack.bounds`);
to make video appear to "move between cards", author GSAP tweens against
`#video-wrap` in the composition's `<script>` (see Step 9).

**No prescribed card roles, no prescribed narrative arc.** Cards emerge
from what the video actually says — could be all quotes or all data,
could open with a number or with a story. Let the transcript drive the
rhythm.

**How many takeaways? — auto-infer from duration + density.** No fixed
upper limit. Pick a **base pace** from the video duration, then adjust
by **information density**. Only **floor is fixed: minimum 5 cards** so
even short videos have rhythm.

**Step 1 — base pace by duration** (the natural sec/card for medium density):

| video duration | base pace (sec per card) | rationale |
|---|---|---|
| < 60s (short reel) | **6–8s** | viewers expect fast cuts in short-form |
| 60s – 3 min | **8–12s** | normal social pace |
| 3 – 10 min | **12–20s** | give breathing room; each card carries more |
| 10 – 30 min | **20–35s** | long-form lecture / interview rhythm |
| > 30 min | **30–60s** | episodic, near-chapter feel |

**Step 2 — density multiplier** (multiplies the base pace):

| signal in the transcript | multiplier | effect |
|---|---|---|
| **High density** — many numbers, distinct claims, staccato pacing, list-like enumeration, every 1–2 sentences is a new idea | **× 0.7** | cuts faster, more cards |
| **Medium density** — mixed flow with both data and narrative | **× 1.0** | base pace |
| **Low density** — one extended story, repeated reframing, slow reflective pacing, single argument unfolding | **× 1.5** | cuts slower, fewer cards |

**Step 3 — compute:**

```
secPerCard = basePace × densityMultiplier
cardCount  = max(5, round(videoDurationSec / secPerCard))
```

Examples (notice — **no upper clamp**; long videos naturally produce more cards):

- **30s reel, single punchline (low density)** → 7 × 1.5 = 10.5s/card → round(30/10.5)=3 → floor to **5** cards
- **60s reflective monologue (low density)** → 10 × 1.5 = 15s/card → **4** → floor to **5** cards
- **121s talking-head with rich data (high density)** → 10 × 0.7 = 7s/card → **17** cards
- **5 min interview, mixed density** → 16 × 1.0 = 16s/card → **19** cards
- **10 min deep-dive, high density** → 16 × 0.7 = 11s/card → **55** cards
- **30 min lecture, medium density** → 28 × 1.0 = 28s/card → **64** cards
- **1 hr podcast, low density** → 45 × 1.5 = 67.5s/card → **53** cards

When a card holds longer than ~15s, plan for a richer card (data block,
multi-step reveal, several sub-points unfolding with staggered
animations) — a static one-liner gets boring past 8s. For long pieces
where many cards exceed 30s, consider **chunking the timeline into
sub-compositions** (one .html per chapter, mounted with
`data-composition-src`) so the GSAP timeline per file stays manageable
— see the `timeline_track_too_dense` HyperFrames lint warning.

`content` can be a plain string ("标题：年化 5.69%\n说明：...") or any JSON
shape that captures the data. The agent decides the shape per card.

**Final card — always append card-cta-vtake.**

After all content cards, add one fixed brand outro card to the `cards` array:

```json
{
  "id": "card-cta-vtake",
  "intent": "Powered By vTake brand outro",
  "startSec": <last content card's endSec>,
  "endSec":   <last content card's endSec + 2.0>,
  "accentIndex": 3,
  "zone": "fullscreen",
  "contentHints": {}
}
```

Also extend `composition.durationSeconds` by 2.0 seconds: set it to the same
value as `card-cta-vtake.endSec`. The source video's track stops at its
natural end; the 2-second tail is a pure-graphic brand moment with no video
visible behind the CTA card. The CTA card uses a 1-second cinematic entrance
sequence ("Editorial Cinema") followed by ~0.7s hold + ~0.3s fade-out.

### 7. Decide Render Strategy

#### Confirm Visual Direction with User (DO THIS FIRST)

Before you start designing cards or deciding bounds, **ask the user to
pick the output ratio, the layout, the style, and the card-density
preset**. Frames are auto-selected from the chosen layout × style
combination (see "Auto-pick frame" table below). Before sending the
question, **precompute two things**:

1. **`recommendedRatio`** from the source video's aspect ratio
   (`metadata.json` width / height):
   - `sourceAspect = width / height`
   - `sourceAspect ≥ 1.5` (≥ ~3:2 wide) → recommend **`16:9`**
   - `sourceAspect ≤ 0.7` (≤ ~9:13 tall) → recommend **`9:16`**
   - `0.7 < sourceAspect < 1.5` (near-square) → recommend **`4:5`**

   Mark the recommended option's label with " (推荐 · 匹配源视频 X:Y)"
   so the user sees why it's recommended.

2. **`autoCount`** from Step 6 (`max(5, round(videoSec / (basePace ×
   densityMultiplier)))`) so the "自动" option's label can show the
   concrete number.

**Environment compatibility — pick the best available question channel.**
Not every runtime exposes the same structured-question tool. Apply this
order:

1. **`AskUserQuestion`** (Claude Code, Anthropic Console) — use the
   structured 4-question call below.
2. **Other native clarification tool** (e.g. `ask_question`,
   `request_user_input`, IDE-specific prompt) — use that tool with the
   same 4 question texts and option lists. Preserve the recommendation
   markers and the precomputed values.
3. **No native tool** (Codex CLI, plain text-only runtimes) — **ask
   directly in normal conversation**. Use the plain-text template at the
   end of this section. Keep it to **one message, 4 numbered questions**
   (the global cap is 2–5 questions per round; we stay inside it).

Rules that apply to every channel:

- Ask **at most 2–5 questions per round**. Our 4 here fits.
- Even if missing info doesn't block rendering, **ask once to confirm
  the parameters that materially affect the final output** (ratio,
  layout, style, cardCount).
- If the user has already pre-approved defaults ("just use defaults",
  "无需询问", "auto-pick everything") or asked you not to ask — **skip
  the question entirely** and use: `recommendedRatio`, `layout="stack"`
  (safest cross-ratio default), `style` chosen from transcript tone in
  the most neutral group (editorial/数据), `autoCount`. Tell the user
  what you picked in one sentence and continue.

**Channel A — native `AskUserQuestion`:**

```
// Precompute before the call:
//   recommendedRatio = "16:9" | "9:16" | "4:5"
//   autoCount        = integer (from Step 6)

AskUserQuestion({
  questions: [
    {
      question: "输出视频比例 (画幅)：",
      header: "画幅",
      multiSelect: false,
      // Reorder so the recommended option appears FIRST (per AskUserQuestion convention).
      // Append " (推荐 · 匹配源视频 W×H)" to the recommended option's label.
      options: [
        { label: "16:9 (1920×1080) 横屏", description: "TV / YouTube / 电脑播放。源视频已经是横屏时最自然，画幅最宽。" },
        { label: "9:16 (1080×1920) 竖屏", description: "抖音 / 小红书 / TikTok / Reels。源视频竖屏时最自然；移动端原生体验。" },
        { label: "4:5 (1080×1350) 方屏偏竖", description: "Instagram feed / 微信朋友圈。近方形源视频或想兼顾两种平台时最稳。" }
      ]
    },
    {
      question: "选择整体布局：视频和卡片在画面里如何共存？",
      header: "布局",
      multiSelect: false,
      options: [
        { label: "左右分屏 (split)",     description: "video 和 card 各占画面一半。访谈 / 数据并列时最稳，画面分隔清晰。" },
        { label: "上下分屏 (stack)",     description: "video 在上方 (~52%)，card 在下方。说话人头像 + 总结句的经典组合，竖屏也好用。" },
        { label: "画中画 (pip)",         description: "card 满屏，video 缩成圆角小窗在右上角。内容为主、speaker 为辅时用。" },
        { label: "全屏浮层 (overlay)",   description: "video 全屏播放，card 作为玻璃浮层落在画面上。情绪 / 电影感强烈。" }
      ]
    },
    {
      question: "选择卡片视觉风格 (style)：",
      header: "风格大类",
      multiSelect: false,
      // NOTE: these 3 groups intentionally match the frame auto-pick matrix
      // rows below, so picking a group resolves both `style` group AND the
      // frame matrix column in one step. Memberships are mutually exclusive.
      options: [
        { label: "温暖纸感 (warm-paper)", description: "academic 学术笔记 · editorial 大字编辑 · whiteboard 手写白板 · xhs 小红书。适合访谈反思、产品发布、生活方式、情绪故事。" },
        { label: "冷峻临床 (clinical)",   description: "audit 审计杂志 · swiss 瑞士网格 · terminal CLI · minimal 现代极简。适合财报分析、调查报告、技术教程、严肃陈述。" },
        { label: "实验前卫 (experimental)", description: "geom 撞色几何 · spotlight 暗色聚光。适合短视频高光、产品发布、强烈情绪、电影质感。" }
      ]
    },
    {
      question: "卡片数量 (takeaway 节奏)：要切多少张？",
      header: "卡片数量",
      multiSelect: false,
      options: [
        { label: "自动 (推荐) · 约 N 张", description: "按视频时长和信息密度自动推断 (见 Step 6 规则)。本次推断约 N 张。带 N 进 label —— N 是你刚算出的 autoCount。" },
        { label: "少量 · 约 round(N × 0.6) 张", description: "切得稀疏一点，每张卡停留更久，适合 reflective / 慢节奏。" },
        { label: "更多 · 约 round(N × 1.5) 张", description: "切得更紧凑，节奏更快，适合 staccato / 数据密集 / 短视频高光。" }
      ]
    }
  ]
})
```

**关于"Other"** — `AskUserQuestion` 会自动给"卡片数量"题加 "Other" 选项，
用户可以直接输入数字（如 "8"、"20"）作为 cardCount 目标值。把输入解析为整数：
若解析成功 → 直接用该值（最少 5 张兜底）；解析失败 → 退回 "自动"。

**Channel B — plain-text fallback** (Codex CLI, runtimes without a
native question tool). Post this as one normal message, then wait for
the reply. Bullet-style 1/2/3/4 keeps the reply parseable:

```
我需要先和你确认四个视觉决策再开始切卡片：

1) 输出比例 (画幅)：
   A. 16:9 横屏 (1920×1080) — TV / YouTube / 电脑播放
   B. 9:16 竖屏 (1080×1920) — 抖音 / 小红书 / TikTok
   C. 4:5 方屏偏竖 (1080×1350) — Instagram feed / 兼顾两端
   ▸ 我的推荐:  <recommendedRatio>  (匹配源视频 W×H = <sourceW>×<sourceH>)

2) 整体布局 (video & card 怎么共存)：
   A. split   左右分屏 (50/50)
   B. stack   上下分屏 (video 顶, card 底)
   C. pip     画中画 (card 满屏, video 圆角小窗)
   D. overlay 全屏浮层 (video 全屏, card 玻璃浮层)

3) 卡片风格大类 (与 frame 自动矩阵同构,3 选 1)：
   A. 温暖纸感 warm-paper      (academic / editorial / whiteboard / xhs)
   B. 冷峻临床 clinical        (audit / swiss / terminal / minimal)
   C. 实验前卫 experimental    (geom / spotlight)

4) 卡片数量 (takeaway 节奏)：
   A. 自动 (推荐) — 约 <autoCount> 张
   B. 少量 — 约 round(<autoCount> × 0.6) 张
   C. 更多 — 约 round(<autoCount> × 1.5) 张
   D. 直接给我一个数字 (如 "8"、"20")

回复格式: "1A 2C 3B 4A" 或自然语言均可。
若你想全部用推荐默认值，回复 "默认" / "auto" / "都用推荐" 即可。
```

Parsing the plain-text reply:
- Accept loose formats: `"1A 2C 3B 4A"`, `"A C B A"`, `"16:9 / pip /
  数据 / 自动"`, full sentences, or `默认`.
- If any answer is ambiguous → re-ask only the ambiguous ones (still
  inside the 2–5 cap).
- If the user says "默认 / auto / 都用推荐" → skip without re-asking.

After the user answers (any channel):

1. **Resolve the output canvas** from the ratio answer — these are the
   exact `storyboard.composition.width / height` values to write:

   | user choice | composition.width × height | storyboard.layout field |
   |---|---|---|
   | `16:9` | **1920 × 1080** | `"landscape"` |
   | `9:16` | **1080 × 1920** | `"portrait"` |
   | `4:5`  | **1080 × 1350** | `"portrait"` (schema treats 4:5 as portrait — height > width) |

   For **4:5 bounds inside `references/layouts/*.html`** — those files
   only document landscape (1920×1080) and portrait (1080×1920). For
   4:5 (1080×1350) derive bounds by **proportional scaling from
   portrait**: keep horizontal values, scale vertical values by
   `1350/1920 ≈ 0.703`. Example: `overlay` portrait card =
   `{ x: 24, y: 1280, w: 1032, h: 564 }` → 4:5 card =
   `{ x: 24, y: round(1280 × 0.703), w: 1032, h: round(564 × 0.703) }`
   = `{ x: 24, y: 900, w: 1032, h: 397 }`.

2. **Map the style group to a specific style** by looking at the
   transcript tone — pick the one that best fits, but stay inside the
   user's chosen group. If you're unsure between two specific styles
   inside the group, send a second `AskUserQuestion` with those 2–4
   specific style options.

3. **Resolve final cardCount** from the density answer:

   | user choice | final cardCount |
   |---|---|
   | 自动 (推荐) | the `autoCount` you already computed |
   | 少量 | `max(5, round(autoCount × 0.6))` |
   | 更多 | `round(autoCount × 1.5)` (no upper clamp) |
   | Other = "<n>" (integer) | `max(5, parseInt(n))` |
   | Other = anything else | fall back to `autoCount` |

4. **Auto-pick the video frame** from this table (frames don't ask the
   user — they follow from layout × style):

   | layout | warm-paper styles (academic / whiteboard / editorial / xhs) | clinical styles (audit / swiss / terminal / minimal) | experimental styles (geom / spotlight) |
   |---|---|---|---|
   | `split` | `polaroid` | `hairline` | `clean` |
   | `stack` | `polaroid` | `hairline` | `clean` |
   | `pip` | `clean` (pip pill already has chrome) | `clean` | `clean` |
   | `overlay` | `clean` (full-bleed forbids deco frames) | `clean` | `clean` |

5. **Tell the user what you chose** in one sentence — ratio (+ canvas
   size), layout, specific style, frame, and final cardCount — then
   proceed with the rest of Step 7 (per-card layouts, motion patterns).
6. Record the five values (ratio / layout / style / frame / cardCount)
   in working memory (no schema field needed); you'll reference them
   while writing each card's HTML in Step 8 and while reading the
   matching `references/<dim>/<key>.html` for tokens and structure.

If the user picks an answer via "Other" with a free-text style name not
in the 10-style library, treat it as a hint to design a fresh card
visual yourself, but still anchor on the chosen layout's bounds.

#### Render Strategy Inputs

With ratio / layout / style / cardCount / frame locked from Step 7.0,
the remaining per-card decisions are:

- **Source-video fit inside the GSAP target**: video element has
  `object-fit: cover` and is clipped to `#video-wrap`'s tween bounds.
  If you want NO cropping (e.g. portrait source on landscape canvas
  shouldn't get its top/bottom chopped), aim the tween at a rect that
  matches the source's aspect ratio and let surrounding canvas show
  through (or fill with the card / a backdrop).
- **`card.zone` per card**: derive from your chosen composition layout
  (split → side-panel, stack → lower-third, pip → fullscreen, overlay
  → video-overlay), OR pick a different zone for one-off variants
  (fullscreen for hero / quote, whiteboard-area for dense data).
- **`accentIndex` per card**: each card pulls one of the 5 theme accent
  colors. Vary across cards for rhythm; reuse the same index when two
  cards belong to the same narrative beat.
- **Motion vocabulary**: pick 2–3 repeatable patterns from
  `data-anim` kinds (see the table later) and stick to them so the
  composition feels coherent.

Pick from these `themeId` palettes (use them as `--accent-N` /
`--bg` / `--text` CSS variables in your composition `<style>` block):

| themeId | accent palette (5 colors) | board bg | text |
|---|---|---|---|
| classic | `#1971c2 #e03131 #2f9e44 #e8590c #9c36b5` | `#FFF9E3` (paper) | `#1e1e1e` |
| noir | `#4cc9f0 #f72585 #4ade80 #fb923c #a78bfa` | `#1a1a1a` | `#f1f1f1` |
| mint | `#0077b6 #d62828 #2d6a4f #e76f51 #7209b7` | `#e8faf0` | `#1b4332` |
| craft | `#bf5700 #d62728 #6c757d #e9b54a #3d5a80` | `#f6efe1` | `#2d2d2d` |
| slate | `#0ea5e9 #ef4444 #22c55e #f97316 #a855f7` | `#1e293b` | `#f1f5f9` |
| mono | `#000 #555 #888 #aaa #ccc` | `#fff` | `#000` |

Available fonts (woff2 in `<SKILL_DIR>/assets/fonts/`, staged to work dir in Step 9): `Caveat` (handwriting),
`LXGW WenKai TC` (Chinese hand-script), `Inter` (modern sans), `Virgil`
(geometric hand). Reference via `@font-face` or `font-family` directly.

For inspiration on visual patterns, `<SKILL_DIR>/references/styles/`
ships 10 self-contained reference cards (academic / editorial / minimal
/ spotlight / geom / whiteboard / audit / terminal / swiss / xhs) that
you can copy as starting points — but **do not feel constrained to
match any of these**. Each card is your own design.

#### Visual Design Library (<SKILL_DIR>/references/)

Beyond the composition-level `themeId`, the skill ships a richer **reference
library** at `<SKILL_DIR>/references/` covering three **orthogonal**
visual dimensions you can freely mix:

```
Style  ×  Layout  ×  VideoFrame
 (10)      (4)         (3)
```

| dimension | keys | what it decides |
|---|---|---|
| **style** | `academic` `editorial` `minimal` `spotlight` `geom` `whiteboard` `audit` `terminal` `swiss` `xhs` | the card's visual language — fonts, colors, ornament, layout-within-card |
| **layout** | `split` `stack` `pip` `overlay` | how the source video and the card share the canvas |
| **frame** | `clean` `hairline` `polaroid` | the decorative chrome around the video element |

Read `<SKILL_DIR>/references/DESIGN_INDEX.md`
for the full matrix and a loose decision guide (访谈 / 产品发布 / 数据分析 /
社交剪辑 / 技术教程 / 情绪故事 …). When you decide to use a specific
style / layout / frame, Read the corresponding file:

- `references/styles/<key>.html` — self-contained card fragment with that
  style's CSS tokens (colors, fonts, padding, ornament) and a placeholder
  takeaway. Copy the `.card[data-card-id="ref-<key>"]` style block, rename
  the data-card-id to your card's id, swap the placeholder content for the
  real takeaway, and you're done.
- `references/layouts/<key>.html` — exact `videoBounds` + `cardBounds` for
  both landscape and portrait, with a copy-paste JSON snippet for
  `storyboard.json`'s per-card `layout` field.
- `references/frames/<key>.html` — decorative HTML to add as a sibling of
  `#video-wrap`, plus placement instructions for the composition CSS.

Pick `style × layout × frame` **per card** — you can change all three
between cards as long as the transitions read smoothly. A common rhythm:
open `editorial × overlay × clean`, switch to `audit × split × hairline`
for the data card, close on `whiteboard × pip × polaroid`.

The 10 styles are skill-side design tokens, **not composition-level themes** —
they don't need to be declared in `storyboard.composition`; they live
inside each card's HTML. The `themeId` field can still pick a
composition-level palette (table above) that controls page-body background
and video border chrome.

#### Layout Compositions (Card + Video)

Two coordinated decisions per card define how it shares the canvas with
the source video:

- **`card.zone`** (declared in `storyboard.json`) — one of the 5 schema
  values; resolve it into pixel bounds (per the table in Step 6) when
  you write the card-host wrapper's inline `style` in Step 9.
- **`#video-wrap` bounds at this card's time window** (declared
  imperatively in the composition's GSAP timeline) — the agent tweens
  `#video-wrap` to a target rect for each layout transition.

Schema does NOT store per-card video bounds. `videoTrack.bounds` is
**one-time** at composition level (defaults to full canvas). Video
"moving" between cards is purely a GSAP animation authored in
`index.html`. There is no `card.layout` field — earlier versions of this
doc invented one; the real schema only has `card.zone`.

**4 composition layouts** (from `references/layouts/`) — each is a
recipe pairing a `zone` with a `#video-wrap` tween target:

| composition layout | recommended `card.zone` | GSAP target for `#video-wrap` (landscape 1920×1080) | GSAP target for `#video-wrap` (portrait 1080×1920) | when to use |
|---|---|---|---|---|
| `split` | `side-panel` | `{ left: 960, top: 0, width: 960, height: 1080 }` | `{ left: 0, top: 960, width: 1080, height: 960 }` (bottom half) | speaker + data side-by-side / 50:50 weight |
| `stack` | `lower-third` | `{ left: 14, top: 14, width: 1892, height: 548 }` (top 52%) | `{ left: 0, top: 0, width: 1080, height: 844 }` (top 44%) | speaker on top + summary card below |
| `pip` | `fullscreen` | `{ left: 1480, top: 760, width: 400, height: 300 }` + add `.framed` class | `{ left: 690, top: 28, width: 360, height: 203 }` + add `.framed` | content-heavy card + corner pip |
| `overlay` | `video-overlay` | `{ left: 0, top: 0, width: 1920, height: 1080 }` (full-bleed) | `{ left: 0, top: 0, width: 1080, height: 1920 }` | cinematic / dramatic / glass card on full video |

For 4:5 (1080×1350), scale portrait y/h values by `1350/1920 ≈ 0.703`
(see Step 7.0 Channel A / Channel B `recommendedRatio` resolution
table).

**Other zone values for one-off variants** (still uses `card.zone`; no
fake "layout" field):

| `zone` | resolved bounds | common use |
|---|---|---|
| `fullscreen` | covers whole canvas | hero card, video tweens to hidden/pip |
| `whiteboard-area` | inset 40px margin (landscape) or bottom 45% (portrait) | dense data card, free margins |
| `lower-third` | bottom 30% band | talking-head annotation |
| `side-panel` | right 42% (landscape) or bottom 40% (portrait) | sidebar / "split" recipe |
| `video-overlay` | full canvas; expect transparent card root | glass overlay on full-bleed video |

You can mix recipes per card — choose `card.zone` based on what suits
the moment, then write the GSAP tween for `#video-wrap` between cards.

#### Storyboard Render Contract

`storyboard.json` is an agent-internal planning artifact — no vtake CLI
command parses it. It exists to keep your timing and content decisions
explicit before you write each card's HTML. Stick to the v3-style
shape below so the same outline drives the composition you assemble in
Step 9.

Required structure (see Step 6 for the full example):

- `schemaVersion: 3`
- `composition: { fps, width, height, durationSeconds, layout, themeId, seed }` — note `durationSeconds`/`fps`/`themeId`/`layout` live **inside** `composition`, NOT at top level
- `videoTrack: { sourcePath, startSec, endSec, bounds? }` — video bounds default to full canvas
- `subtitles: { enabled, ... }`
- `cards[]` — each card has the 6 required fields: `id`, `intent`, `startSec`, `endSec`, `accentIndex`, `zone`, `contentHints`

Rules:

- Card times stay inside `composition.durationSeconds` and should not overlap unless intentional (use `data-track-index` to control z-order when they do).
- Visual details live in card HTML fragments (Step 8), NOT in `contentHints`. `contentHints` is your own structured prompt for designing the card; the rendered look is the HTML.
- Keep the storyboard shape stable — even though nothing parses it, you read it back while authoring Step 8/9, and consistency keeps card IDs and timing in sync.
- Agent-side decisions like "I picked overlay × geom × clean" do NOT belong in `storyboard.json` — keep them in working memory and use them when authoring card HTML + GSAP tweens.

**Transparent card backgrounds for cards that share canvas with video.**
When the GSAP tween leaves video visible behind/beside the card (overlay
recipe, pip recipe, or any `card.zone = 'lower-third' | 'video-overlay'`
moment), the card's `.root` MUST NOT paint a full opaque background —
otherwise it occludes the video. Two patterns:

```css
/* Pattern A: transparent root, page body provides the cream backdrop */
html, body { background: var(--bg); }
.card[data-card-id="card-X"] .root { background: transparent; }

/* Pattern B: explicit per-card background ONLY for fullscreen cards */
.card[data-card-id="card-hero"] .root { background: var(--bg); }
.card[data-card-id="card-overlay"] .root { background: transparent; }
```

For `side-panel`-zone cards (split recipe), the card-host is already
only half the canvas, so an opaque card bg is fine — it only covers its
half.

### 8. Write Each Card's HTML

Create `$WORK_DIR/public/cards/{card-id}.html` for each card. Each file
contains a single rooted HTML fragment that follows this contract:

#### Card HTML Contract

```html
<div class="card" data-card-id="{cardId}">
  <style>
    /* MUST: every rule starts with .card[data-card-id="{cardId}"] */
    .card[data-card-id="card-01"] .root {
      width: 100%; height: 100%;
      display: flex; ...;
      font-family: 'Caveat', 'LXGW WenKai TC', serif;
      color: var(--text);
      background: var(--bg);
    }
    .card[data-card-id="card-01"] .title { font-size: 84px; ... }
  </style>

  <div class="root">
    <h1 id="card-01-title"
        data-anim="kinetic-chars"
        data-anim-at="0.3"
        data-anim-duration="0.5"
        data-anim-stagger="0.04"
        data-anim-pattern="pop">
      <span class="char">字</span>
      <span class="char">幕</span>
    </h1>
    <div id="card-01-line"
         data-anim="grow-x"
         data-anim-at="0.65"
         data-anim-duration="0.5"
         data-anim-target-w="420"
         style="width:0;height:8px;background:var(--accent-0);border-radius:4px;"></div>
  </div>
</div>
```

**Hard rules** (`hyperframes` lint will reject violations):

- Single root `<div class="card" data-card-id="{cardId}">`
- Inline `<style>` rules MUST be prefixed with the scope selector above
- **No `<script>` tags**
- **No external URLs** in `src=` / `href=` (no CDN, no remote fonts)
- **No inline event handlers** (`onclick=` etc.)
- All assets via relative paths into the same `public/` directory
- Colors via `var(--accent-N)` etc. for portability across themes

**Animations are declared, not coded.** Use `data-anim-*` attributes
only; never write `<script>` to animate. You compile every `data-anim-*`
declaration into the single master GSAP timeline in Step 9.

#### Card Sizing — Mobile-First in Portrait

The 10 `references/styles/*.html` are sized for a **1920×1080 landscape**
preview. When `storyboard.layout = "portrait"` (1080×1920, the dominant
case for social / mobile), **scale every visual size up** — phones hold
the screen close, and the same pixel count reads smaller than on a
landscape TV-style canvas.

| token | landscape baseline | **portrait target** | scale |
|---|---|---|---|
| title (h1/h2 hero) | 64–96px | **88–132px** | ×1.35 |
| detail / body | 24–30px | **30–40px** | ×1.30 |
| kicker / chip label | 14–16px | **18–22px** | ×1.30 |
| timecode / meta | 12–14px | **16–18px** | ×1.30 |
| data block primary number | 48–60px | **64–88px** | ×1.40 |
| line-height multiplier | 1.05–1.5 | same | (don't scale) |

**Rule of thumb:** `portraitPx = round(landscapePx × 1.3)`, then floor
to a nearby 4px multiple for visual rhythm. Hero headlines may go up to
×1.4; small meta text stays at ×1.2 to avoid crowding.

Padding **shrinks slightly** in portrait — the card is narrower so big
landscape padding (40–64px) eats too much width. Use 24–36px horizontal
padding in portrait.

If you're producing a single card that must work in **both** layouts,
prefer a `@container` query on the card root over hard-coding sizes:

```css
.card[data-card-id="X"] .root { container-type: inline-size; }
.card[data-card-id="X"] .title { font-size: clamp(64px, 8.5cqi, 132px); }
.card[data-card-id="X"] .detail { font-size: clamp(24px, 3.2cqi, 40px); }
```

But for most cards, a single layout choice is fine — just pick the size
table column that matches the storyboard's `layout` field.

#### Available `data-anim` Kinds

| kind | use for | key params |
|---|---|---|
| `fade-in` | enter | `at`, `duration`, `ease?` |
| `fade-out` | exit | `at`, `duration`, `ease?` |
| `slide-in` | slide enter | `at`, `duration`, `from=left\|right\|top\|bottom`, `distance` |
| `kinetic-chars` | per-char pop | `at`, `duration`, `stagger`, `pattern=pop\|fade` — element needs `<span class="char">` children |
| `typewriter` | per-char fade | same as kinetic-chars but slower default stagger |
| `count-up` | animate number | `at`, `duration`, `from`, `to`, `format=.0f\|.1f\|.2f\|,d` |
| `draw-path` | SVG path reveal | `at`, `duration` — element should be a `<path>` |
| `grow-y` | bar height | `at`, `duration`, `target-h` (px) — element starts `height:0` |
| `grow-x` | bar width | `at`, `duration`, `target-w` (px) — element starts `width:0` |
| `scale-pop` | pop entrance | `at`, `duration` |
| `blur-in` | unfocused → focused | `at`, `duration` |
| `mask-reveal` | clip reveal | `at`, `duration`, `direction=left\|right\|top\|bottom` |
| `morph-to` | tween any CSS | `at`, `duration`, `props='{...JSON...}'` |

`data-anim-at` is **seconds relative to the card's startSec** — when you
compile each declaration into the GSAP timeline in Step 9, add the
card's `startSec` to get the absolute time and quantize to 1/fps.

#### card-cta-vtake: Fixed Brand Outro Card

Save this **fixed template** as `$WORK_DIR/public/cards/card-cta-vtake.html`
— do not modify. Design language: **"Editorial Cinema"** — viewfinder corner
marks, top/bottom film-credit meta strips, a dual rotating ring around the
vT mark, a diamond-flanked divider, an italic tagline, and amber sparkle
particles. The entrance sequence completes in **1.00s**; ambient breathing
(ring rotation, particle pulse) continues for the remaining ~1.0s.

```html
<!-- public/cards/card-cta-vtake.html — FIXED TEMPLATE, do not alter -->
<div class="card" data-card-id="card-cta-vtake">
  <style>
    .card[data-card-id="card-cta-vtake"] .root {
      width: 100%; height: 100%;
      background:
        radial-gradient(ellipse 60% 80% at 50% 50%, rgba(232,144,52,0.12) 0%, transparent 55%),
        radial-gradient(ellipse 120% 40% at 50% 100%, rgba(201,169,97,0.10) 0%, transparent 60%),
        #0D0B08;
      position: relative; overflow: hidden;
      container-type: inline-size;
      font-family: 'Inter', ui-sans-serif, sans-serif;
      color: #F5EFE1;
    }
    /* fractal noise overlay — kills the "plastic" look */
    .card[data-card-id="card-cta-vtake"] .root::after {
      content: '';
      position: absolute; inset: 0;
      background-image: url("data:image/svg+xml,%3Csvg viewBox='0 0 200 200' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.85' numOctaves='2' stitchTiles='stitch'/%3E%3CfeColorMatrix values='0 0 0 0 1  0 0 0 0 1  0 0 0 0 1  0 0 0 0.06 0'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23n)'/%3E%3C/svg%3E");
      mix-blend-mode: overlay;
      opacity: 0.5;
      pointer-events: none;
    }
    /* viewfinder corner marks (1.5px hairline) */
    .card[data-card-id="card-cta-vtake"] .corner {
      position: absolute;
      width: 3.5cqi; height: 3.5cqi;
      border-color: rgba(245,239,225,0.18);
      pointer-events: none;
    }
    .card[data-card-id="card-cta-vtake"] .corner.tl { top: 3.5cqi; left: 3.5cqi; border-top: 1.5px solid; border-left: 1.5px solid; }
    .card[data-card-id="card-cta-vtake"] .corner.tr { top: 3.5cqi; right: 3.5cqi; border-top: 1.5px solid; border-right: 1.5px solid; }
    .card[data-card-id="card-cta-vtake"] .corner.bl { bottom: 3.5cqi; left: 3.5cqi; border-bottom: 1.5px solid; border-left: 1.5px solid; }
    .card[data-card-id="card-cta-vtake"] .corner.br { bottom: 3.5cqi; right: 3.5cqi; border-bottom: 1.5px solid; border-right: 1.5px solid; }
    /* top film-credit strip */
    .card[data-card-id="card-cta-vtake"] .top-meta {
      position: absolute; top: 4.8cqi; left: 0; right: 0;
      display: flex; justify-content: center;
      gap: clamp(14px, 2cqi, 36px);
      font-size: clamp(9px, 0.85cqi, 16px);
      font-weight: 500;
      letter-spacing: 0.36em;
      text-transform: uppercase;
      color: rgba(245,239,225,0.18);
    }
    .card[data-card-id="card-cta-vtake"] .top-meta .sep { color: #E89034; opacity: 0.7; }
    /* bottom film-credit strip */
    .card[data-card-id="card-cta-vtake"] .bot-meta {
      position: absolute; bottom: 5cqi; left: 0; right: 0;
      display: flex; justify-content: space-between;
      padding: 0 7cqi;
      font-size: clamp(9px, 0.8cqi, 15px);
      font-weight: 500;
      letter-spacing: 0.32em;
      text-transform: uppercase;
      color: rgba(245,239,225,0.18);
    }
    .card[data-card-id="card-cta-vtake"] .bot-meta .right { color: #E89034; opacity: 0.85; }
    /* center stage */
    .card[data-card-id="card-cta-vtake"] .stage {
      position: absolute; top: 50%; left: 50%;
      transform: translate(-50%, -50%);
      display: flex; flex-direction: column; align-items: center;
      z-index: 2;
    }
    /* "Powered by" row with short flanking lines */
    .card[data-card-id="card-cta-vtake"] .powered-row {
      display: flex; align-items: center;
      gap: clamp(10px, 1.4cqi, 24px);
      margin-bottom: clamp(24px, 3cqi, 56px);
    }
    .card[data-card-id="card-cta-vtake"] .powered-row .line {
      height: 1px;
      background: rgba(245,239,225,0.18);
      width: 0;
    }
    .card[data-card-id="card-cta-vtake"] .powered-row .label {
      font-size: clamp(11px, 1.05cqi, 20px);
      font-weight: 600;
      letter-spacing: 0.46em;
      text-transform: uppercase;
      color: rgba(245,239,225,0.40);
    }
    /* vT mark wrapper + dual concentric rings */
    .card[data-card-id="card-cta-vtake"] .mark-wrap {
      position: relative;
      width: clamp(160px, 15cqi, 280px);
      height: clamp(160px, 15cqi, 280px);
      display: flex; align-items: center; justify-content: center;
    }
    .card[data-card-id="card-cta-vtake"] .mark-ring {
      position: absolute; inset: 0;
      border-radius: 50%;
      border: 1.5px dashed rgba(232,144,52,0.22);
      animation: ctaRingSpin 32s linear infinite;
    }
    .card[data-card-id="card-cta-vtake"] .mark-ring.inner {
      inset: 8%;
      border: 1px solid rgba(245,239,225,0.10);
      animation: ctaRingSpin 32s linear infinite reverse;
    }
    .card[data-card-id="card-cta-vtake"] .vtake-mark {
      width: 60%; height: auto;
      color: #F5EFE1;
      position: relative; z-index: 1;
      overflow: visible;
    }
    /* main brand name */
    .card[data-card-id="card-cta-vtake"] .vtake-name {
      font-size: clamp(48px, 6cqi, 112px);
      font-weight: 900;
      letter-spacing: -0.05em;
      margin-top: clamp(20px, 2.5cqi, 48px);
      line-height: 1;
      overflow: hidden;
      clip-path: inset(0 100% 0 0);
    }
    .card[data-card-id="card-cta-vtake"] .name-accent { color: #E89034; }
    /* decorative divider: ── ◆ ── */
    .card[data-card-id="card-cta-vtake"] .divider {
      display: flex; align-items: center;
      gap: clamp(6px, 0.7cqi, 14px);
      margin-top: clamp(18px, 2.2cqi, 40px);
    }
    .card[data-card-id="card-cta-vtake"] .divider .seg {
      width: 0; height: 1px;
      background: rgba(232,144,52,0.5);
    }
    .card[data-card-id="card-cta-vtake"] .divider .diamond {
      width: clamp(4px, 0.45cqi, 8px);
      height: clamp(4px, 0.45cqi, 8px);
      background: #E89034;
      transform: rotate(45deg) scale(0);
    }
    /* tagline below divider */
    .card[data-card-id="card-cta-vtake"] .tagline {
      margin-top: clamp(14px, 1.8cqi, 32px);
      font-size: clamp(10px, 0.9cqi, 17px);
      font-weight: 500;
      letter-spacing: 0.56em;
      text-transform: uppercase;
      color: rgba(245,239,225,0.40);
      padding-left: 0.5em;
    }
    /* amber sparkle particles */
    .card[data-card-id="card-cta-vtake"] .spark {
      position: absolute;
      width: clamp(2px, 0.2cqi, 3px);
      height: clamp(2px, 0.2cqi, 3px);
      background: #E89034;
      border-radius: 50%;
      box-shadow: 0 0 14px #E89034;
      opacity: 0;
    }
    .card[data-card-id="card-cta-vtake"] .spark.s1 { top: 32%; left: 22%;  animation: ctaSparkle 3.5s ease-in-out 1.0s infinite; }
    .card[data-card-id="card-cta-vtake"] .spark.s2 { top: 64%; left: 76%;  animation: ctaSparkle 4.2s ease-in-out 1.4s infinite; }
    .card[data-card-id="card-cta-vtake"] .spark.s3 { top: 26%; right: 22%; animation: ctaSparkle 3.8s ease-in-out 1.6s infinite; }
    .card[data-card-id="card-cta-vtake"] .spark.s4 { bottom: 32%; left: 38%; animation: ctaSparkle 4.6s ease-in-out 0.8s infinite; }
    .card[data-card-id="card-cta-vtake"] .spark.s5 { top: 48%; right: 14%; animation: ctaSparkle 3.2s ease-in-out 2.0s infinite; }
    @keyframes ctaRingSpin { from { transform: rotate(0deg); } to { transform: rotate(360deg); } }
    @keyframes ctaSparkle {
      0%, 100% { opacity: 0; transform: scale(0.5); }
      50%      { opacity: 0.85; transform: scale(1.3); }
    }
  </style>
  <div class="root">
    <!-- Viewfinder corners -->
    <span class="corner tl" id="cta-corner-tl"
          data-anim="fade-in" data-anim-at="0.20" data-anim-duration="0.50"></span>
    <span class="corner tr" id="cta-corner-tr"
          data-anim="fade-in" data-anim-at="0.20" data-anim-duration="0.50"></span>
    <span class="corner bl" id="cta-corner-bl"
          data-anim="fade-in" data-anim-at="0.20" data-anim-duration="0.50"></span>
    <span class="corner br" id="cta-corner-br"
          data-anim="fade-in" data-anim-at="0.20" data-anim-duration="0.50"></span>

    <!-- Top film-credit strip -->
    <div class="top-meta" id="cta-top-meta"
         data-anim="fade-in" data-anim-at="0.25" data-anim-duration="0.50">
      <span>VTAKE&trade;</span>
      <span class="sep">&diams;</span>
      <span>Cinematic Postproduction</span>
      <span class="sep">&diams;</span>
      <span>Est. 2026</span>
    </div>

    <!-- Bottom film-credit strip -->
    <div class="bot-meta" id="cta-bot-meta"
         data-anim="fade-in" data-anim-at="0.30" data-anim-duration="0.50">
      <span>&copy; 2026 vTake Studio</span>
      <span class="right">vtake.app</span>
    </div>

    <!-- Sparkle particles (pure CSS ambient animation, no entry tween) -->
    <span class="spark s1"></span>
    <span class="spark s2"></span>
    <span class="spark s3"></span>
    <span class="spark s4"></span>
    <span class="spark s5"></span>

    <div class="stage">
      <!-- "Powered by" with grow-x flanking lines -->
      <div class="powered-row" id="cta-powered-row">
        <span class="line" id="cta-line-l"
              data-anim="grow-x" data-anim-at="0.18"
              data-anim-duration="0.40" data-anim-target-w="56"></span>
        <span class="label" id="cta-powered-label"
              data-anim="fade-in" data-anim-at="0.15"
              data-anim-duration="0.45">Powered by</span>
        <span class="line" id="cta-line-r"
              data-anim="grow-x" data-anim-at="0.18"
              data-anim-duration="0.40" data-anim-target-w="56"></span>
      </div>

      <!-- vT mark with dual rings + 3 stroked paths + amber dot -->
      <div class="mark-wrap">
        <span class="mark-ring" id="cta-ring-outer"
              data-anim="fade-in" data-anim-at="0.55" data-anim-duration="0.50"></span>
        <span class="mark-ring inner" id="cta-ring-inner"
              data-anim="fade-in" data-anim-at="0.60" data-anim-duration="0.50"></span>
        <svg class="vtake-mark" viewBox="0 0 200 130" fill="none"
             aria-label="vTake" role="img">
          <g transform="skewX(-14) translate(16 0)"
             stroke="currentColor" stroke-width="14"
             stroke-linecap="round" stroke-linejoin="round">
            <path id="cta-path-v"     d="M24 32 L56 100 L88 32"
                  data-anim="draw-path" data-anim-at="0.05" data-anim-duration="0.40"/>
            <path id="cta-path-tbar"  d="M100 32 L164 32"
                  data-anim="draw-path" data-anim-at="0.30" data-anim-duration="0.25"/>
            <path id="cta-path-tstem" d="M132 32 L132 100"
                  data-anim="draw-path" data-anim-at="0.50" data-anim-duration="0.25"/>
          </g>
          <circle id="cta-dot" cx="48" cy="118" r="7.2" fill="#E89034"
                  data-anim="scale-pop" data-anim-at="0.65" data-anim-duration="0.25"/>
        </svg>
      </div>

      <!-- Main brand name reveals left → right -->
      <div id="cta-vtake-name" class="vtake-name"
           data-anim="mask-reveal" data-anim-at="0.40"
           data-anim-duration="0.45" data-anim-direction="bottom">
        <span class="name-accent">v</span>Take
      </div>

      <!-- Decorative divider: ── ◆ ── -->
      <div class="divider">
        <span class="seg" id="cta-seg-l"
              data-anim="grow-x" data-anim-at="0.75"
              data-anim-duration="0.30" data-anim-target-w="56"></span>
        <span class="diamond" id="cta-diamond"
              data-anim="scale-pop" data-anim-at="0.85" data-anim-duration="0.30"></span>
        <span class="seg" id="cta-seg-r"
              data-anim="grow-x" data-anim-at="0.75"
              data-anim-duration="0.30" data-anim-target-w="56"></span>
      </div>

      <!-- Italic tagline -->
      <div class="tagline" id="cta-tagline"
           data-anim="fade-in" data-anim-at="0.90" data-anim-duration="0.45">
        Visual Take &middot; Reimagined
      </div>
    </div>
  </div>
</div>
```

**Animation timeline (1.00s entrance, all `data-anim-at` values are relative
to `card-cta-vtake.startSec`):**

| At | Element | Animation | Dur |
|---|---|---|---|
| 0.05 | V-shape stroke | draw-path | 0.40 |
| 0.15 | "Powered by" label | fade-in | 0.45 |
| 0.18 | flanking lines | grow-x (→56px) | 0.40 |
| 0.20 | viewfinder corners ×4 | fade-in | 0.50 |
| 0.25 | top film-credit strip | fade-in | 0.50 |
| 0.30 | bottom film-credit strip | fade-in | 0.50 |
| 0.30 | T-bar stroke | draw-path | 0.25 |
| 0.40 | "vTake" main name | mask-reveal | 0.45 |
| 0.50 | T-stem stroke | draw-path | 0.25 |
| 0.55 | outer dashed ring | fade-in (+ 32s spin) | 0.50 |
| 0.60 | inner solid ring | fade-in (+ 32s reverse) | 0.50 |
| 0.65 | amber dot | scale-pop (elastic) | 0.25 |
| 0.75 | divider segments | grow-x (→56px) | 0.30 |
| 0.85 | divider diamond | scale-pop | 0.30 |
| 0.90 | tagline | fade-in | 0.45 |
| 1.00+ | ambient | ring spin + sparkle loop | — |

### 9. Assemble the Composition HTML

Stage the assets and write `$WORK_DIR/public/index.html`:

```bash
# SKILL_DIR is injected by the host ("Base directory for this skill: …")
SKILL_DIR="<SKILL_DIR>"

mkdir -p "$WORK_DIR/public/fonts" "$WORK_DIR/public/vendor" "$WORK_DIR/public/cards"
cp -n "$SKILL_DIR/assets/fonts/"*            "$WORK_DIR/public/fonts/"
cp -n "$SKILL_DIR/assets/vendor/gsap.min.js" "$WORK_DIR/public/vendor/"
# stage the input video so the composition can reference it by relative path
ln -f "$VIDEO_PATH" "$WORK_DIR/public/input-video.mp4" 2>/dev/null \
  || cp "$VIDEO_PATH" "$WORK_DIR/public/input-video.mp4"
```

#### Composition Template

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<style>
@font-face { font-family: 'Caveat'; src: url('fonts/Caveat-400-latin.woff2') format('woff2'); font-weight: 400; font-display: block; }
@font-face { font-family: 'Caveat'; src: url('fonts/Caveat-700-latin.woff2') format('woff2'); font-weight: 700; font-display: block; }
@font-face { font-family: 'LXGW WenKai TC'; src: url('fonts/LXGWWenKaiTC-400-latin.woff2') format('woff2'); font-weight: 400; font-display: block; }
@font-face { font-family: 'Inter'; src: url('fonts/Inter-400-latin.woff2') format('woff2'); font-weight: 400; font-display: block; }
@font-face { font-family: 'Inter'; src: url('fonts/Inter-700-latin.woff2') format('woff2'); font-weight: 700; font-display: block; }
@font-face { font-family: 'Virgil'; src: url('fonts/Virgil.woff2') format('woff2'); font-display: block; }

:root {
  /* Pick from the themeId palette table in Step 7 — example: classic */
  --bg: #FFF9E3;
  --text: #1e1e1e;
  --accent-0: #1971c2;
  --accent-1: #e03131;
  --accent-2: #2f9e44;
  --accent-3: #e8590c;
  --accent-4: #9c36b5;
  --font-family: 'Caveat', 'LXGW WenKai TC', serif;
}
* { box-sizing: border-box; }
/* Body font-family MUST list concrete font names (not just var(--font-family)) —
   the HyperFrames renderer's static analyzer doesn't expand CSS variables when
   resolving fonts, so a var-only chain triggers `font_family_without_font_face`
   lint and falls back to a generic. Use the concrete chain here; cards that
   want the theme font can still reference var(--font-family) internally. */
html, body { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #000;
             font-family: 'Inter', 'Caveat', 'LXGW WenKai TC', ui-sans-serif, system-ui, sans-serif; }
#stage { position: relative; width: 100%; height: 100%; overflow: hidden; }

/* video-wrapper holds the source video. Its position / size are animated
   over time by the master timeline (one tween per layout transition). */
.video-wrapper {
  position: absolute;
  left: 0; top: 0; width: 1920px; height: 1080px;
  overflow: hidden;
  border-radius: 0;
  box-shadow: none;
}
.video-wrapper video { width: 100%; height: 100%; object-fit: cover; }

.card-host { position: absolute; pointer-events: none; overflow: hidden; }
.card-host .card { position: relative; width: 100%; height: 100%; overflow: hidden; }
.card-host .char { display: inline-block; visibility: visible; }

/* Subtle drop shadow + rounded corners for non-fullscreen video framings */
.video-wrapper.framed {
  border-radius: 16px;
  box-shadow: 0 12px 40px rgba(0,0,0,0.35);
}
</style>
</head>
<body>
<div
  id="stage"
  data-composition-id="vtake"
  data-start="0"
  data-duration="121.2"
  data-fps="30"
  data-width="1920"
  data-height="1080"
>
  <!-- Layer 1: source video — initial position matches card-01's layout -->
  <div class="video-wrapper" id="video-wrap">
    <video id="bg-video"
           src="input-video.mp4"
           muted playsinline
           data-start="0"
           data-duration="121.2"
           data-track-index="1"></video>
  </div>

  <!-- Layer 2: each card-host sits at the bounds dictated by its layout. -->
  <!-- IMPORTANT: every card-host MUST carry BOTH "card-host" and "clip" classes. -->
  <!--   - "card-host"  → our positioning + pointer-events styles                 -->
  <!--   - "clip"       → HyperFrames runtime uses this to enforce visibility     -->
  <!--                    only during data-start … data-start+data-duration.      -->
  <!--                    Without "clip" the host stays visible the whole video   -->
  <!--                    (lint: timed_element_missing_clip_class).               -->
  <!-- Example: card-01 with zone="fullscreen" → card-host covers (0,0,1920,1080) -->
  <div class="card-host clip"
       data-card-id="card-01"
       data-start="1.0000"
       data-duration="6.5000"
       data-track-index="2"
       style="left:0;top:0;width:1920px;height:1080px;visibility:hidden;opacity:0;">
    <!-- paste the contents of public/cards/card-01.html here -->
  </div>

  <!-- Example: card-02 with zone="side-panel" (split composition layout) → card on left half -->
  <div class="card-host clip"
       data-card-id="card-02"
       data-start="8.0000"
       data-duration="12.0000"
       data-track-index="2"
       style="left:0;top:0;width:960px;height:1080px;visibility:hidden;opacity:0;">
    <!-- card-02 HTML -->
  </div>

  <!-- ...one "card-host clip" per card with inline bounds matching resolveZoneBounds(card.zone)... -->

  <script src="vendor/gsap.min.js"></script>
  <script>
  (function(){
    // count-up formatter helper
    window.__fmt = function(v, fmt) {
      if (typeof fmt === 'string' && /^\.[0-9]+f$/.test(fmt)) {
        return Number(v).toFixed(Number(fmt.slice(1, -1)));
      }
      if (fmt === ',d') return Math.round(v).toLocaleString();
      return String(Math.round(v));
    };

    const tl = window.gsap.timeline({ paused: true });

    // ── Card lifecycle (one block per card) ──
    // Example for card-01 [1.0, 7.5] with kinetic-chars at +0.3, grow-x at +0.65:

    // Enter (fade in over 0.4s)
    tl.set('.card-host[data-card-id="card-01"]',  { visibility: 'visible' }, 1.0000);
    tl.fromTo('.card-host[data-card-id="card-01"]',
              { opacity: 0 }, { opacity: 1, duration: 0.4000, ease: 'power2.out' }, 1.0000);

    // Card-internal anims (compile each data-anim-* declaration here)
    tl.from('.card[data-card-id="card-01"] #card-01-title .char',
            { opacity: 0, y: 8, scale: 0.8, duration: 0.5000, ease: 'power2.out', stagger: 0.0400 },
            1.3000);
    tl.fromTo('.card[data-card-id="card-01"] #card-01-line',
              { width: 0 }, { width: 420, duration: 0.5000, ease: 'power2.out' }, 1.6500);

    // Exit (fade out over 0.35s, ending at endSec)
    tl.to('.card-host[data-card-id="card-01"]',
          { opacity: 0, duration: 0.3500, ease: 'power2.in' }, 7.1500);
    tl.set('.card-host[data-card-id="card-01"]', { visibility: 'hidden' }, 7.5000);

    // ── Video framing transitions ──
    // When the next card uses a different composition layout, animate the
    // video-wrapper to its new bounds. Example: card-01 = fullscreen
    // (video hidden behind), card-02 = split composition (zone="side-panel"
    // → video on right, card on left).

    // Card-02 enters at 8.0s with the split composition. Animate video to
    // the right half during the card-01 → card-02 gap (between 7.5 and 8.0s).
    tl.set('#video-wrap', { className: 'video-wrapper framed' }, 7.5);
    tl.to('#video-wrap',
          { left: 960, top: 0, width: 960, height: 1080,
            duration: 0.6, ease: 'power2.inOut' }, 7.5);

    // Card-02 enter — same pattern as card-01
    tl.set('.card-host[data-card-id="card-02"]', { visibility: 'visible' }, 8.0);
    tl.fromTo('.card-host[data-card-id="card-02"]',
              { opacity: 0 }, { opacity: 1, duration: 0.4, ease: 'power2.out' }, 8.0);
    // ...card-02 internal anims...

    // ── repeat for each card; if the NEXT card's layout differs,
    //    insert another tl.to('#video-wrap', ...) tween before its enter ──

    window.__timelines = window.__timelines || {};
    window.__timelines["vtake"] = tl;
  })();
  </script>
</div>
</body>
</html>
```

#### GSAP Statement Cheat Sheet

Compile each `data-anim` attribute into a GSAP statement. Times are
**absolute seconds** = card.startSec + data-anim-at, quantized to 1/fps.
Selector is `.card[data-card-id="X"] #elementId`.

| data-anim | GSAP statement template |
|---|---|
| `fade-in` | `tl.fromTo(SEL, { opacity: 0 }, { opacity: 1, duration: D, ease: 'power2.out' }, T);` |
| `fade-out` | `tl.to(SEL, { opacity: 0, duration: D, ease: 'power2.in' }, T);` |
| `slide-in` (from=left, dist=80) | `tl.fromTo(SEL, { opacity: 0, x: -80 }, { opacity: 1, x: 0, duration: D, ease: 'power2.out' }, T);` |
| `kinetic-chars` (pop) | `tl.from(SEL + ' .char', { opacity: 0, y: 8, scale: 0.8, duration: D, ease: 'power2.out', stagger: S }, T);` |
| `count-up` | `(function(){const o={v:FROM};tl.to(o,{v:TO,duration:D,ease:'power2.out',onUpdate:function(){const el=document.querySelector(SEL);if(el)el.textContent=__fmt(o.v,'FMT');}},T);})();` |
| `draw-path` | `(function(){const el=document.querySelector(SEL);if(!el)return;const L=el.getTotalLength();tl.set(SEL,{strokeDasharray:L,strokeDashoffset:L},T);tl.to(SEL,{strokeDashoffset:0,duration:D,ease:'power2.inOut'},T);})();` |
| `grow-x` (target-w=W) | `tl.fromTo(SEL, { width: 0 }, { width: W, duration: D, ease: 'power2.out' }, T);` |
| `grow-y` (target-h=H) | `tl.fromTo(SEL, { height: 0 }, { height: H, duration: D, ease: 'power2.out' }, T);` |
| `scale-pop` | `tl.fromTo(SEL, { opacity: 0, scale: 0.6 }, { opacity: 1, scale: 1, duration: D, ease: 'back.out(1.6)' }, T);` |
| `mask-reveal` (direction=left) | `tl.fromTo(SEL, { clipPath: 'inset(0 100% 0 0)' }, { clipPath: 'inset(0 0 0 0)', duration: D, ease: 'power2.inOut' }, T);` |

Quantize: `T = Math.round(absSec * fps) / fps`. At 30fps the smallest
step is `1/30 ≈ 0.0333s`; rounding to 4 decimals (`.toFixed(4)`) is fine
inside the JS literal.

#### Video Framing Reference (per `layout` value)

The selector for the video container is `#video-wrap`. Animate its
bounds between cards using `tl.to('#video-wrap', { ...bounds }, T)`.
Initial bounds should be set inline on the element to match card-01's
layout. Pick a transition duration of 0.5–0.7s with `ease: 'power2.inOut'`.

**Decorative frames** (`clean` / `hairline` / `polaroid`) sit as a
**sibling** of `#video-wrap` and follow it through layout transitions.
See
[`references/frames/`](references/frames/) for each frame's placement
HTML, suggested CSS, and which layouts it pairs with. Quick rule:
`overlay` layout suppresses decorative frames (the full-bleed video
clashes with chrome); PiP layouts already have their own pill treatment
(border-radius + white ring + shadow), so add a decorative frame only on
top of `split` / `stack`.

**GSAP target lookup table** for `#video-wrap` per composition layout
(landscape 1920×1080 — for portrait & 4:5 see `references/layouts/*.html`
which list all three ratios):

| composition layout | typical card.zone | `#video-wrap` GSAP target | extra css class |
|---|---|---|---|
| `split` | `side-panel` | `{ left: 960, top: 0, width: 960, height: 1080 }` | — |
| `stack` | `lower-third` | `{ left: 14, top: 14, width: 1892, height: 548 }` (top 52%) | — |
| `pip` (bottom-right) | `fullscreen` | `{ left: 1480, top: 760, width: 400, height: 300 }` | `pip-pill` (border-radius + ring + shadow) |
| `pip` (top-left) | `fullscreen` | `{ left: 40, top: 40, width: 400, height: 300 }` | `pip-pill` |
| `overlay` (video full-bleed) | `video-overlay` | `{ left: 0, top: 0, width: 1920, height: 1080 }` (no change from default) | — |
| **hide video** (pure-graphic moment) | `fullscreen` | `{ opacity: 0 }` (or move off-canvas) | — |

To toggle the pip-pill chrome (border-radius + white ring + drop shadow)
when entering or leaving a pip moment:

```js
// Enter pip — add chrome
tl.set('#video-wrap', { className: 'video-wrapper pip-pill' }, T);
tl.to('#video-wrap', { left: 1480, top: 760, width: 400, height: 300,
                       duration: 0.6, ease: 'power2.inOut' }, T);

// Leave pip — back to clean full-bleed
tl.set('#video-wrap', { className: 'video-wrapper' }, T_NEXT);
tl.to('#video-wrap', { left: 0, top: 0, width: 1920, height: 1080,
                       duration: 0.6, ease: 'power2.inOut' }, T_NEXT);
```

**Card-host bounds match the zone**. Resolve the card's `zone` into
pixel bounds using the table at the top of Step 6, then write those
into the card-host's inline `style="left:Xpx;top:Ypx;width:Wpx;
height:Hpx;..."`. For `video-overlay` zone (overlay recipe), the
card-host fills the full canvas — your CSS inside `.card .root`
decides where the actual visible card sits.

#### HyperFrames Layout / Animation QA Rules

- Build each card's static hero frame first: the moment where the card is fully visible and readable.
- Confirm video, cards, subtitles/captions, and diagrams do not unintentionally overlap.
- Confirm hidden video areas are clipped by the frame and not visible outside intended bounds.
- Register one paused master timeline as `window.__timelines["vtake"]`.
- Build timelines synchronously at page load; no `async`, `setTimeout`, Promises, or media `play()` calls.
- Do not use `Math.random()` or `Date.now()` in render paths.
- Do not use `repeat: -1`; calculate finite repeats from the video duration.
- Prefer GSAP transforms and opacity (`x`, `y`, `scale`, `rotation`, `opacity`) over layout properties (`top`, `left`, `width`, `height`) for motion.
- Animate wrappers such as `#video-wrap`, not the video element dimensions directly.
- Avoid animating the same property on the same element from multiple timelines at the same time.
- Use `data-track-index`, not `data-layer`; use `data-duration`, not `data-end`.
- Every timed element (`card-host`, sub-composition, etc.) MUST include `class="clip"` alongside its own classes — e.g. `class="card-host clip"`. The HyperFrames runtime uses `.clip` to gate visibility to the `data-start … data-start+data-duration` window. Without it the element is visible for the whole video (lint: `timed_element_missing_clip_class`).
- For body / global `font-family`, list **concrete font names** (`'Inter', 'Caveat', …`) — not a CSS variable like `var(--font-family)`. The HyperFrames font resolver doesn't expand CSS vars during static analysis (lint: `font_family_without_font_face`). Cards may still use `var(--font-family)` internally since their `@font-face` declarations are loaded.

#### card-cta-vtake: Fixed GSAP Animation Block

At the end of the GSAP timeline block (just before the final `window.__timelines`
registration), append this fixed code block. Replace `CTA_START` with
`card-cta-vtake.startSec` and `CTA_END` with `card-cta-vtake.endSec`:

```js
// ── card-cta-vtake: Editorial Cinema brand outro (2.0s total) ──
// Sequence: 1.0s entrance · 0.7s hold (CSS ambient breathing) · 0.3s fade-out

const PREFIX = '.card[data-card-id="card-cta-vtake"]';

// Fade source video out at the start of the CTA
tl.to('#video-wrap', { opacity: 0, duration: 0.30, ease: 'power2.in' }, CTA_START);

// Card host enter
tl.set('.card-host[data-card-id="card-cta-vtake"]', { visibility: 'visible' }, CTA_START);
tl.fromTo('.card-host[data-card-id="card-cta-vtake"]',
          { opacity: 0 }, { opacity: 1, duration: 0.30, ease: 'power2.out' }, CTA_START);

// V-shape stroke draw
(function(){
  const sel = PREFIX + ' #cta-path-v';
  const el = document.querySelector(sel); if(!el) return;
  const L = el.getTotalLength();
  tl.set(sel, { strokeDasharray: L, strokeDashoffset: L }, CTA_START + 0.05);
  tl.to(sel, { strokeDashoffset: 0, duration: 0.40, ease: 'power2.inOut' }, CTA_START + 0.05);
})();

// "Powered by" label
tl.fromTo(PREFIX + ' #cta-powered-label',
          { opacity: 0 }, { opacity: 1, duration: 0.45, ease: 'power2.out' }, CTA_START + 0.15);

// Flanking lines grow-x
tl.fromTo(PREFIX + ' #cta-line-l',
          { width: 0 }, { width: 56, duration: 0.40, ease: 'power2.out' }, CTA_START + 0.18);
tl.fromTo(PREFIX + ' #cta-line-r',
          { width: 0 }, { width: 56, duration: 0.40, ease: 'power2.out' }, CTA_START + 0.18);

// Viewfinder corners ×4 fade-in
['tl','tr','bl','br'].forEach(pos => {
  tl.fromTo(PREFIX + ' #cta-corner-' + pos,
            { opacity: 0, scale: 0.7 },
            { opacity: 1, scale: 1, duration: 0.50, ease: 'power2.out' },
            CTA_START + 0.20);
});

// Top/bottom film-credit meta strips
tl.fromTo(PREFIX + ' #cta-top-meta',
          { opacity: 0, y: 6 }, { opacity: 1, y: 0, duration: 0.50, ease: 'power2.out' }, CTA_START + 0.25);
tl.fromTo(PREFIX + ' #cta-bot-meta',
          { opacity: 0, y: -6 }, { opacity: 1, y: 0, duration: 0.50, ease: 'power2.out' }, CTA_START + 0.30);

// T-bar stroke draw
(function(){
  const sel = PREFIX + ' #cta-path-tbar';
  const el = document.querySelector(sel); if(!el) return;
  const L = el.getTotalLength();
  tl.set(sel, { strokeDasharray: L, strokeDashoffset: L }, CTA_START + 0.30);
  tl.to(sel, { strokeDashoffset: 0, duration: 0.25, ease: 'power2.inOut' }, CTA_START + 0.30);
})();

// "vTake" main name mask-reveal (left → right)
tl.fromTo(PREFIX + ' #cta-vtake-name',
          { clipPath: 'inset(0 100% 0 0)' },
          { clipPath: 'inset(0 0% 0 0)', duration: 0.45, ease: 'power2.inOut' },
          CTA_START + 0.40);

// T-stem stroke draw
(function(){
  const sel = PREFIX + ' #cta-path-tstem';
  const el = document.querySelector(sel); if(!el) return;
  const L = el.getTotalLength();
  tl.set(sel, { strokeDasharray: L, strokeDashoffset: L }, CTA_START + 0.50);
  tl.to(sel, { strokeDashoffset: 0, duration: 0.25, ease: 'power2.inOut' }, CTA_START + 0.50);
})();

// Dual concentric rings fade-in (CSS @keyframes handles the slow spin)
tl.fromTo(PREFIX + ' #cta-ring-outer',
          { opacity: 0 }, { opacity: 1, duration: 0.50, ease: 'power2.out' }, CTA_START + 0.55);
tl.fromTo(PREFIX + ' #cta-ring-inner',
          { opacity: 0 }, { opacity: 1, duration: 0.50, ease: 'power2.out' }, CTA_START + 0.60);

// Amber dot scale-pop (elastic)
tl.fromTo(PREFIX + ' #cta-dot',
          { opacity: 0, scale: 0.4, transformOrigin: '48px 118px' },
          { opacity: 1, scale: 1, duration: 0.25, ease: 'back.out(1.7)' },
          CTA_START + 0.65);

// Decorative divider segments grow-x
tl.fromTo(PREFIX + ' #cta-seg-l',
          { width: 0 }, { width: 56, duration: 0.30, ease: 'power2.out' }, CTA_START + 0.75);
tl.fromTo(PREFIX + ' #cta-seg-r',
          { width: 0 }, { width: 56, duration: 0.30, ease: 'power2.out' }, CTA_START + 0.75);

// Divider diamond pop
tl.fromTo(PREFIX + ' #cta-diamond',
          { rotate: 45, scale: 0 },
          { rotate: 45, scale: 1, duration: 0.30, ease: 'back.out(1.7)' },
          CTA_START + 0.85);

// Tagline fade-in
tl.fromTo(PREFIX + ' #cta-tagline',
          { opacity: 0, y: 4 }, { opacity: 1, y: 0, duration: 0.45, ease: 'power2.out' }, CTA_START + 0.90);

// Card exit — fade-out in the last 0.30s of the 2.0s window
tl.to('.card-host[data-card-id="card-cta-vtake"]',
      { opacity: 0, duration: 0.30, ease: 'power2.in' }, CTA_END - 0.30);
tl.set('.card-host[data-card-id="card-cta-vtake"]', { visibility: 'hidden' }, CTA_END);
```

### 10. Render to MP4

```bash
cd "$WORK_DIR"
PRODUCER_BROWSER_GPU_MODE=hardware npx hyperframes render public \
  -o output.mp4 \
  --fps 30
```

`hyperframes render <dir>` reads `<dir>/index.html` and produces the MP4.
The flag `PRODUCER_BROWSER_GPU_MODE=hardware` (or `--browser-gpu`) is
strongly recommended on macOS — software-only Chrome rendering times out
on most laptops.

For a sanity check before the full render, capture a single frame at a
specific timestamp:

```bash
npx hyperframes snapshot public --at 5 --out snapshot-5s.png
```

### 11. Report Results

Tell the user:

- Work directory path
- `storyboard.json` (the card outline you designed)
- `public/cards/*.html` (one HTML per card)
- `public/index.html` (the assembled composition)
- `output.mp4` (the final video)
- ASR provider used
- Card count + how you chose them (in 1 sentence)
- Any missing keys or quality caveats

Do not delete the work directory unless the user asks.
