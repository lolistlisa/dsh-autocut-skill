# autocut-workflow — DSH skill

A DeepSeek Harness skill for the **AutoCut** video pipeline: script → segments → local/online assets → BGM → timeline → encrypted JianyingPro (剪映) draft.

This skill is the operating manual. The actual tools come from the [`dsh-autocut`](https://github.com/lolistlisa/dsh-autocut) plugin (20 `autocut_*` tools).

## Install

1. Install the plugin first: `dsh plugin --profile desktop add "github:lolistlisa/dsh-autocut"`, then fully restart DSH.
2. Install this skill by copying the folder into your workspace:

   ```text
   <workspace>/.dsh/skills/autocut-workflow/SKILL.md
   ```

   or into your user-level skills directory (e.g. `%USERPROFILE%\.dsh\skills\autocut-workflow\`).
3. Ask DSH to "按 AutoCut 工作流剪一条片", or simply mention `autocut_*`.

## What it enforces

- Script and recordings are **read-only**; no rewriting or merging.
- The user picks assets, music, and approves the preview; automation only shortlists.
- Three hard stops: selection (`review.html`), preview (`preview.mp4`), final Jianying draft.
- Existing Jianying drafts are never modified; only new drafts are created.
- No silent fallback: missing dependencies, plaintext drafts, or failed downloads stop the run.

## Requirements

Windows, JianyingPro 11.x for draft export, Python 3.10+ with `faster-whisper` and the `aoguai/pyJianYingDraft` fork, FFmpeg/ffprobe, and a `DEEPSEEK_API_KEY`. Pexels/Pixabay/Jamendo keys are optional.

See the plugin README for the full environment setup.

## License

MIT
