# Notes for Claude Code sessions

- This repo is a fork of kaiyoti/open-dungeon-raid. Read `DEVLOG.md` first: it has the current
  status, what has changed and why, and the build, test and profiling recipes.
- Add a dated entry to the top of `DEVLOG.md` for every meaningful change (what, why, how it
  was verified, what is still open), and keep its status table current.
- Web-game work is almost always in `rebuild/web/template.html`. `rebuild/web/dr.js` (the wasm)
  is committed, so template-only changes don't need Emscripten (see DEVLOG, *Building without
  Emscripten*).
- Before pushing, run `for t in rebuild/web/test_*.js; do node "$t" || exit 1; done` after
  rebuilding the pages.
- Code comments cite the original binary (`Function @0xaddr`) and files like `tasks/…` or
  `AGENTS.md` that are not in this public tree. They are citations, not missing files.
