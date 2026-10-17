<!-- BEGIN:nextjs-agent-rules -->
# This is NOT the Next.js you know

This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` before writing any code. Heed deprecation notices.
<!-- END:nextjs-agent-rules -->

# Film Design System (`film-design-system`)

Local-first app that manages a film's visual world like a design system and generates
imagery against it. Entities (characters, props, settings) own curated **reference images**;
you compose a shot, the app allocates references within the model's caps, calls the
provider, and records complete lineage. Good outputs get **promoted** back into the
reference set — that's what keeps a character consistent across hundreds of generations.

Read [README.md](./README.md) for the full loop; it's accurate and current.

## Commands

```bash
npm run dev
npm run build
npm run lint          # eslint
npm test              # vitest run — 83 tests, all offline
npm run db:push       # drizzle-kit push
npm run seed          # tsx scripts/seed.ts
```

The live check **costs one real generation** and is opt-in:

```bash
SMOKE_GEMINI=1 npx vitest run tests/smoke-gemini.test.ts
```

**Never run the smoke test without Nick's explicit go.** Everything else is free and offline.

`.claude/launch.json` is configured — prefer `preview_start` for the dev server. The app is
reachable from other devices on the tailnet at `http://100.75.213.56:3000` (Mac Studio).

## Architecture

```
library/          images + library.db — THE ACTUAL FILM ARCHIVE, git-ignored
src/lib/          models, select-refs, prompt, output-spec, store, generate, providers/
src/db/           drizzle schema + client
src/app/          screens, server actions, API routes
docs/superpowers/ design spec, multi-provider addendum, implementation plan
```

- **Images are written to disk exactly once.** References are DB rows pointing at them, so
  promoting a result never copies a file and there is no sync drift. Don't "fix" promotion
  by copying.
- **`selectRefs` is strategy-aware and must stay that way.** The two providers have opposite
  reference philosophies and the app encodes the difference rather than averaging it:
  Gemini has hard per-category caps and you **fill** them; OpenAI has one shared pool of 16
  and you **under-fill** (3–5 curated refs beat 16, because references compete for
  influence). A "unified" allocation strategy would silently degrade both.
- Every generation records its exact final prompt, the references actually sent, the model,
  and the style-token revision in force. Preserve that — it's how drift gets diagnosed.

## Gotchas

- **`gpt-image-2` is registered but shows as unavailable.** It needs `OPENAI_API_KEY` plus an
  adapter at `src/lib/providers/openai.ts` implementing the existing `ImageProvider`
  interface. The registry, pooled allocation, and pixel-size mapping are already built and
  tested — that adapter is the only missing piece. Google Gemini works and is verified live.
- **HEIC/HEIF images cannot be imported.** `sharp` here has no HEVC decoder and iPhone photos
  are HEIC by default. The dropzone rejects them upfront on purpose — don't "fix" it by
  accepting the file and failing deep. Export as JPEG first.
- Warnings appear when a character is squeezed below 3 references; that's where consistency
  degrades. Treat it as a real signal, not noise.
- **The local directory name has a trailing space** — `~/ClaudeCode Projects/Image Generation Pipeline `.
  Quote paths. The package is `film-design-system` and the remote is
  `Image-Generation-Pipeline-`; all three names differ.
- Never push to `main` — branch per task, PR.

<!-- BEGIN:generation-family -->
## Sibling repos

This repo is one of six related repos for AI-generated film work: images, 3D models, 3D
previs, video, audio, and continuity repair. They stay separate on purpose (different stacks,
no shared code). The same block sits in all six AGENTS.md files between the
`generation-family` markers; change it in all six or in none.

| Repo | Local folder (always quote) | What it does | Stack |
|---|---|---|---|
| Film Design System | `~/ClaudeCode Projects/Image Generation Pipeline ` (trailing space) | Canon of characters, props and settings with curated reference images; image generation against it | Next.js, SQLite, Gemini |
| 3D Models | `~/ClaudeCode Projects/3D Models` | Text-to-3D through Meshy, a three.js viewer and gallery, Blender and procedural models | Next.js, react-three-fiber, Meshy API |
| Scene Builder 3D | `~/ClaudeCode Projects/scene-builder-3d` | Shot-first 3D previs: build a set, place characters, frame shots with a camera, play the scene back | Vite, React, three.js, browser-only |
| Video Generation Pipeline | `~/ClaudeCode Projects/Video Generation Pileline` ("Pileline" is the real folder name) | Seedance and Runway clients; reference packs in `library/` | Python CLI |
| Audio Generation Pipeline | `~/ClaudeCode Projects/Audio-Generation-Pipeline` | Speech (Qwen3-TTS, Higgs Audio v2), voice cloning, sound effects, on the Mac Studio and the Art PC | Python, shell, PowerShell |
| Continuity Lab | `~/ClaudeCode Projects/Video Lab` | Find and repair inconsistencies between generated video shots | Next.js, SQLite, ffmpeg |

GitHub, all under `weeeha/`: `Image-Generation-Pipeline-` (trailing dash), `scene-builder-3d`,
`Video-Generation-Pipeline` and `Audio-Generation-Pipeline` are public; `3d-models-playground`
(the 3D Models repo) and `Video-Lab` are private.

### How they connect

No code or data path links them yet; files move by hand. Known, planned and wanted links:

- **Image → video (planned).** The Film Design System spec names Seedance reference-to-video
  as a later consumer of its canon. The video repo's `library/<category>/<name>/` packs
  (`refs/` + `card.md`) hold the same kind of thing as folders, so an export from the canon
  would land there.
- **3D Models → Scene Builder 3D and video (wanted).** 3D Models generates GLB files through
  Meshy, and Scene Builder 3D stores props as GLB. Nick wants generated models usable as
  previs props and as Seedance references; no step connects them yet. 3D Models also holds
  3D-printing work (toys, sculpture), which is unrelated to film.
- **Scene Builder 3D → video (planned).** Its roadmap milestone M9 exports a shot as a
  provider-neutral zip (grey guide clip, first frame, camera path, prompt) for manual upload,
  with Seedance as the first target. Its idle prototype, `weeeha/Scene-Builder-v2` (folder
  `Scene Builder V2 (3d editor)`), already packaged shots for Seedance.
- **Video → Continuity Lab.** The Lab imports existing clips into a project folder. Its own
  generation backend (`seedance-fal`, via fal.ai) is planned for its plan 5 and is separate
  from the BytePlus client in the video repo.
- **Audio → video (open).** Seedance 2.5 accepts audio references and the audio repo makes
  voices and effects, but no step connects them. The audio repo also serves OpenClaw's
  Speaker agent, which is unrelated to film work.

### Working across repos

- Read the sibling's own AGENTS.md (or README) before changing it. Spend rules differ: image,
  3D and video generations cost money and need Nick's go; audio runs on the Art PC GPU and
  can pause llama-server while it runs.
- Don't copy code between repos. When a shared format appears (a canon export, say),
  document it in both repos it connects and update this block.
- Branch per task in each repo; never push to `main`.
<!-- END:generation-family -->
