# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used Claude Code (Opus 4.8) as a pair-programming assistant throughout, but drove the decisions myself and verified every claim it made.

- **Reconnaissance:** had it map the codebase (models, services, routes, tests) and diff the feature branch against `main` so I understood the UUID refactor and the exact shape of the conflict before touching anything.
- **Mechanical edits:** the rename (Comment 1) and its call-site search, the deduplication logic patterned on `add_to_collection` (Comment 2), and the test file scaffolding (Comment 3). I checked each against the existing collection code rather than accepting them blind.
- **Verification, not just generation:** every change was confirmed by running `pytest tests/` and, for the watchlist flow, an ad-hoc end-to-end test through the Flask test client (add/duplicate/not-found/sort). The `entry.film` crash bug in `get_watchlist` was caught precisely *because* I insisted on exercising a non-empty watchlist rather than trusting the code looked right.
- **Design responses (Comments 4 & 5):** the positions and the pushback are mine. I used AI to pressure-test the arguments and articulate the tradeoffs clearly, but the calls — keep `public=True` on the community-app rationale, and oldest-first over the reviewer's newest-first — are judgments I chose and would defend.
- **Git surgery:** AI proposed the cherry-pick reconstruction to reword the messy commit without interactive rebase; I reviewed each step and confirmed the final history was linear, merge-free, and conventional.

What I did *not* do: accept generated code without running it, or let the tool make the design calls for me.

## Comment 1 — Rename
**What I did:**
Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` so the watchlist service matches the project's `verb_to_noun` naming convention documented in `CONTRIBUTING.md` (`add_to_collection`, `remove_from_collection`, `get_collection`). Updated both the import and the call site in `routes/watchlist/watchlist.py`.

**Where I looked to find all call sites:**
Ran a project-wide search — `grep -rn "save_to_watchlist" --include="*.py" .` (excluding `.venv/`) — before touching anything. It returned exactly three hits: the definition in `services/watchlist_service.py:12`, the import in `routes/watchlist/watchlist.py:8`, and the invocation in `routes/watchlist/watchlist.py:32`. There are no tests referencing the old name yet (Comment 3 adds those against the new name).

**How I verified:**
After editing, re-ran the same grep — zero remaining `save_to_watchlist` references — and confirmed all three sites now use `add_to_watchlist`. Ran the full suite (`pytest tests/ -v`): 4/4 passing, no regressions. Committed on its own as `refactor:` (naming change, no behavior change).

## Comment 2 — Deduplication
**What I did:**
Modeled the fix directly on `add_to_collection()` in `services/collection_service.py`, which handles duplicates in two layers. I replicated both for the watchlist:
1. **Service-level check** — added an `AlreadyInWatchlistError` exception class and, before creating the entry, query for an existing `WatchlistEntry` with the same `(user_id, film_id)`. If one exists, raise the typed error instead of silently inserting a duplicate.
2. **DB-level backstop** — added a `UniqueConstraint("user_id", "film_id", name="unique_user_film_watchlist")` to the `WatchlistEntry` model, exactly as `CollectionEntry` has. This closes the race window the pure service check leaves open.
3. **Route handling** — translated `AlreadyInWatchlistError` → HTTP 409 in `routes/watchlist/watchlist.py`, mirroring the collection route. I also wired up `FilmNotFoundError` → 404, which was imported but never caught (it would previously have surfaced as a 500).

**How I verified the deduplication logic works:**
Since the watchlist test file didn't exist yet (Comment 3), I ran an ad-hoc verification against an in-memory DB: added a film to a user's watchlist once (succeeded), added the same film a second time (raised `AlreadyInWatchlistError`), and confirmed `WatchlistEntry.query.filter_by(user_id, film_id).count() == 1` — no duplicate row was written. Comment 3 then codifies the duplicate case as a permanent regression test. Full suite (`pytest tests/ -v`) stayed at 4/4 passing. Committed separately from the rename as `fix:`.

## Comment 3 — Missing test
**What I did:**
Created `tests/test_watchlist.py`. I used `tests/test_collection.py` as my model — copying the `app`, `sample_user`, and `sample_film` fixtures verbatim so the setup is identical, then writing the watchlist equivalents of the collection tests. The specific test the reviewer asked for is `test_add_to_watchlist_nonexistent_film_raises`, modeled directly on **`test_add_to_collection_nonexistent_film_raises`**: same shape — a `sample_user`, a fake film id, and `with pytest.raises(FilmNotFoundError): add_to_watchlist(...)`.

I reused the reference test's fake id string (`"00000000-0000-0000-0000-000000000000"`). I first checked empirically that `db.session.get(Film, <uuid-string>)` returns `None` (not an error) under the current integer `Film.id` schema, so the test passes now *and* stays valid after the Comment 6 rebase migrates film ids to UUID.

Beyond the requested test, CONTRIBUTING.md requires happy-path and duplicate/conflict tests for any new service function, so I also added `test_add_to_watchlist_creates_entry` and `test_add_to_watchlist_duplicate_raises` — the latter also serves as the permanent regression test for the Comment 2 deduplication fix.

**How I verified:**
`pytest tests/test_watchlist.py -v` → 3 passed. Then the full suite `pytest tests/ -v` → 7 passed (4 collection + 3 watchlist), confirming no regressions across the three code changes. Committed on its own as `test:`.

## Comment 4 — Default visibility
**My position:**
Keep `public=True` as the default for `WatchlistEntry`. This is a deliberate choice, not an oversight, and I'd like to defend it rather than flip it.

**Reasoning — the user behavior I'm optimizing for:**
CineLog is described in the README as *"a community film tracking app."* The whole reason a watchlist has social value is discovery: seeing what friends and other users are excited to watch is how people find their next film and how the community's network effects actually materialize. A watchlist is fundamentally an *expressive, aspirational* artifact ("here's what I'm looking forward to") — much lower sensitivity than, say, viewing history — and sharing it is the point.

The closest real-world analog to CineLog is Letterboxd, and it's instructive that Letterboxd makes user watchlists **public by default**. The dominant product in this exact domain treats the watchlist as a social object, because that's what drives engagement and recommendations. Defaulting to public means the community features have content to work with on day one, instead of sitting empty behind an opt-in that the large majority of users never toggle. Defaults are powerful precisely because most people never change them — so the default *is* the product decision about what kind of app this is.

**Tradeoff acknowledged (the case for the other option — private by default):**
Private-by-default is the more conservative, privacy-by-design choice, and it has a real argument: it honors the principle of least surprise, and it protects the user who doesn't realize that "I want to watch this" is being broadcast. Publishing user-generated data without explicit consent is the kind of thing that erodes trust, and the damage is asymmetric — an accidental over-share is far harder to walk back than an under-share (a user who wants visibility just flips one toggle). If CineLog's audience skewed toward users who treat their to-watch list as private planning rather than public expression, private-by-default would be the correct call, and I would not fight it.

I'm choosing public because it matches the app's stated community mission and the genre precedent, **but** the honest mitigation is that this choice is only defensible if the public nature is *disclosed at the moment of adding* (so it isn't a surprise) and the per-entry `public` flag already on the model is exposed as an easy, visible toggle. Without that disclosure, the reviewer's instinct toward private would win. I'd want that disclosure treated as a requirement that ships alongside the default, not a nice-to-have.

## Comment 5 — Sort order
**My position:**
I'm proposing a third option. I agree with the maintainer that alphabetical-by-title is the wrong default and that sorting should be based on `date_added` — so I've changed it — but I've gone with **date_added ascending (oldest first)** rather than the newest-first order the collection uses. I also removed the now-unnecessary `join(Film)`.

**Reasoning:**
The reviewer's alphabetical critique is correct: title order is arbitrary for a to-watch list. Nobody opens their watchlist thinking "show me the titles starting with A" — they open it to decide *what to watch next*, and alphabetical order carries no information about that. So the sort should be time-based.

Given time-based, the question is direction. I chose oldest-first because a watchlist is a **backlog to work through**, and the failure mode of every watchlist is the "graveyard": films get added in a burst of enthusiasm and then sink out of view forever. Oldest-first directly counteracts that — the film that's been waiting longest is the one the user is most at risk of never watching, so it's the one worth surfacing. FIFO is the natural ordering for a queue you intend to drain.

**Engagement with the reviewer's point:**
The maintainer's argument is consistency with `get_collection`, which is newest-first. I think that argument is half right and half a false economy, and it's worth separating the two halves:
- **Consistency of *mechanism* — agreed.** Both lists should sort by `date_added`, not by ad-hoc fields. My change brings the watchlist in line with that.
- **Consistency of *direction* — I disagree.** The collection and the watchlist have opposite time-orientations. The collection is a *retrospective log* of films already watched; "what did I most recently watch/rate" is genuinely the most relevant thing at the top, so newest-first is right *there*. The watchlist is a *prospective queue* of films not yet watched; the goal is to eventually clear it, and newest-first actively works against that goal by burying the oldest, most-neglected entries under every new impulse-add. Making the two lists sort the same direction just because they're both lists optimizes for surface symmetry over what each list is *for*.

I'll concede the honest counterargument: a user who just added a film they're excited about will expect to see it near the top, and oldest-first pushes it to the bottom. That's a real cost. My view is that recency-of-excitement is better served by an explicit, user-controlled sort (a `?sort=` param — a reasonable follow-up) than by making it the silent default, because the *default* should serve the watchlist's core job of not-forgetting, and the enthusiastic just-added film is the one the user least needs help remembering. If the maintainer still prefers newest-first after this, I'd treat it as a genuine judgment call rather than a correctness issue and defer — but I wanted to make the backlog argument explicitly rather than just mirror the collection.

*(Note: while adding a non-empty-watchlist test to verify this ordering, I found `get_watchlist` would have crashed on any non-empty watchlist — `WatchlistEntry` had no `film` backref. Fixed separately in `fix: add Film.watchlist_entries relationship`, since it's an unrelated latent bug, not a sort-order concern.)*

## Comment 6 — Rebase
**What conflicted:**
`models.py`, and only `models.py`. While my PR was open, `main` merged `refactor: migrate film IDs from integer to UUID`, which did two things to that file: (1) changed `Film.id` and `CollectionEntry.film_id` from `Integer` to `String(36)` UUID, and (2) deleted the `WatchlistEntry` class entirely. My branch, meanwhile, had modified `WatchlistEntry` (I added a `UniqueConstraint` in the deduplication commit). Git saw "main deleted this block / my commit modified this block" and surfaced the whole `WatchlistEntry` class as a conflict when it replayed my `fix: prevent duplicate watchlist entries` commit. `Film` and `CollectionEntry` themselves auto-merged — the refactor's UUID versions were taken without my intervention because my branch didn't touch those lines.

Two pre-existing setup steps before the rebase would even start: an untracked `.gitignore` (main now tracks its own, a superset that adds `.pytest_cache/`) would have blocked the base checkout, so I removed the local copy and let main's tracked version take over; and I backed up the untracked `pr-response.md` to scratch as insurance (it isn't in main, so the rebase left it alone).

**How I resolved it:**
I kept `WatchlistEntry` and migrated it into the new UUID world — changed its `film_id` from `db.Column(db.Integer, ...)` to `db.Column(db.String(36), ...)` so its foreign key matches the now-UUID `film.id`, while preserving the `UniqueConstraint` that commit had added. That's the core of what the reviewer asked for: "update your watchlist code to use UUIDs where it still references integer IDs." My later `fix: add Film.watchlist_entries relationship` commit then replayed cleanly on top with no conflict. Git only flags files that exist on both sides, so it did *not* flag the stale `film_id (int)` / `Body: { "film_id": <int> }` references in the watchlist service and route (those files don't exist on main) — I found and fixed those manually in a follow-up `refactor:` commit so no integer-ID references survive anywhere.

**History rewrite (part of the resubmission standard):**
The oldest commit on the branch was `added watchlist model and endpoint / fixed a bug / more changes` — three of CONTRIBUTING.md's explicitly "Not acceptable" examples in one message. I reworded it to `feat: add watchlist service and endpoints` (its diff is cohesively one logical change — blueprint registration + route + service — so it needed a reword, not a split). Because interactive rebase (`git rebase -i`) isn't available in this environment, I did it by reconstruction: detach at `origin/main`, cherry-pick the messy commit, `commit --amend` its message, then cherry-pick the remaining eight commits back on top, and repoint `feature/watchlist` at the result. All nine commits are now valid Conventional Commits, each a single logical change.

**How I verified no conflict remains:**
- `grep` for conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`) across all `.py` files → none.
- `git log --merges origin/main..HEAD` → empty, confirming a linear branch with no merge commits.
- `git log --format=%s origin/main..HEAD` → every subject line starts with a valid type prefix (`feat/fix/refactor/test`).
- Full suite `pytest tests/ -v` → 8 passed (4 collection + 4 watchlist).
- End-to-end HTTP smoke test with real UUID film IDs through the Flask test client: add → `201`, duplicate → `409`, nonexistent film → `404`, missing body → `400`, and `GET /watchlist/<user_id>` → `200` with oldest-first ordering. This proves the UUID foreign keys actually resolve at runtime, not just that the file parses.

## PR Description

### What this feature does
Adds a **watchlist** to CineLog — a per-user list of films a user wants to watch (as distinct from the collection, which is films already watched). Users can save a film to their watchlist and retrieve their full watchlist. Duplicate saves are rejected, and the list is returned oldest-first so the longest-waiting films surface first.

### Endpoints
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/watchlist/<user_id>` | Return the user's watchlist, oldest-added first |
| POST | `/watchlist/<user_id>/add` | Add a film (`{ "film_id": "<uuid>" }`) |

Status codes: `201` created, `400` missing `film_id`, `404` film not found, `409` already on the watchlist.

### Design decisions
- **Default visibility is `public=True`** (Comment 4). Optimizes for CineLog's community/discovery mission, following the Letterboxd precedent for watchlists; the per-entry `public` flag lets users opt out. Tradeoff and mitigation (add-time disclosure) discussed above.
- **Sort order is `date_added` ascending / oldest-first** (Comment 5). A watchlist is a backlog to drain, so FIFO surfaces the most-neglected film. This intentionally diverges from the collection's newest-first ordering because the two lists have opposite time-orientations (retrospective log vs. prospective queue).
- **Deduplication mirrors the collection** (Comment 2): a service-level check raising `AlreadyInWatchlistError` plus a DB `UniqueConstraint` on `(user_id, film_id)`.
- Naming follows the `verb_to_noun` convention: `add_to_watchlist`, `get_watchlist` (Comment 1).

### How to manually test end to end
```bash
pip install -r requirements.txt
python app.py   # serves on http://localhost:5000
```
You need a `user_id` and a `film_id` (both UUIDs). Grab a film id from `GET /films/` and a user id from your seeded data, then:
```bash
# Add a film to the watchlist -> 201
curl -X POST http://localhost:5000/watchlist/<user_id>/add \
     -H 'Content-Type: application/json' -d '{"film_id": "<film_uuid>"}'

# Add the same film again -> 409 (deduplication)
curl -X POST http://localhost:5000/watchlist/<user_id>/add \
     -H 'Content-Type: application/json' -d '{"film_id": "<film_uuid>"}'

# Add a bogus film id -> 404
curl -X POST http://localhost:5000/watchlist/<user_id>/add \
     -H 'Content-Type: application/json' -d '{"film_id": "00000000-0000-0000-0000-000000000000"}'

# View the watchlist -> 200, oldest-added film first
curl http://localhost:5000/watchlist/<user_id>
```
Automated equivalent: `pytest tests/ -v` (8 tests, covering both services).
