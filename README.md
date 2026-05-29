# vtake-skills

Agent skills for [VTake](https://vtake.app) — turn a local video into an
AI-composed, card-based recut.

Distributed for the open [`skills`](https://github.com/vercel-labs/skills)
CLI. Skills are convention-discovered under `skills/<name>/SKILL.md`.

## Skills

| name | what it does |
|---|---|
| **`vtake-cut`** | Local video → metadata + transcript → agent-designed HTML cards → rendered MP4. The agent writes each card's HTML in conversation, assembles a GSAP-timed composition, and renders via `hyperframes`. |

## Install

```bash
# global (available across all projects → ~/.claude/skills/vtake-cut/)
npx -y skills add notedit/vtake-skills --skill vtake-cut --yes --global

# project-local
npx -y skills add notedit/vtake-skills --skill vtake-cut --yes

# list everything in this repo first
npx skills add notedit/vtake-skills --list
```

Then invoke it from your agent (e.g. `/vtake-cut <video.mp4>` in Claude Code).

## Runtime dependencies

The skill is self-contained except for these, which it resolves on demand:

- **`@notedit/vtake` CLI** — `extract` / `transcribe` / `doctor`, invoked via
  `npx -y @notedit/vtake@latest …` (published on npm; first call downloads it).
- **`hyperframes` CLI** — rendering, invoked via `npx hyperframes render`.
- **system `ffmpeg` / `ffprobe`** — required for audio/metadata extraction.
- macOS render: `export PRODUCER_BROWSER_GPU_MODE=hardware` strongly
  recommended.

Optional `ELEVEN_API_KEY` enables direct ElevenLabs ASR; without it the CLI
falls back to the rate-limited `https://vtake.app/api/transcribe` proxy
(3 req/min/IP).

## Bundled assets

Shipped inside `skills/vtake-cut/assets/` so the skill works standalone:

- **Fonts** (`assets/fonts/*.woff2`): Inter, Caveat, Virgil, and
  [LXGW WenKai TC](https://github.com/lxgw/LxgwWenkaiTC) — all under the
  SIL Open Font License (OFL).
- **`assets/vendor/gsap.min.js`**: [GSAP](https://gsap.com) standard build
  (free, redistributable).

The visual design library lives at `skills/vtake-cut/references/`
(10 styles × 4 layouts × 3 frames + `DESIGN_INDEX.md`).

## License

Skill content: see [LICENSE](LICENSE). Bundled fonts/GSAP retain their
respective upstream licenses noted above.
