# The Beatles Encyclopedia

A single-page fan encyclopedia of The Beatles' catalogue: browse the discography,
sort a recording/release timeline, rate every song and run head-to-head "battles"
that feed a shared, live-updating ranking.

The whole app is one static `index.html` — no build step, no dependencies to install.

## Running it

Because the page loads the Firebase SDK as an ES module, it must be served over
HTTP rather than opened straight from the filesystem:

```bash
python3 -m http.server 8000
# then open http://localhost:8000
```

Any static host works (GitHub Pages, Netlify, S3). The page also renders fine
without a network connection to Firebase — see [Offline behaviour](#offline-behaviour).

## Features

| Tab | What it does |
| --- | --- |
| **Discography** | All 27 releases with cover art, tracklists and album metadata. Search matches album titles, years and song titles. |
| **Timeline** | Every song that has both a recording and a release date (239 of 628), sortable on any column, with CSV export and clipboard copy. |
| **Song Ranking → Rate The Songs** | Score songs 1–10. Shows the current community average next to each song and updates it live. |
| **Song Ranking → Battleground** | Pick songs, play every pairwise duel, results are merged into the shared win-rate table. Keyboard: `1`/`←`, `2`/`→`, `Backspace` to undo. |
| **Song Ranking → Song / Album Ranking** | Rating, battleground and blended "overall" leaderboards with CSV export. |
| **Configuration** | Deletes ratings and/or battle results per album from the shared database. |

Tabs are deep-linkable: `#timelines`, `#song-ranking/battleground`, etc.

## Data model

Album and song data is a hard-coded `beatlesData` array inside `index.html`.
Each album carries release metadata plus a `songs` array:

```js
{
  type: 'studio' | 'compilation',
  title, year, cover, released, recorded, studio, genre, length, label, producer,
  songs: [
    { title, duration, authors, recordDate: 'YYYY-MM-DD', releaseDate: 'YYYY-MM-DD' }
  ]
}
```

`recordDate` / `releaseDate` are optional and only present where a reliable date is
known; the Timeline tab lists exactly the songs that have both.

### Rateable albums

Only studio albums, `Past Masters` and a synthetic **Reunion Songs** album (Free as a
Bird, Real Love, Now and Then) can be rated. Compilations are excluded on purpose —
they repackage the same recordings, so including them would let one song accumulate
votes under several different album names.

### Song identity

A song's database key is `slugify("<album title>-<song title>")`, e.g.
`revolver-taxman`. The key is album-scoped, so the `Help!` on the *Help!* album and
the one on a compilation are distinct records.

### Firestore collections

```
beatles_song_ratings/{songId}     { songTitle, albumTitle, numRatings, totalStars, averageRating }
beatles_battle_rankings/{songId}  { songTitle, albumTitle, totalWins, totalBattles }
```

Both are written through transactions so concurrent voters cannot clobber each other's totals.

### Scoring

- **Average rating** — `totalStars / numRatings`, on a 1–10 scale.
- **Win rate** — `totalWins / totalBattles`, as a percentage.
- **Overall score** — `rating × 0.6 + (winRate / 10) × 0.4` when a song has *both*
  ratings and battles. With only one of the two, that component is used on its own,
  so a well-rated song that has never battled is not dragged down by an implied 0% win rate.
- **Album averages** only count songs that actually have data. The *Rated* column shows
  coverage (e.g. `4/17`), so a partly rated album is visibly provisional rather than
  silently penalised.

## Offline behaviour

The Firebase SDK is imported dynamically. If it cannot be reached — offline, corporate
proxy, ad blocker — the page still boots: Discography, Timeline, search, sorting and CSV
export all work, and the Rating/Battleground/Configuration tabs show a banner explaining
what is unavailable and why. A status pill in the header reports the connection state.

## Security notes

⚠️ **The Firestore database is world-writable through this page.** Anonymous
authentication is enough to write ratings, battle results *and* to use the Configuration
tab to delete data. Anyone who can load the page can wipe the rankings.

The `firebaseConfig` in `index.html` is not a secret — web API keys are meant to be
public and identify the project, not authorise access. What actually protects the data is
your Firestore security rules. Consider tightening them, for example:

```js
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /beatles_song_ratings/{songId} {
      allow read: if true;
      allow write: if request.auth != null
        && request.resource.data.numRatings is int
        && request.resource.data.numRatings > 0;
      allow delete: if false;   // deletions via console/admin only
    }
    match /beatles_battle_rankings/{songId} {
      allow read: if true;
      allow write: if request.auth != null;
      allow delete: if false;
    }
  }
}
```

With `allow delete: if false`, the Configuration tab's deletions will fail with a clear
error instead of succeeding for any visitor. There is also no per-user vote limit: one
person can submit unlimited votes for the same song. Adding a `beatles_votes/{uid}_{songId}`
guard document would fix that if the ranking is ever meant to be authoritative.

## Known trade-offs

- **Tailwind is loaded from the Play CDN**, which compiles classes in the browser. That
  keeps the project to a single file with no build step, but it costs a little startup
  time and logs a production warning. Moving to a compiled stylesheet would mean
  introducing a build pipeline.
- **Album/song data lives in the HTML file.** It is ~280 lines of the file and would be
  cleaner as a separate JSON module, at the cost of an extra request.

## Credits

Fan-made, non-commercial project. Album artwork is served from `thebeatles.com` and
Wikipedia; song and release data belong to their respective rights holders.
