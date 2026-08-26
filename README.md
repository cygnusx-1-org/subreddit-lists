# subreddit-lists

This is a maintained list of SFW(Safe for Work) and NSFW(Not Safe for Work) subreddits.

The lists are intended for clients that want to offer a "random subreddit" feature. Reddit's own
`/r/random` and `/r/randnsfw` endpoints no longer work. Both return HTTP 404 with
`reason: banned`, so a client that wants this feature has to bring its own pool of subreddit names.

## Files

| File | Contents |
|---|---|
| `subreddits-sfw.txt` | Subreddits not flagged NSFW by Reddit |
| `subreddits-nsfw.txt` | Subreddits flagged NSFW by Reddit |

## Format

Both files are plain text, one subreddit name per line:

```
AskReddit
askscience
NoStupidQuestions
```

Rules, so a parser can rely on them:

- One name per line, terminated by LF (`\n`). The file ends with a trailing newline.
- No `r/` prefix, no comments, no blank lines, no trailing whitespace.
- Names match `^[A-Za-z0-9_]+$`, with one exception: `reddit.com`, the original 2006 subreddit,
  is the only name containing a dot. A parser that must be strict can use `^[A-Za-z0-9_.]+$`.
- Names are 2 to 21 characters. A handful of two-character names (`de`, `fr`, `ja` and other
  language subreddits) predate Reddit's current three-character minimum.
- Sorted byte-wise, as produced by `LC_ALL=C sort -u`. Sorting is case-sensitive, so uppercase
  names sort before lowercase ones.
- UTF-8, though in practice every byte is ASCII.

Plain text rather than JSON because the expected access pattern is "pick N names at random", which
needs no parser at all: scan once for newline offsets, then read only the lines you picked. A JSON
array would be about 22% larger and would force a full parse to reach the same place. There is no
per-row metadata to carry, because the NSFW split is expressed by which file a name is in.

## How the lists are built

Every subreddit Reddit has ever had was enumerated from the
[Arctic Shift](https://arctic-shift.photon-reddit.com) archive, 27.7 million of them, including
21.7 million `u_*` per-account profile subreddits. Those were discarded, along with communities
under 100 subscribers, and the remainder was checked against Reddit's live API. Only subreddits
that still exist and are still reachable are published here.

Reddit's `over18` flag decides which of the two files a subreddit lands in.

## Updating a cached copy

Fetch the files from the raw CDN:

```
https://raw.githubusercontent.com/cygnusx-1-org/subreddit-lists/master/subreddits-sfw.txt
https://raw.githubusercontent.com/cygnusx-1-org/subreddit-lists/master/subreddits-nsfw.txt
```

Use a conditional GET so an unchanged file costs nothing:

```
GET /cygnusx-1-org/subreddit-lists/master/subreddits-sfw.txt
If-None-Match: "<the ETag stored from the last 200 response>"

    304 Not Modified   unchanged, no body is transferred
    200 OK             changed, the body is the new file,  replace your cache and store the new ETag
```

Suggested cadence:

- Check no more than once every 24 hours.
- If a check fails like a network error, timeout, 5xx, or any 4xx, then retry hourly until one succeeds,
  then return to the 24 hour cadence.
- Track each file independently. One can change without the other.

Do not poll the GitHub REST API for this. Unauthenticated it allows only 60 requests per hour per
IP address, which is easy to exhaust when many clients share an address behind carrier NAT. The raw
CDN has no such limit.

### The ETag is opaque

The value `raw.githubusercontent.com` returns looks like a SHA-256, 64 hexadecimal characters, but
it is not the SHA-256 of the file, nor of its git blob. Store the string and send it back verbatim.
Do not try to compute or verify it locally.

If you want to verify a download by hand, `sha256sum` the file and compare it against a copy you
trust.

## Recommended client usage

A name in these lists existed when the list was generated. It may have been banned, taken private,
or deleted since. Rather than checking one subreddit at a time, validate a batch:

1. Pick 100 names at random from the cached file.
2. Ask Reddit about all 100 in a single request:
   `GET https://oauth.reddit.com/api/info?sr_name=name1,name2,...&raw_json=1`
3. In the response, a subreddit with a non-null `subscribers` value is live. One that comes back
   with `subscribers: null`, or does not come back at all, is banned, private, or gone then skip it.
4. Use one of the survivors and keep the rest as a pre-validated pool for subsequent picks.

That costs one network round trip per roughly 99 picks rather than one per pick. The response also
carries a current `over18` for each subreddit, so a client can re-check the NSFW flag against live
data at the moment it picks.

## Caveats

Read these before relying on the lists.

- **The SFW list is not a safety guarantee.** It means "not flagged NSFW by Reddit", which is not
  the same as "safe". The flag is set by subreddit moderators, so an NSFW subreddit that was never
  flagged appears in the SFW list. Any client showing these to users should say so.
- **Liveness decays.** These are snapshots. In the 18 months between the archive being captured and
  being validated, 16.1% of the subreddits in this set were banned. That is why the batch check
  above matters. The list is a pool of plausible candidates, and the check is what makes it true.
- **The subscriber floor is historical.** A subreddit qualifies if it had 100 or more subscribers
  when the archive was captured and still has 100 or more today. Subreddits that grew past 100 more
  recently are not here, because finding them would mean re-checking millions of names.
- **Not every subreddit Reddit has ever had is represented.** The archive can only contain what it
  observed. Subreddits created and banned quickly, or always private, may never have entered it.
