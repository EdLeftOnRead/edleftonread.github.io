# EdwinFIT — notes for future Claude sessions

This is Edwin's personal just-for-fun website: a Windows XP desktop simulation
with a night-shift schedule, and a few "programs" you can open from desktop
icons (Contact notepad, Watermelon video, Comments guestbook, Blog, etc).

## No build step

Everything lives in one file: `index.html`. Plain HTML/CSS/JS, no bundler,
no npm, no framework. Edit it directly. `WindowsXP.jpg` is the wallpaper,
`videos/watermelon.mp4` is the video player's source.

This local folder is not a git repo — see **Deployment** below for how
changes actually reach the live site.

## The big thing to understand: desktop and mobile are TWO DIFFERENT UIs

This is the most important thing to know before touching anything. The site
does **not** just reflow the desktop layout for small screens — mobile gets a
deliberately different interaction model, switched at the `max-width: 640px`
media query. Every window/"app" has to support both, and they behave
differently on purpose:

| | Desktop (>640px) | Mobile (≤640px) |
|---|---|---|
| Icons | Fixed column, top-left of desktop | Grid, fills the screen (home screen) |
| Opening an app | Double-click the icon | Single tap |
| How many windows open at once | Several, freely | **Exactly one at a time** — opening a new one force-closes whatever was open |
| Window chrome | Draggable, has _ (minimize) / □ (maximize) / ✕ (close) | Static, fullscreen, has a big "←" back button instead |
| Returning to icons | Icons are always visible underneath/behind windows | Icons are hidden while a window is open; back button (or closing the window some other way) reveals them again |
| Taskbar | Shows an app button per open window, click to switch/minimize | App buttons are hidden entirely (`.taskbar-app-btn { display: none !important }`) — navigation is icon grid + back button only |

**The user explicitly likes this and wants it kept this way.** Don't "simplify"
mobile back toward the desktop model (e.g. don't make multiple mobile windows
stack, don't bring back min/max buttons on mobile) unless asked.

### Why this matters when you add or edit a window

If you add a new "app" (new window-shell) and only wire it up for desktop,
it will render broken on mobile: overlapping content in one long scroll with
no way to navigate back, no icon grid, etc. If you only wire it for mobile,
desktop dragging/multi-window will misbehave. **Always update both.**

## Anatomy of one "app" / window

Look at the Contact notepad (`contactWindow`) or Comments wall
(`commentsWindow`) as reference implementations — they're the most recently
added and cleanest examples. Each app needs all of these pieces:

1. **Desktop icon** — a `.desktop-icon` button inside `.desktop-icons`, with
   an emoji and a `data-i18n` label.
2. **Window shell** — a `.window-shell` div (give it a unique modifier class
   like `.contact-window` for its desktop floating position: `top`/`left`/
   `transform` offset, chosen so it doesn't land exactly on top of other
   windows when several are open on desktop).
   - Titlebar: `.win-titlebar` → `.win-titlebar-left` (back button + title)
     and `.win-btns` (_, □, ✕). The back button (`.win-back-btn`) is
     `display:none` by default and only shown inside the mobile media query —
     don't remove it or the desktop titlebar breaks.
   - Optional `.win-menubar` with a "Plik"/File → "Zamknij"/Close item.
   - Body content.
3. **Taskbar button** — `.taskbar-app-btn`, starts `style="display:none;"`
   (shown once the window is opened for the first time). Hidden entirely on
   mobile via CSS.
4. **Start menu entry** — a button in `#startMenu`.
5. **i18n strings** — every visible label needs an entry in **both** the `pl`
   and `en` blocks of the `translations` object (search for
   `icon_contact_label` etc. as a template). The site defaults to Polish.
6. **JS wiring**, all near the bottom of the `<script>` block:
   - `const xWin = document.getElementById('xWindow')`
   - `const xCtrl = initWindow(xWin, { titlebar, taskbarBtn, btnMinimize, btnMaximize, onClose? })`
   - An `openX()` function that starts with:
     ```js
     if (!isDesktopMode()) { showExclusive(xCtrl, [/* all other ctrls */]); return; }
     ```
     followed by the normal desktop open/restore/wiggle logic.
   - Wire the desktop icon via `handleIconClick('iconX', openX)`.
   - Wire the back button: `document.getElementById('xBackBtn').addEventListener('click', goHome)`.
   - If closing the window needs side effects (e.g. pausing a video), pass
     `onClose` to `initWindow` — this fires on `.close()` from any path
     (back button via `showExclusive`'s exclusivity-close, taskbar close,
     Plik→Zamknij, etc), so you don't need to special-case mobile for cleanup.

## Key JS functions (all in the single inline `<script>`)

- `isDesktopMode()` — `window.innerWidth > 640`. The single source of truth
  for which UI mode is active. Never hardcode the breakpoint elsewhere.
- `initWindow(el, opts)` — returns `{ minimize, restore, toggleMinimize, wiggle, open, close }`
  for one window. Handles dragging, z-index, and (importantly) calls
  `updateMobileHome()` internally on every state change, so mobile
  home-screen visibility self-corrects no matter which code path closed/opened
  a window.
- `showExclusive(ctrl, others)` — the mobile "open this one, close everything
  else" primitive. Uses `.close()` (not `.minimize()`) on the others so
  `onClose` side effects (like pausing video) actually fire.
- `goHome()` — mobile back-button target. Minimizes every window and pauses
  the video.
- `updateMobileHome()` — toggles `body.mobile-app-open`, which is what the
  mobile CSS uses to decide "show icon grid" vs "show whatever window is
  open". Recomputed reactively; don't need to call it manually except in new
  one-off code paths that bypass `initWindow`'s functions entirely.
- `enforceMobileState(forceHomeIfDefault)` — runs at boot and on every
  `resize`. Handles two edge cases: (a) landing on mobile for the first time
  should show the icon grid, not the schedule window that's open by default
  on desktop; (b) resizing a desktop window down across the 640px breakpoint
  with multiple windows open should collapse to just one (keeps the
  highest z-index / most recently focused one, closes the rest).

### A timing gotcha worth knowing if mobile behavior seems flaky in testing

In at least one browser-automation/preview tool, emulated viewport resizing
can take a while to actually be reflected in `window.innerWidth` inside the
page's own JS — long enough that a plain "check once at script start" missed
it. That's why `enforceMobileState` is also wired to the `resize` event
rather than trusting a single synchronous check. If you ever see the mobile
layout not kick in when testing with a resized/emulated viewport, re-check
`window.innerWidth` from a fresh `javascript_exec` call before assuming the
CSS/JS is broken — it may just be that the automation tool's viewport
override hasn't settled yet. Real phones don't have this problem (their
width is correct from the very first script line).

## i18n system

Everything text-visible uses `data-i18n="key"` and gets swept by
`setLanguage(lang)`, which pulls from `translations.pl` / `translations.en`.
Language preference persists via `localStorage` (`grafik_lang`). When adding
any new user-facing string, add it to both language blocks — don't leave one
language with stale/English fallback text.

## Everything new must be translatable

This is a hard requirement, not a nice-to-have: **any text a visitor can see
must switch correctly when the language flag (or menuLang) is toggled.** If
you add a new app/window, dialog, button, tooltip, status message, or any
other on-screen string, it needs a `data-i18n="key"` attribute (or, for
JS-generated text, a lookup like `translations[currentLang].key`) plus a real
entry for that key in **both** the `pl` and `en` blocks — never just one, and
never a hardcoded string sitting outside the `translations` object. Before
considering any new feature done, switch languages with the flag buttons and
confirm nothing you added is still showing the other language (or is blank).
This applies even to small, seemingly cosmetic additions — e.g. the browser
tab title (`browser_tab_title`) is its own key, separate from the in-app
`window_title`, specifically so each piece of text can be translated and
worded independently.

## Comments guestbook backend (Firebase)

The comments wall (`commentsWindow`) is the one part of the site with a
real backend — Firestore, via the Firebase JS SDK loaded as an ES module at
the bottom of the file. Config lives in the `firebaseConfig` object right
there; if it still has `"PASTE_..."` placeholder values, the UI shows a
"not connected yet" message and disables posting instead of erroring —
keep that graceful-degradation behavior if you touch this code. Firestore
security rules (documented in the HTML comment right above the script) allow
public read + size-limited create, and explicitly deny update/delete — so
comment moderation/deletion is a manual Firebase console action, not
something the page can do. Don't try to add a client-side delete button
without first re-confirming with the user whether they want real
authentication added (this was explicitly discussed and declined in favor of
"delete via Firebase console" for now).

## Blog

The "Blog" window (`blogWindow`) shows a scrapbook-style feed — date, free
text, and inline photos/videos — newest first. Unlike the guestbook, this is
**read-only from the page**: there is no submit form and there shouldn't be
one. That's deliberate — Edwin is the only one who should be able to post,
and the site has no login system — so publishing goes through the data file
instead of a live UI:

- Posts live in the `blogPosts` array (top of the `<script>` block, right
  after `scheduleData`). Each entry looks like:
  ```js
  { date: 'YYYY-MM-DD', text: '...', media: [{ type: 'image'|'video', src: 'blog/...' }] }
  ```
  `renderBlogPosts()` sorts by `date` descending and re-runs on every
  `setLanguage()` call (so the date format follows the current language) —
  no need to call it manually after editing the array.
- Media files live under a `blog/` folder — a dated subfolder per post is a
  reasonable convention, e.g. `blog/2026-09-01/photo1.jpg` — referenced by
  that relative path, same as `videos/watermelon.mp4` already is.
- `text` is shown exactly as written and is **not** run through the i18n
  system, same as guestbook comments: it's Edwin's own content in whatever
  language he wrote it, not site chrome. Only the window around it (title,
  the "no posts yet" empty state) is translated.
- To publish: drop the media file(s) into `blog/`, add one entry to
  `blogPosts`, then deploy like the rest of the site (see **Deployment**).

## Deployment

The live site is `https://github.com/EdLeftOnRead/edleftonread.github.io`
(a public GitHub Pages repo). This local folder is not a clone of it —
deploys so far have gone through GitHub's web "Add files via upload" flow
rather than `git push` from this machine. That's workable for a single
`index.html`, but gets tedious once the blog has recurring posts with several
media files per entry each time — worth setting this folder up as a real git
repo pointed at that remote if Edwin wants a one-command `git push` publish
flow instead. Don't push to that remote on your own initiative — it's a live
public site, so confirm with Edwin first.

## Visual style

XP look via CSS custom properties (`--sys-gray`, `--sys-blue`, etc, defined
on `:root`). Body font is Tahoma/Verdana/Arial; the big pixel-font site title
uses `VT323` from Google Fonts. Keep new UI consistent with the existing
"beveled" button/panel border style (light top-left, dark bottom-right — see
`.win-btn`, `.dialog-actions button` for the pattern) rather than introducing
flat/modern styling.
