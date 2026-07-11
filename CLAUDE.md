# testdrive — train-robbery

Git repo → `git@github-personal:vaibhavgit9210/train-robbery.git` (personal account — see `../CLAUDE.md` for the two-account rules).

- **Live site:** https://vaibhavgit9210.github.io/train-robbery/ — served from the `gh-pages` branch. After changing `index.html`, push BOTH: `git push && git push origin main:gh-pages` (force `main:gh-pages` if histories diverge).
- `index.html` and `train-robbery.html` are the same single-file WebGL animation (index is the copy Pages serves). Edit `index.html`, then sync: `cp index.html train-robbery.html`.
- `README.md` is a teaching guide (how the scene was built, for a smaller LLM) — keep it in sync with any architectural change to the animation.
- `board/` is a **different project** (splittable whiteboard, own CLAUDE.md) and is git-ignored here on purpose. Don't add it to this repo.
- Verify renders with the headless-Chrome screenshot command in `../CLAUDE.md` — never ship a visual change unseen.
