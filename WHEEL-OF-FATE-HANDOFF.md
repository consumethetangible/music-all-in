# Wheel of Fate: sync review and recommended approach

Written from a session in `consumethetangible/music-all-in`, where I reviewed `consumethetangible/the-wheel-of-fate` (read-only, from the code; nothing was run or reproduced). Hand this to the session working on the Wheel repo.

## Goal

1. **Fix the JSON sync**, which is "always buggy".
2. **Optionally**, later, support multiple wheels (separate listening groups or seasons), reusing the "lists" pattern built in `music-all-in`.

Do 1 first. It is the bigger win, and it should come before 2, because copying the lists feature on top of the current sync would copy the bugs.

## How the Wheel stores data today

- Single `index.html`, no build step, vanilla JS. Hosted on GitHub Pages.
- State = `wheel-state.json` in the same (public) repo: `{ round, nominees[6], chronicle[] }`. `artUrl` is session-only and always nulled.
- `localStorage` key `wheelOfFates_v3` is a fast local cache. GitHub is the source of truth.
- Read path: `fetchCloudState()` pulls `raw.githubusercontent.com/.../wheel-state.json?_=<now>`, then `applyCloudState()` **replaces** local state.
- Write path: `saveState()` (called on blur, seal, spin, etc.) writes localStorage then `saveCloudState(payload)`. That chains saves in a promise queue and, for each, does `GET contents/wheel-state.json` for the current `sha`, then `PUT` the **whole file**.
- Token: fine-grained PAT in `localStorage` (`wheelOfFates_ghToken`), pasted via `prompt()`.
- Hard-coded: owners `Baxter`/`Chris`/`Erik`, 6 seats (2 each). `tools/validate-wheel-state.js` checks the file structure offline.
- Seat model: each member enters their own two seats, usually from their own device.

## Findings (ranked)

1. **Whole-file, last-write-wins. Probably the main bug.**
   Each save does GET-latest-sha, then PUTs the entire local state. The sha is fetched *immediately before* the write, so it never detects that someone else changed the file since this device loaded it. Three members saving from three devices means whoever saves last wipes everyone else's seats. The 409 retry only covers a race of milliseconds and then overwrites anyway. The README's "manual weight adjustment… when syncing across devices" is the symptom.

2. **No re-sync after load.**
   There is no `visibilitychange`/focus refresh. A tab left open (a phone, say) stays stale indefinitely, and its next save overwrites newer cloud state.

3. **Failed saves are dropped, then clobbered.**
   `saveCloudState` ends in `.catch(() => {})`, and `_doSaveCloudState` catches network errors and shows "Offline". Nothing retries. On the next page load `applyCloudState` replaces local state with the older cloud copy, so the unsynced edit is lost.

4. **Load overwrites in-progress edits.**
   The initial `fetchCloudState().then(applyCloudState)` runs in the background. Anything typed before it resolves is replaced.

5. **A 403 deletes the token.**
   `_doSaveCloudState` removes the token on 401 *or* 403. GitHub also returns 403 for rate limits and secondary limits, so a transient limit forces re-pasting the token. A 422 (stale/missing sha) is not handled at all and just shows "Sync error".

6. **One commit and two API calls per blur.**
   No debounce or coalescing, so tabbing through fields creates a pile of identical commits ("Wheel state: Round XIX").

7. **Smaller:**
   - The read path (raw CDN) and write path (contents API) differ, so reads can lag a fresh write by a few seconds.
   - Cloud JSON is applied without validating its shape beyond `Array.isArray(nominees)`.
   - `wheel-of-fates-history.json` looks like a stale manual export.

## What already solves this in music-all-in

Reference: `https://github.com/consumethetangible/music-all-in`, `index.html`, inside the IIFE. Function names are the stable handles, not line numbers.

| Concept | Where | What it does |
|---|---|---|
| Version tracking | `useStore`, `store.load()` returns `{ text, version }` | Remembers the file's `sha` as it was loaded. |
| Conflict-safe save | `githubStore().save(text, ver, message, force)` | PUTs with the *loaded* sha. 409, or 422 with a sha message, throws `Conflict`. `force` re-reads the latest sha. |
| Conflict UX | `save()` catch branch | Banner offering "Load the latest (drop my edits)" or "Overwrite with what is on screen". |
| Debounced autosave | `markDirty()` | Sets `dirty`, then saves after `store.delay` (1800 ms). Edits coalesce into one commit. |
| Serialized saves | `saving` / `saveAgain` flags | One save in flight. Another edit during a save triggers one follow-up save, not a queue of stale payloads. |
| Re-sync on return | `visibilitychange` + `refreshRemote()` | On tab focus, if nothing is dirty and the remote sha changed, reload. If dirty, do not clobber. Saves on hide. |
| Online retry | `window.addEventListener('online', ...)` | Saves again when connectivity returns. |
| Leave guard | `beforeunload` | Warns if there are unsaved changes. |
| Error mapping | `problem(res)` | 401 = bad token (do **not** assume 403 means bad token), 403 = needs Contents RW, 404, other. |
| Lists (optional phase) | `listFiles`, `create`, `readSettings`/`writeSettings`, `switchList` | See Phase 2. |

Sketch of the core (condensed from `music-all-in`):

```js
async function load() {
  const res = await fetch(url + '?ref=' + branch, { headers: hdr(), cache: 'no-store' });
  if (!res.ok) throw await problem(res);
  const j = await res.json();
  return { text: b64ToText(j.content), version: j.sha };
}
async function save(text, loadedSha, message) {
  const res = await fetch(url, { method: 'PUT', headers: hdr({ 'Content-Type': 'application/json' }),
    body: JSON.stringify({ message, content: textToB64(text), sha: loadedSha, branch }) });
  if (res.status === 409) throw new Conflict();
  if (res.status === 422) { /* sha mismatch message -> Conflict, else problem(res) */ }
  if (!res.ok) throw await problem(res);
  return (await res.json()).content.sha;   // new version
}
```

The key difference from the Wheel: **save with the sha you loaded, not the sha you fetched a moment ago.** A mismatch means someone else wrote in between, and you must reconcile instead of overwriting.

## Recommended approach

### Phase 1: fix sync (do this first)

1. **Replace the read path** with the contents API (token required) so you get `{ content, sha }` in one call and no CDN lag. Keep `raw.githubusercontent` only as a read-only fallback when there is no token.
2. **Track `version` (sha)** per loaded state. Save with that sha. Update it from the PUT response.
3. **Debounce saves** (about 1.5 to 2 s) and keep one save in flight with a "save again" flag, as in `markDirty`/`save`.
4. **On a conflict, auto-merge by seat owner** instead of asking the user. Each member only edits their own two seats, so the merge is mostly deterministic:
   - Re-fetch the latest cloud state and its sha.
   - Start from the cloud state. For each nominee seat the *local user owns*, take the local value. For other seats, take the cloud value.
   - `round` and `chronicle`: take the higher `round`. Append chronicle entries that are in local but not cloud, keyed by `round`.
   - Retry the PUT with the new sha. Cap retries (about 3), then surface the banner (load latest or overwrite).
   - Needs a "who am I on this device" notion, or infer ownership from which seats were edited since load. The simplest version is a per-device owner setting in the UI. Ask the user before choosing.
5. **Refresh on return** (`visibilitychange`): if the tab is clean and the remote sha changed, apply it. If dirty, save/merge first.
6. **Stop swallowing failures.** Keep a `dirty` flag that survives failed saves. On `online` and on the next load, **retry before overwriting local**. If local is dirty and cloud differs, merge, do not replace.
7. **Don't apply cloud state over in-progress typing.** Apply the load result only if nothing is dirty. Otherwise merge.
8. **Token handling:** clear the token only on 401. On 403, show the GitHub message and keep the token.
9. **Run `tools/validate-wheel-state.js` logic in-app** (or a small shared validator) before PUT and after applying cloud state, so a bad blob never gets pushed or applied.
10. **Status line:** keep the existing "☁️ Synced / Offline / Not connected" element, and add states for "Unsaved changes…", "Merging…", and "Updated from cloud".

### Phase 2: multiple wheels (optional, after Phase 1 is solid)

Mirror the Nine Circles pattern: each wheel is its own `wheels/<name>.json` (or `<name>-state.json`) in the same repo. A small `settings` JSON beside them holds per-wheel config (member names, seats per member, display name). A switcher lists the files via the contents API directory listing, with "+ New wheel" creating a blank state file. Remember the last wheel per device. Replace the hard-coded `Baxter`/`Chris`/`Erik` and 6 seats with the settings. Keep `defaultNominees()` driven by those settings. Existing `wheel-state.json` becomes the default wheel with no migration (no settings entry means the current behavior).

### Out of scope / don't do

- Don't add a shared runtime dependency between the two repos. Copy and adapt. Both are single-file apps and diverge.
- Don't change the wheel's weighting, UI, or lore text as part of the sync work.

## Open questions to settle with the user before building

1. **Owner merge:** are Baxter, Chris, and Erik the only editors, each editing only their own seats? How should the device know who it is (a saved "I am…" setting)?
2. **Phase 2:** is a second wheel (another group or a season) actually wanted? If so, what does "a wheel" mean: another club, or a new season of the same club?
3. **Public repo:** `wheel-state.json` is public. Is that acceptable? (Presumably yes, since the raw read path is unauthenticated.)
4. **Token UX:** keep the `prompt()` paste flow, or add a small connect dialog like the one in `music-all-in`?

## Suggested test plan (before merging)

- Two browser contexts (or two devices) with the same state: edit different seats in each, confirm both edits survive.
- Offline edit, then reconnect: confirm it retries and is not overwritten on reload.
- Stale tab: leave one open, change from another, refocus the first, confirm it refreshes.
- Type during initial load: confirm edits are not wiped.
- Force a 403 (for example by using a rate-limited/invalid-scope token): confirm the token is kept and the message is shown.
- Mock the GitHub contents API (a Playwright `page.route` mock worked well in `music-all-in`) to exercise 409/422 paths without a real repo.

## Access note

The reviewing session had **read-only** access to `consumethetangible/the-wheel-of-fate`. A session that will implement this needs write access (attach it with push access) and should work on a branch with a PR.
