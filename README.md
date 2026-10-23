# xiaohongshu-post

An agent skill for publishing image-and-text notes to **Xiaohongshu (小红书 / RED)** from a
logged-in Chrome profile, via Playwright over CDP.

The hard part of this task is not clicking "发布". It is:

1. **Keeping the login alive across agent turn boundaries** — a browser process gets reaped when
   the turn ends, no matter how it was started (`playwright-cli open --headed`, a self-launched
   CDP Chrome, or a background task). **What persists is the profile directory, not the process**:
   cookies (including `id_token`) live in `$PROFILE_DIR/Default/Cookies`. The fix is to
   re-launch Chrome against the *same* profile every turn; it comes back logged in. Never
   create a fresh profile per turn — that means a fresh SMS login every turn.
2. **Getting a link that actually works for others** — Xiaohongshu note pages are client-rendered,
   so `curl` can never confirm public visibility; the only reliable check is a fresh logged-out
   browser context. And bare note URLs 404 without an `xsec_token`.

This skill encodes those two, plus the fiddly parts: multi-image upload, title/body limits, and
inserting **real** topic tags through the Tiptap suggestion dropdown (plain `#text` is not a topic).

## What it gives you

- A browser bootstrap (`scripts/bootstrap_browser.sh`) that survives turn boundaries.
- `scripts/publish_note.js` — fill title/body/topics, upload images, screenshot the draft.
  **It does not click 发布 unless you pass `--publish`.**
- `scripts/verify_note.js` — extract note id / `xsec_token` / short link, then prove public
  visibility from a logged-out context.
- `references/selectors.md` — every selector, API field, and known pitfall (with causes).
- `references/copy-guide.md` — title/body/image specs and a copy template.

## When to activate (triggers)

- "发小红书" / "上传小红书" / "帮我发个笔记" / "post this to RED / Xiaohongshu"
- Automating a note publish where the account is already logged in.
- Recovering a public share link for a published note.

Do NOT use for: video notes, live, comment automation, multi-account matrix publishing.

## Run it

```bash
# 1) start Chrome with a CDP port (Windows Git Bash)
#    ⚠️ reuse the SAME profile dir every time — that is what carries the login
bash scripts/bootstrap_browser.sh .tmp/xhs-cdp 9222 "https://creator.rednote.com/publish/publish"

# 2) if `loginWall` is true, the user logs in **in that window** (SMS only — no QR on the
#    creator platform). Once logged in, the profile keeps it for every later run.

# 3) fill the draft (does NOT publish)
node scripts/publish_note.js \
  --title "1024 捕虫：点掉虫子合成翻倍" \
  --body-file body.txt \
  --images cover.png,shot1.png,shot2.png \
  --topics "WorkBuddy,1024程序员节" \
  --out draft_top.png

# 4) show the draft to the user, wait for an explicit go

# 5) publish, then verify public visibility
node scripts/publish_note.js ... --publish
node scripts/verify_note.js --expect "1024 捕虫"
```

The browser process may be reaped between turns — that is expected and harmless. Just re-run
step 1 with the same profile dir; the session is still logged in.

Exit codes: `0` ok, `1` error, `2` bad input, `3` published but NOT publicly visible.

## Hard rules this skill enforces

- Never ask the user to paste their phone number or SMS code into the chat — let them type it
  into the real browser window.
- Never click 发布 before the user has seen the draft and said so.
- Never report "published" as success until a logged-out context can load the note.

## Install

```bash
npx skills add CatsMansion/xiaohongshu-post
```

Repo: https://github.com/CatsMansion/xiaohongshu-post

## License

MIT. See [LICENSE](LICENSE).
