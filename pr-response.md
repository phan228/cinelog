# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used an AI coding assistant (Claude Code) throughout this project. Concretely:

- **Codebase orientation:** had it read the repo and summarize the architecture (Flask app factory, service/route split, the `collection` feature as the pattern to mirror) before writing anything.
- **Recovering the PR under review:** the watchlist code wasn't present in my clone, so the assistant fetched the actual PR diff and the six maintainer comments from the upstream GitHub PR and reconstructed the as-submitted baseline as the first commit, so each review comment mapped to a real code change.
- **Applying the fixes:** it made the edits for each comment, but I directed the *decisions* — especially the two judgment calls. For **Comment 4** I chose private-by-default (over the assistant's neutral framing of either option) and for **Comment 5** I chose to adopt the maintainer's date-added order with a consistency argument.
- **Verification:** the assistant ran `pytest` after every change and wrote small throwaway scripts to exercise behavior directly (duplicate rejection, private default, newest-first ordering). This is how the latent `entry.film` relationship bug was caught — `get_watchlist` had no test and crashed when first exercised.
- **What I checked myself:** I reviewed every diff and commit message for correctness and convention compliance, confirmed the grep-based call-site search for the rename, and verified the "rebase" claim honestly rather than letting a conflict be fabricated.

AI accelerated the mechanical work (searching, editing, running tests, drafting prose); the design positions, the decision to override the public default, and the correctness review were mine.

## Comment 1 — Rename
> `save_to_watchlist()` should follow the project's naming convention (`verb_to_noun`, like `add_to_collection()`). Rename to `add_to_watchlist()` and update all call sites.

**What I did:**
Renamed `save_to_watchlist()` → `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` convention (consistent with `add_to_collection`, `remove_from_collection`, `get_collection`).

To find every call site I ran a project-wide search rather than trusting memory:
`grep -rn "save_to_watchlist" . --exclude-dir=.git --exclude-dir=.venv`

That surfaced exactly three references:
1. `services/watchlist_service.py:12` — the function definition
2. `routes/watchlist/watchlist.py:8` — the import
3. `routes/watchlist/watchlist.py:32` — the call inside `add_film()`

I updated all three (and also reworded the docstring "Save a film…" → "Add a film…" for consistency).

**How I verified:**
- Re-ran the same grep afterward — zero remaining references to `save_to_watchlist` (only `add_to_watchlist` now).
- Ran the full suite (`pytest tests/ -v`) — 4 passed, no import errors.
- Commit: `refactor: rename save_to_watchlist to add_to_watchlist` (`f47cd00`).

## Comment 2 — Deduplication
> What happens if a user adds a film that's already on their watchlist? The current implementation creates a duplicate. Please handle it (compare with `add_to_collection()`).

**What I did:**
Followed the exact pattern in `add_to_collection()`. Before inserting, `add_to_watchlist()` now queries for an existing entry for the `(user_id, film_id)` pair and raises a new `AlreadyInWatchlistError` if one exists — the mirror of `AlreadyInCollectionError`:

```python
existing = WatchlistEntry.query.filter_by(user_id=user_id, film_id=film_id).first()
if existing:
    raise AlreadyInWatchlistError(...)
```

I also completed the route: `routes/watchlist/watchlist.py` now catches `AlreadyInWatchlistError` → HTTP 409 (and the already-imported `FilmNotFoundError` → 404, which was previously imported but never caught), matching the collection `add_film` endpoint's error mapping. The model already had a `UniqueConstraint(user_id, film_id)`, so this is defense at the service layer in front of the DB constraint — same as collection.

**How I verified:**
- Wrote a throwaway script that adds the same film twice in an in-memory app: first add returns an entry, second raises `AlreadyInWatchlistError`, and `WatchlistEntry.query.count()` stays at **1** (no duplicate persisted).
- The regression test `test_add_to_watchlist_duplicate_raises` (added under Comment 3) now locks this in.
- Full suite green. Committed separately from the rename: `fix: prevent duplicate entries in add_to_watchlist` (`6193ce1`).

## Comment 3 — Missing test
> Add a test for the case where `film_id` doesn't exist in the database. The pattern is in `test_collection.py`.

**What I did:**
Created `tests/test_watchlist.py`. I used **`test_add_to_collection_nonexistent_film_raises`** as my model — copied its fixture setup (`app` with in-memory SQLite, `sample_user`, `sample_film`) and its assertion shape (`with pytest.raises(FilmNotFoundError):` against a fake UUID `00000000-0000-0000-0000-000000000000`).

Because `add_to_watchlist()` is a *new* service function, CONTRIBUTING.md requires happy-path + duplicate + nonexistent-ID coverage, so I mirrored all three of `test_collection.py`'s cases:
- `test_add_to_watchlist_creates_entry` (happy path)
- `test_add_to_watchlist_duplicate_raises` (conflict — locks in Comment 2)
- `test_add_to_watchlist_nonexistent_film_raises` (the explicitly requested case)

**How I verified:**
- `pytest tests/test_watchlist.py -v` → 3 passed.
- `pytest tests/ -v` → all 7 pass (4 collection + 3 watchlist).
- Commit: `test: add watchlist service tests` (`8f62bd8`).

## Comment 4 — Default visibility
> I notice watchlists default to `public=True`. We don't have a documented decision on default visibility for user lists. Add a note explaining your reasoning — I want us being intentional, not just inheriting a default.

**My position:**
New watchlists should default to **private** (`public=False`). I changed the code to match this position rather than only documenting the inherited `True` — commit `269a157` (`fix: default new watchlists to private`). Sharing becomes an explicit opt-in.

**Reasoning — what user behavior I'm optimizing for:**
A watchlist is a record of *future intent* — "films I want to watch." That is more revealing than a watched-history: it can signal mood, plans, tastes a user hasn't committed to, or films they're curious about but might not want associated with them publicly. I'm optimizing for the user who adds films quickly without stopping to think about audience — they should never be surprised to learn their in-progress list was world-visible the whole time.

The decisive argument is **reversibility asymmetry**: making a private list public later is a safe, lossless one-tap action; making a public list private *after the fact does not undo exposure* — it may already have been seen, screenshotted, or indexed. When one default is reversible and the other isn't, the safe default is the reversible one. This is the standard privacy-by-default / privacy-by-design principle, and it's the intentional choice the reviewer asked for — I'm deliberately *overriding* the inherited `True`, not inheriting it.

On the "CineLog is a community app" counterpoint: community value should come from *intentional* sharing, which is also more trustworthy and higher-signal than lists that are public only because nobody changed a default. A single visibility toggle preserves the entire social loop at negligible friction.

**Tradeoff acknowledged:**
Private-by-default has a real cost. It weakens out-of-the-box social discovery: the community/browse features start with less public content to surface, and users who *would* have happily shared won't all bother to flip the toggle, so the network effect builds more slowly. If CineLog's growth model depends on public watchlists seeding discovery, this default actively slows that engine. I accept that cost: a slower discovery ramp is recoverable; exposing users by default is not. If product later shows discovery is critical, the right move is a clear opt-in *prompt* at creation ("Share this watchlist?"), not a silent public default.

## Comment 5 — Sort order
> I'd prefer watchlists to default to "date added" order rather than alphabetical. Most users want to see what they added recently. Open to discussion — but let's decide and document.

**My position:**
I **implemented the maintainer's preference**: `get_watchlist()` now sorts by `date_added` descending (newest first) instead of `Film.title` alphabetical — commit `1e0a60e`. I also added a regression test (`test_get_watchlist_returns_newest_first`, commit `a6f70b6`).

**Reasoning + engagement with the reviewer's point:**
I agree with the maintainer's stated reasoning and think it's stronger than they framed it. Their point — "most users want to see what they added recently" — is exactly right for a watchlist specifically: it's a *queue of intent*, and the most recent add is the film most likely on the user's mind right now. Alphabetical order optimizes for *lookup* ("is X on my list?"), but lookup is better served by search/filter than by sort; the default sort should serve browsing, and for browsing recency wins.

I'll add an argument the maintainer didn't raise, which makes the decision easy: **consistency**. `get_collection()` already sorts `date_added` descending. Having the two nearly-identical list endpoints sort by different keys (collection by recency, watchlist by title) is a surprising inconsistency that would trip up both API consumers and future maintainers. Aligning them removes that cognitive cost, and it let me delete the `.join(Film)` that only existed to support title ordering.

I considered two alternatives and rejected them:
- **Keep alphabetical:** its only real win (scannable lookup on large lists) is better solved by explicit search, and it breaks consistency with collection. Rejected.
- **Oldest-first (FIFO queue):** defensible if you model a watchlist as a strict "watch these in order" backlog. But users add films opportunistically, not as an ordered plan, and burying the newest add at the bottom hides the item they most likely came to act on. Rejected.

If a user genuinely wants alphabetical, the clean future extension is an optional `?sort=title` query param — an additive change that keeps recency as the sensible default without re-litigating it.

## Comment 6 — Rebase
> A refactor merged to `main` changed film IDs from integers to UUIDs. Your watchlist code still references integer IDs. Please rebase on `main` and update accordingly.

**What conflicted:**
Honest answer: **no git-level conflict occurred.** My `feature/watchlist` branch was already based on the tip of `main` (`bbe206c`), which already contains the UUID refactor (`07ca580 refactor: migrate film IDs from integer to UUID`). I verified this before doing anything:

```
$ git merge-base HEAD origin/main   → bbe206c...
$ git rev-parse origin/main         → bbe206c...   (identical)
$ git rebase origin/main            → "Current branch feature/watchlist is up to date."
```

So the rebase was a no-op fast-forward — there was no text conflict to resolve, and I did not manufacture one.

What the reviewer correctly caught was a **semantic** leftover, not a merge conflict: the watchlist *code* still *described* `film_id` as an integer even though the schema was UUID. Specifically:
- `services/watchlist_service.py` docstring: `film_id (int): ID of the film. (Note: integer — pre-refactor)`
- `routes/watchlist/watchlist.py` docstring: `Body: { "film_id": <int> }`

**How I resolved it:**
- Ran `git rebase origin/main` (confirmed up-to-date, no replay needed).
- Updated the two stale references to UUID: `film_id (str): UUID of the film.` and `Body: { "film_id": "<uuid>" }` — commit `00ed9ae`.
- Confirmed the `WatchlistEntry` FK columns were already `db.String(36)` (UUID), consistent with `User.id`/`Film.id`, so no schema/migration change was required.

**How I verified no conflict remains:**
- `git status` clean, no rebase in progress, linear history (no merge commits — satisfies CONTRIBUTING's "No Merge Commits" rule).
- `grep -rnE "\bint\b|<int>|integer"` over the watchlist code returns only the pre-existing migration-history comments in `models.py`, none in the watchlist feature.
- Full suite green (8 passed), and a manual `add_to_watchlist` / `get_watchlist` round-trip works with real UUID film IDs.

**Note for the maintainer:** because my clone was already current with `main`, I could not reproduce the actual integer→UUID rebase conflict a stale branch would have hit. If my branch had been cut *before* `07ca580`, the rebase would have replayed my watchlist commits onto the UUID schema and I'd have had to change `WatchlistEntry.film_id`/`user_id` column types from `Integer` to `String(36)` and fix the FK types — that's the resolution I would have applied.

## PR Description

### What this feature does
Adds a **watchlist** so users can save films they *want to watch* (distinct from the existing collection, which is films they've *already watched*). It introduces a `WatchlistEntry` model (a user↔film join with `date_added` and a `public` visibility flag, plus a unique `(user_id, film_id)` constraint), a `watchlist_service` layer, and REST endpoints under `/watchlist`.

Endpoints:
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/watchlist/<user_id>` | Return a user's watchlist, newest-first |
| POST | `/watchlist/<user_id>/add` | Add a film. Body: `{ "film_id": "<uuid>" }` |

### Design decisions
- **Visibility defaults to private (`public=False`).** Watchlists express future intent, exposure is irreversible while sharing is a lossless opt-in, so the safe default is private. Full reasoning + tradeoff in **Comment 4** above.
- **Sorted newest-first (`date_added` desc).** Matches user expectation for a queue of intent and stays consistent with `get_collection()`. A `?sort=title` param is the clean future extension. See **Comment 5**.
- **Deduplication at the service layer** mirrors `add_to_collection`: a second add of the same film raises `AlreadyInWatchlistError` → HTTP 409, in front of the DB unique constraint. See **Comment 2**.
- **Naming** follows the project's `verb_to_noun` convention (`add_to_watchlist`, `get_watchlist`). See **Comment 1**.

### How to manually test end to end
```bash
pip install -r requirements.txt
python app.py                      # starts on http://localhost:5000
```
You need a real user UUID and film UUID from the DB (create/seed one, or use `flask shell`). Then:
```bash
# 1. Watchlist starts empty
curl http://localhost:5000/watchlist/<user_id>
# → []

# 2. Add a film
curl -X POST http://localhost:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" -d '{"film_id": "<film_uuid>"}'
# → 201, entry JSON with "public": false

# 3. Adding the same film again is rejected
curl -X POST http://localhost:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" -d '{"film_id": "<film_uuid>"}'
# → 409 {"error": "Film '<uuid>' is already on this user's watchlist"}

# 4. Unknown film → 404
curl -X POST http://localhost:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" -d '{"film_id": "does-not-exist"}'
# → 404 {"error": "No film found with id 'does-not-exist'"}

# 5. Add a second film, then GET — most recently added comes first
curl http://localhost:5000/watchlist/<user_id>
# → newest-first list
```
Automated coverage: `pytest tests/ -v` → 8 passing (4 collection + 4 watchlist: happy path, duplicate, nonexistent film, sort order).

### Commits (linear, Conventional Commits, one logical change each)
```
feat: add watchlist feature
refactor: rename save_to_watchlist to add_to_watchlist          (Comment 1)
fix: prevent duplicate entries in add_to_watchlist              (Comment 2)
test: add watchlist service tests                               (Comment 3)
fix: default new watchlists to private                          (Comment 4)
fix: sort watchlist by date added, newest first                 (Comment 5)
fix: add WatchlistEntry film/user relationships                 (bug found while testing sort)
test: assert watchlist is sorted newest-first                   (Comment 5)
docs: update watchlist film_id references from int to UUID      (Comment 6)
```
