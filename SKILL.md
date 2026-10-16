---
name: keimou-doga
description: Create awareness videos from research to structured files.
version: 1
author: hermes-agent
license: MIT
metadata:
  hermes:
    tags: [creative, educational, video, awareness, 啓蒙動画, scene-planning]
    related_skills: [health-video-production, creative-content-generation]
---

# 啓蒙動画作成 (Educational Video Production)

## When to Use

- User asks to create an educational or awareness video on any topic (health, science, social issues, etc.)
- User needs research → scenario → scene breakdown → AI image prompts → output files workflow
- User requests structured summary files (abstract, detail) for a topic
- User needs video tool output (YAML, CSV) from a completed scenario

Procedural memory for creating educational/awareness videos: research-driven content with a 5-chapter structure, scene-level breakdown, and multi-format output.

## Procedure

### Step 0 — Information gathering

1. Run 3–4 parallel `web_search` calls with different query angles (e.g., causes/types, prevention/risks, early symptoms/recognition, latest statistics/research).
2. Extract key facts from top results via `web_extract` on the most relevant URLs.
3. Use these facts to ground the scenario content — cite statistics, include official sources, and structure chapters around verified information.
4. Append a reference section at the end of the final markdown file with title + URL of every source used.

### Step 1 — Plan structure (5 chapters)

For educational/awareness content:

1. **Hook/Opening** (2–3 scenes): Vivid metaphor that creates emotional impact and curiosity.
2. **Chapter 1 — Reality/Definition** (3–5 scenes): What is it? Statistics. Why it matters now.
3. **Chapter 2 — Core Mechanisms/Types** (4–5 scenes): Detailed breakdown of categories or underlying causes.
4. **Chapter 3 — Recognition/Warning Signs** (3–5 scenes): How to identify, difference from normal.
5. **Chapter 4 — Action/Prevention** (3–5 scenes): Practical steps, lifestyle, resources.
6. **Chapter 5 — Hope/Future** (3–4 scenes): New perspectives, treatments, community, call to action.
7. **Summary + Ending** (2 scenes): Recap key takeaways, emotional closure.

Total: 20–30 scenes. Adjust per complexity.

### Step 2 — Write scenario in markdown

Each scene includes:
- Scene number and title
- Chapter assignment
- Narration text (ナレーション)
- Image prompt (画像生成プロンプト)
- BGM suggestion

Chapter 5 ends with recovery/hope and the final takeaways; keep the closing thanks and subscription message exclusively in the template's separate fixed `E-1` scene, so the ending is not duplicated.

### Step 3 — Write summary files

- **abstract.md** (~400 characters): High-level overview of the topic and key takeaways.
- **detail.md** (~2000 characters): Detailed chapter-by-chapter breakdown with facts and statistics.

### Step 4 — Write scene image prompt CSV

File: `*_scenes.csv`
Columns: `scene_num,scene_name,seed,file_name,positive_prompt`
- `scene_num` format: `<chapter>-<scene>` (e.g., 1-1, 3-2)
- `seed`: increment by ~100 per scene for diversity while maintaining consistency
- `file_name`: `<prefix>_<scene_num>_<desc>.ext` (kebab-case)
- **Image prompt rules**: no text/captions, no proper nouns (studio names, character names), no anime/artistic style references. Use `photorealistic style` as base modifier.

### Step 5 — Write video tool YAML

File: `scenes.yaml` (same directory).

1. Load `templates/scenes.yaml` with `skill_view(name='keimou-doga', file_path='templates/scenes.yaml')` before generating YAML. Always use this user-specified full-document template, never a scene-only list.
2. Replace only the line `  # (ここにシーンを追加します)` with the generated scene entries, in scenario order. The marker is a YAML comment so the stored template itself remains valid YAML; do not leave the marker in the final output.
3. Preserve all template settings and the complete final ending scene `E-1` unchanged, including `image: ending.jpg` and its narration. Insert generated scenes BEFORE `E-1`; do not generate another ending or duplicate the ID `E-1`.
4. Write the full document with `write_file`, retaining the top-level mappings `video`, `subtitle`, `scenes`, and `options`. Quote scene titles and image paths as necessary for YAML safety; preserve narration line breaks using `voiceover: |`.

Generated scene format (indent exactly like the existing ending scene):
```yaml
  - id: {chapter}-{scene}
    image: output/{file_name from CSV}
    teletext:
      text: {scene title}
      position: top_left
      style:
        fill: "#FFE066"
        stroke: "#222222"
    voiceover: |
      {narration text}
    transition: cut
    voice_override:
      voice: narrator_male
      speed: 1.0
```

**YAML indentation requirements:**
- Top-level `video:`, `subtitle:`, `scenes:`, `options:`: 0 spaces.
- Scene list entries `  - id:`: 2 spaces; scene keys such as `image`, `teletext`, `voiceover`, `transition`, `voice_override`: 4 spaces.
- `teletext.text`, `teletext.position`, `teletext.style`, `voice_override.voice`, `voice_override.speed`, and narration lines under `voiceover: |`: 6 spaces.
- `teletext.style.fill` and `teletext.style.stroke`: 8 spaces.
- Use spaces only, never tabs. Do not strip indentation from narration lines.

**Verification:** Parse the generated document with an available YAML parser (for example PyYAML), not merely a visual check. Best practice: parse the stored template file itself and assert the generated doc's fixed parts equal it (the template nests `subtitle:` under `video:`, so its parsed root keys are video/scenes/options — do NOT assume four root mappings); confirm all fixed settings match the template; `scenes` is a list; every generated scene matches CSV order/IDs/image paths; the scene count equals generated scenes plus one fixed ending; IDs are unique; and the final scene is exactly the template's `E-1`. Confirm the insertion marker is absent. If a parser is unavailable, obtain one or explicitly report that syntax validation is blocked; do not claim the YAML was validated.

### Step 6 — Output convention

- All scene content aligned by scene_num across CSV and YAML files.
- Image prompts: visual metaphor / symbolic representation over literal depiction.
- Narration: conversational Japanese, second-person direct address (あなた).
- References section at end of markdown file: title + URL only, no annotations.

## Pitfalls

- **Single-source generation** — hold all scene fields (id/title/narration/prompt/bgm/seed/filename) in one Python data structure in the output folder and emit markdown, CSV, and YAML from it; guarantees cross-file alignment and lets fixes be regenerated everywhere at once. Keep the generator (`_gen_*.py`) and a `_validate.py` in the project folder for later iteration.
- **Language contamination** — long drafts can leak wrong-language punctuation/characters (e.g. Chinese ， or simplified glyphs) into Japanese narration. Scan artifacts for them in the validator, not just by eye.
- **write_file stale_write_blocked** — obtain a fresh complete read of an existing file, merge the intended changes, and retry `write_file`; do not bypass the protection with shell overwrites.
- **CSV line number leakage** — `read_file` display includes `LINE_NUM|` prefixes. Keep display prefixes out of artifact content; use structured CSV generation and `write_file`. Compare YAML block narration with CSV using `.rstrip('\n')`, because YAML `|` retains a trailing newline.
- **Related-skill YAML conflicts** — when combining this workflow with a health-video skill, this skill's user-specified full-document template takes precedence over any older scene-only YAML examples. Keep the fixed `E-1` once and include it in CSV alignment checks.
- **CSV consumer compatibility** — use UTF-8 with BOM for CSV intended for Windows spreadsheet applications, and read it with `utf-8-sig` during verification. Keep IDs, titles, paths, narration, and order aligned across every row.
- **YAML parser availability** — check the actual execution interpreter for PyYAML or another YAML parser; availability differs between environments. Preserve the template's formatting when writing and parse the result for verification.
- **Separate ending scene** — keep the template's fixed final scene `E-1` exactly once, after all inserted scenes; do not add a second closing scene.
- **Image prompt rules**: EXCLUDE text, letters, numbers, signs, logos from visual descriptions. Never include artist names (Studio Ghibli, Makoto Shinkai, Disney). Use descriptive style terms instead.

## Output Conventions

- All scene content aligned by scene_num across CSV and YAML files
- Image prompts: visual metaphor / symbolic representation over literal depiction
- Narration: conversational Japanese, second-person direct address (あなた)
- References section at end of markdown file: title + URL only, no annotations
