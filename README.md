# All-In Music Logger

A small web app for logging a year of music listening, 1-2 sentences per album, capturing album art, Bandcamp or other listening links, and references to reviews.

The plan for 2026: leverage this for my Nine Circles end-of-year post.  The plan is to show off every single album I listened to in the year, with some kind of badge that shows and/or jumps to my Top 9 ranked albums (the "inner circle" if you will) with every other album (the "outer circle") arranged alphabetically within one of several genre categories.  Currently at ~182 albums based on the last import). The logger is where the list gets written. The CSV is the master copy. The WordPress page is built from the CSV.

The page is hosted on Github Oages at: https://consumethetangible.github.io/music-all-in/

## How the pieces fit

| Piece | Where it lives | What it does |
|---|---|---|
| Logger app (`index.html`) | This public repo, served by GitHub Pages | Search artists, edit albums, save straight to the CSV. Works on desktop, tablet, and phone. |
| The list (CSV) | Private repo `music-all-in-database` | `Nine Circles – All-In 2026 - Albums.csv` on branch `main`. Every save is a commit, so there is free backup and undo. |
| WordPress page | ninecircles.co | `all-in-body.html` pasted into a Custom HTML block, styled by `all-in.css` in Additional CSS. |

This repo is public and holds no data. The private repo holds the list.

## Setup

1. **Public repo (this one):** put `index.html` at the root, then turn on Pages (Settings, Pages, deploy from the `main` branch, root folder). The app is served at `https://<username>.github.io/<repo>/`.
2. **Private repo:** `music-all-in-database`, containing `Nine Circles – All-In 2026 - Albums.csv` on `main`.
3. **Access token (once per device):** GitHub, Settings, Developer settings, Personal access tokens, Fine-grained tokens, Generate new token.
   - Resource owner: your account.
   - Repository access: **Only select repositories**, then `music-all-in-database`.
   - Permissions, Repository permissions: **Contents** set to **Read and write**. Leave the rest alone.
   - Set an expiry, generate, and copy the token right away. GitHub shows it once.
4. **Open the app and connect.** The repo name, file name, and branch are pre-filled. Enter your GitHub username and paste the token. To skip the username next time, put it in `DEFAULTS.owner` near the top of the script in `index.html`.
5. **On a phone:** open the same Pages address, connect once, then use Share, Add to Home Screen. The token is stored per browser, so each device needs it entered once.

The token is kept in that browser's local storage and can touch only the one private repo. **Menu, Forget this device** removes it.

## Using the logger

- **Search** matches artists and album titles, ignoring accents (typing "mol" finds MØL) and a leading "The" or "A". Press `/` to jump to the search box.
- **Filters:** All, **Ranked**, **New**, To write (artists with an album still missing a sentence), **Needs link** (artists with an album whose Listen link is not yet a normal web link), **Needs cover**, Top 9, Picks.
- **Ranked view:** every album can be ranked 1 to N. This view lists albums (not artists) in rank order, with a divider after #9. **The first nine are your Top 9 and label themselves.** Drag a row onto another to take its slot, or tap the rank number and type a new position. The ✕ removes an album from the ranking. Albums not yet ranked sit underneath under "Not ranked yet": tap **+ Rank** to add one at the end, or drag it into the exact slot you want. The search box works inside this view.
- **Artist view:** one card per album with a cover preview. The **Listen link** is right at the top with an **Open ↗** button (amber dashed while empty, with a note if it is a Bandcamp embed code instead of a link). Then the Sentence (with a word count), then Genre and Type. The **rank pill** shows "Rank it" or the album's position ("#3 · Top 9"); tap it to add or remove the album from the ranking. **Genre pick** is a tap toggle. "Ranking ›" jumps to the album in the Ranked view. Artist, Album, Year, Date added, Cover URL, and Write-up URL sit under **Details** (opens automatically for a new, untitled album).
- **Find cover:** the button under each cover searches Apple Music and shows candidates. Tap one to fill Cover URL. If it is not there, use the Search Bandcamp link, copy the image address, and paste it into Cover URL.
- **Next to write:** the button at the top and bottom of each artist jumps to the next artist (A to Z, wrapping) with an album still missing a sentence and puts the cursor in that sentence box. Ctrl or Cmd + Enter in the sentence box does the same.
- **Adding albums:** the **+ Add** button next to the search box (or press `n`) opens a dialog with two tabs.
  - **One album:** Artist (autocompletes and matches the spelling already in your list), Album, and Genre (remembers your last pick). **Add** keeps the form open for the next one; **Add and open** jumps into the card to write the sentence. It warns if the album is already in the list.
  - **Paste a list:** one `Artist - Album` per line (a dash, tab, or `|` works, and leading numbers or bullets are ignored). A preview shows what will be added and what is skipped as a duplicate or unreadable. One genre can be applied to all.
  - **Cover and Type** are filled from Apple Music, only when it finds the same artist and a matching title. Otherwise they stay blank (use Find cover). "Title - EP" and "Title - Single" set the Type. The Listen link is always left blank. Apple limits how fast it answers, so a pasted list is looked up one at a time.
  - You can also type `Artist - Album` in the search box and choose "Add ... by ...".
  - The **New** filter shows artists added this session or dated today. **Needs cover** shows albums with no cover link yet.
- **Other ways to add:** type a name that is not in the list and choose "Add as a new artist," or use "Add another album by..." on an artist. New rows get Year 2026 and today's date.
- **Progress chips** along the top show Written, Top 9, and Picks, and turn red when something is wrong (more than 9, or two picks in one genre).
- **Review sheet** (tap the Top 9 or Picks chip, or "See the list" on a card): the current Top 9 as a wall of covers with open slots, and the picks by genre. Each row has Open (jump to that album) and Clear (remove the flag and save). The Top 9 here is read-only and follows the ranking. Use "Edit the full ranking" to change it.
- **Saving:** about two seconds after you stop typing, and immediately when you switch away from the app. Commit messages name the artist, for example "Update Mastodon."
- **If two devices edit at once,** the app stops and asks whether to load the latest or overwrite. It never overwrites silently.
- **Offline:** the edit stays on screen with a Try again button.
- **Menu:** reload from GitHub, download a copy of the CSV, connection settings, forget this device.
- **Local file mode** (desktop Chrome or Edge only) is on the connect screen for trying the tool without GitHub.

## The CSV

The app reads columns by header name, keeps columns it does not know about, keeps their order, and keeps the original line endings. A no-change save rewrites the file byte for byte.

| Column | Notes |
|---|---|
| Artist, Album | Rows are grouped by exact artist name. |
| Year | Always 2026. |
| Genre | Dropdown. Current list: Black Metal, Death & Doom, Sludge, Stoner & Psych, Prog & Tech, Trad & Thrash, Hardcore & Punk. |
| Type | LP, EP, Split, Demo, Live, Archival, Reissue, Soundtrack, Single. |
| Top 9 | `Yes` or blank. Once you use ranking this is written for you: `Yes` for ranks 1 to 9, blank otherwise. |
| Rank | Optional. 1 to N across all ranked albums, blank means unranked. The column is added the first time you rank something (an older "Top 9 Rank" column is recognised too). Albums already marked Top 9 are used to seed the ranking. |
| Genre Pick | `Yes` or blank. One per genre. |
| Sentence | The one-sentence review. |
| Listen URL | See "Known issues." |
| Cover URL | Image link. Missing covers show an initials tile. |
| Write-up URL | Link to a full review. Blank means none. |
| Date Added | `yyyy-mm-dd`. |

**What appears on the published page:** a row appears once it has a Sentence, a Genre, and a Type of LP, EP, or blank (blank counts as LP). Split, Demo, Live, Archival, Reissue, Soundtrack, and Single rows stay in the CSV, but are not shown and not counted.

**To change the genres:** edit the `GENRES` list near the top of the script in `index.html`. Any genre that appears in the CSV but not in that list is added to the dropdown and the picks review automatically. The page styling and anchors (`black`, `doom`, `sludge`, `stoner`, `prog`, `trad`, `hardcore`) will need the same change.

**How the 182 rows got there:** imported on 2026-10-02 from the Apple Music playlist "The Year in METAL" (library XML export), collapsed to one row per album, with first-pass genres filled in for 106 albums.

## The WordPress page

Files: `all-in-body.html` and `all-in.css` (suggested location in this repo: `wordpress/`).

1. Paste `all-in.css` into **Additional CSS** (Site Editor, Styles, the three-dot menu). It is all scoped under `.nc`, so it will not touch the rest of the site. Replace the old copy rather than adding a second one.
2. In the post: write the intro as normal paragraphs, add a **Read more** break, then add a **Custom HTML** block containing `all-in-body.html`.
3. The block starts at the Genre Picks strip on purpose. There is no title, hero, or intro text in it, and no `<h1>`, because the post title already is one.

**Layout**

- **Genre Picks strip:** one cover tile per genre, a red Top 9 badge where the pick is also a Top 9, and click to jump to the section. Seven tiles across when wide, 4 + 3 in a narrow column, a swipe row on phones.
- **Jump bar:** sticky, one link per genre. Its offset below the theme's sticky nav is the `--stick` variable at the top of the CSS (28px by default, and +32px when the WordPress admin bar is showing). Set it to 0 if the theme nav is not sticky.
- **Genre sections:** A to Z by artist (ignoring "The" and "A"), two cards across when the column is wide and one when narrow. The layout responds to the width of its box, not the screen, using container queries.
- **Show all toggle:** the first 10 cards in a genre are visible, and the rest sit behind "Show all N." It is a plain `<details>` element, with no JavaScript.
- **Card:** cover, artist, Top 9 badge, album, sentence, Listen link.
- **Fonts and colors:** two variables, `--hf` (headings, Barlow Condensed) and `--bf` (body, Source Serif 4), to swap for theme fonts. The panel is warm black with a red accent. A light variant has not been built.

## Known issues and loose ends

- **Not yet seen in a real browser.** The app was built and tested without one. Its logic has automated tests (42 checks against a mocked GitHub), but the layout, especially on phones, has not been checked by eye. Send screenshots of anything off.
- **Listen URL holds Bandcamp shortcodes** (`[bandcamp ... album=123 ...]`). They work for embeds in a post, but cannot be turned into a plain link, so the test page links to Bandcamp's bare player page. A normal album URL in that column would fix it.
- **Covers are hotlinked** from Bandcamp's CDN. They work, but depend on Bandcamp. Uploading them to the media library is the longer-term plan.
- **The "Full write-up" link** is in the design, but not yet in the generated page.
- **The theme match is untried.** Expect to tune the sticky offset, the panel width (full width or inside the column), and the fonts after pasting into the draft.
- GitHub's contents API handles files up to 1 MB, which is far more than this list needs.

## What is next

1. **Build page button** in the logger: reads the same CSV, applies the display rules above, and copies the HTML for WordPress, including Write-up links and the 10-card toggle.
2. **Top 9 post:** the short companion post, reusing the same styling and linking to the All-In post.
3. **Match the theme** on the draft page.
4. **Revisit the genre list** (planned for last).
5. **Clean up** Listen URLs and covers.

## Decisions so far

- The CSV in the private repo is the single master copy. The earlier Google Sheet and the Apps Script generator idea are retired.
- One genre per album.
- Only LPs and EPs count. All other types are kept in the data but left off the page.
- A row appears on the page once it has a sentence.
- Genre Pick is a plain Yes or blank flag. Every album can be ranked, and the Top 9 is simply ranks 1 to 9. The Top 9 column is kept in sync as Yes or blank, so the page generator does not need to know about ranking.
