# ncid

**Date:** 2026-05-20

ncid is a local-first command-line tool that helps you find out whether your
own images are showing up on websites, forums, or subreddits without your
consent. You point it at a page, a domain, or a subreddit; it downloads what's
there, compares it against images you already own, and tells you what
matched. It does not upload your images anywhere unless you explicitly ask it
to, and it does not store copies of anything it finds.

This tool is built for people who need to check where their own images have
ended up — not for searching for other people. See "Intended use and limits"
below.

## Privacy model

ncid was designed around six rules. They are enforced by automated tests, not
just documented as intentions — a change that breaks one of these is treated
as a bug regardless of what feature it enables.

- **Your images stay on your machine.** In normal ("strict-local") use, ncid
  never sends your image, or anything derived from it, to anyone. The only
  outbound network traffic is ordinary page and image downloads from the
  sites you tell it to check.
- **Uploading requires you to type a word, every time.** Reverse-image search
  engines (currently Google Vision) only work by uploading your image to
  that company's servers. ncid will never do this silently. **Your images
  never leave your computer unless you explicitly type SEND to upload one to
  a search engine.** A command-line flag alone is never enough — there is no
  `--yes` shortcut for this prompt, and it tells you exactly which file is
  about to be sent to which company before you decide.
- **Downloaded images are never saved.** When ncid checks a candidate image
  from a website or subreddit, it holds that image in memory only long
  enough to compare it, then discards it. It is never written to disk, to a
  cache, or to the database. ncid is not a tool that builds up a local
  collection of this material.
- **Only metadata is stored, never pictures.** The local database records
  hashes, URLs, match scores, and timestamps — never image bytes, yours or
  anyone else's. Nothing is logged that contains image data. ncid has no
  telemetry, no crash reporting, and no update check; it does not phone
  home.
- **API keys stay in your environment.** Keys are read from environment
  variables only. ncid never writes them to a config file, never logs them,
  and never includes them in a report.
- **You can delete everything, on demand.** `ncid forget` removes
  fingerprints — and, if you ask for it, all local data — completely.
  Deletion is a supported command, not something you have to do by hand in a
  database file.

## Install

```
uv tool install ncid
```

or

```
pipx install ncid
```

From source:

```
git clone <this repo>
cd non-consent-image-detection
uv sync
```

Requires Python 3.12+.

## Quickstart

1. **Fingerprint your images.** This hashes them locally and stores the
   hashes — never the images themselves — in your local database.

   ```
   ncid fingerprint photo1.jpg photo2.jpg --label "vacation set"
   ```

2. **Scan a source.** Point ncid at a page, a domain, or a subreddit:

   ```
   ncid scan --source url:https://example.com/some-page
   ncid scan --source domain:example.com
   ncid scan --source reddit:r/somesubreddit
   ncid scan --source reddit:u/someuser
   ```

   This is strict-local by default: ncid crawls the site or subreddit
   (respecting `robots.txt`), downloads candidate images, hashes them in
   memory, compares them to your fingerprints, and discards the image
   bytes. Nothing of yours goes out.

3. **Read the report.**

   ```
   ncid report
   ncid report --format json
   ncid report --format md --out report.md
   ```

Other commands: `ncid list` shows your stored fingerprints (never the
images), `ncid forget --fingerprint ID` or `ncid forget --all` deletes local
data, and `ncid config` shows the effective configuration and which API keys
are currently set.

## Engine search (opt-in, requires consent)

By default ncid only checks sources you name directly. If you want to ask a
reverse-image search engine to suggest additional leads, pass `--engines`
along with `--image` pointing at the original file:

```
ncid scan --source url:https://example.com --engines google --image photo1.jpg
```

Before anything is sent, ncid prints exactly which file is about to be
uploaded and to which company, and asks you to type `SEND` to confirm. Type
anything else and nothing is uploaded. Leads returned by the engine are not
trusted directly — ncid fetches those pages itself and re-verifies the match
locally before reporting it as more than an unconfirmed lead.

### Google Cloud Vision setup

1. Create a Google Cloud project and enable the **Vision API**.
2. Create an API key restricted to the Vision API.
3. Set the environment variable:

   ```
   export NCID_GOOGLE_API_KEY=your-key-here
   ```

### Reddit setup

1. Go to <https://www.reddit.com/prefs/apps> and create a **script**-type
   app.
2. Set the environment variables:

   ```
   export NCID_REDDIT_CLIENT_ID=your-client-id
   export NCID_REDDIT_CLIENT_SECRET=your-client-secret
   ```

### Other environment variables

- `NCID_DATA_DIR` — override where ncid stores its local database (defaults
  to the platform's standard data directory).

## Source specs

| Spec | Meaning |
|---|---|
| `url:https://example.com/page` | A single page |
| `domain:example.com` | Crawl a domain (still bound by `robots.txt`) |
| `reddit:r/somesubreddit` | A subreddit's recent posts |
| `reddit:u/someuser` | A user's recent submissions |

## Match tiers

Every match ncid reports is labeled with a tier so you know how much to
trust it:

- **exact** — byte-identical, or a near-perfect hash match. Very high
  confidence.
- **probable** — strong hash agreement across multiple hash types. High
  confidence, but not certain.
- **possible** — weaker hash agreement. **Needs human confirmation** —
  look at it yourself before treating it as a real match.
- **unverified-lead** — a search engine suggested this page or image, but
  ncid could not confirm it locally (the page or image was gone, or the
  hash didn't match closely enough when it checked). Kept separate from
  confirmed matches so it can't be mistaken for one.

## Intended use and limits

ncid is meant for **searching for your own images** — checking whether
material of you has been posted somewhere without your consent. It is not a
general reverse-image search tool, and it is not built or intended for
searching for images of other people.

Some honest limits, so you know what to expect:

- **It is not a monitoring service.** ncid checks the sources you point it
  at, once, when you run it. It does not run in the background or watch
  anything continuously. Running it again later, or on a schedule you set
  up yourself, is the way to check for new matches over time.
- **Crops and heavy edits are not detected in this version.** Images cropped
  by more than roughly 20%, or heavily edited, generally will not match.
  Detecting those is planned for a later phase (see Roadmap).
- **`possible` matches need a human to look at them.** Don't treat a
  `possible` tier as confirmed — verify it yourself.
- **This is one tool, not a complete response.** If you're dealing with
  non-consensual image sharing, ncid can help you find where images are, but
  it doesn't remove anything or file reports for you. Two resources worth
  using alongside it:
  - [StopNCII.org](https://stopncii.org) — a service that can help prevent
    known images from being (re-)uploaded to participating platforms.
  - The reporting channels of the specific platform where you found the
    content — most major platforms have a dedicated non-consensual-imagery
    reporting path that is faster than a general abuse report.

## Roadmap

- **Phase 2 — likeness matching.** Detect your face in candidate images
  even when the exact picture doesn't match, using on-device face
  embeddings. Framed honestly as lower-confidence than hash matching, with
  human confirmation required.
- **Phase 3 — AI-alteration signals.** Signals (not verdicts) that an image
  may have been AI-generated or altered — provenance metadata, artifact
  heuristics — evaluated against public datasets before any claim is made
  in the UI.

## Development

```
uv sync
uv run pytest -v
```

Tests include a dedicated privacy-invariant suite (`I1`–`I6` above) that
fails the build if the tool starts sending image data outbound, caching
candidate images, or storing image bytes in the database.
