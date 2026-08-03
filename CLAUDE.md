# blog.lgcode.me

Hugo (extended, v0.164.0) + PaperMod, deployed to GitHub Pages by
`.github/workflows/hugo.yaml`. Posts live in `content/posts/*.md`.

## How publishing works

There is no CMS, queue service or external state. Scheduling is a property of the
content itself:

- `hugo.yaml` sets `buildFuture: false`, so a post whose `date` is in the future
  is **excluded from the build entirely** — its page, the list pages, the sitemap
  and the RSS feed. It does not exist on the site.
- The deploy workflow runs on every push **and on a six-hourly cron**
  (00:10/06:10/12:10/18:10 UTC = 21:10/03:10/09:10/15:10 local). A post
  therefore goes live on the first build after its `date` passes, with no action
  from anyone.

So "publish a post" means: set `draft: false`, set `date` to a future moment,
commit, push. Cadence lives in the post dates, not in the cron — the cron only
sets the granularity, and six-hourly means any time-of-day you write in a `date`
goes live the same day. (It was daily at first; that fixes the publish hour, so
a post stamped later in the day appears a day after its own printed date.)

**Both conditions matter and they fail differently.** `draft: true` hides a post
from every build *forever*, whatever its date says — a date alone publishes
nothing. A future `date` with `draft: false` is what actually schedules.

Default slots when the tool picks: **Wednesday and Saturday, 09:00 -03:00**.

## The `scripts/blog` helper

```
scripts/blog queue                      # drafts / scheduled / published + next free slots
scripts/blog new my-post-slug           # draft from the archetype (draft: true)
scripts/blog preview                    # local server WITH drafts and future posts
scripts/blog schedule FILE [FILE ...]   # next free slots, in the order given
scripts/blog schedule --drafts          # every draft, filename order
scripts/blog schedule --dry-run ...     # show the plan, write nothing
scripts/blog schedule --start 2026-09-01 ...
scripts/blog arm FILE ...               # publish on the date already in the file
scripts/blog arm --drafts               # ...for every draft
scripts/blog unschedule FILE ...        # back to draft: true
scripts/blog check                      # production build; list what is live right now
```

**`schedule` vs `arm`.** `schedule` owns the date: it picks the next free slot
and overwrites whatever was there. `arm` trusts a date you set by hand and only
flips `draft` — use it when you've laid out the dates yourself, because
`schedule` would silently replace them. Anything undated is skipped either way.

A past date means "publish on the very next build", so `arm` treats it by
intent: **`--drafts` skips past-dated posts** (a bulk sweep is where a stale date
turns into an accidental publish), while naming a file explicitly arms it and
prints a warning — that's a coherent "put this out now".

Slot assignment is *earliest free*: the next Wed/Sat 09:00 strictly after now
that no other post already occupies. A post manually pinned far out therefore
does not push the queue behind it — gaps get filled.

`schedule` refuses to touch an already-scheduled or already-published post
without `--force`, so re-running it on a whole directory is safe.

The script rewrites only the `date:` and `draft:` lines, in place. It is not a
YAML round-trip — frontmatter here uses folded blocks (`description: >-`) that a
naive parser would mangle. Keep it that way.

## Normal batch flow

1. Write several posts as drafts (`draft: true`). They are invisible to the site
   at every stage.
2. Review: `scripts/blog preview`, read them at `localhost:1313`, fix.
3. `scripts/blog schedule --drafts --dry-run` to see the slot plan, then drop
   `--dry-run` (or pass the files explicitly to control the order). If you set
   the dates yourself instead, use `scripts/blog arm` so they survive.
4. Commit and push. Nothing changes on the site yet.
5. Each post appears on its own date. Verify with `scripts/blog queue`.

To pull a post back before its date: `scripts/blog unschedule <file>`, commit,
push. Before the date passes this is invisible — nothing was ever published.

## Conventions

- Frontmatter: `title`, `date`, `draft`, `tags`, `description`. `description` is
  the meta description and the list-page excerpt — write it, don't leave it empty.
- Dates always carry an explicit `-03:00` offset. Uruguay has no DST, so the
  offset is constant; `timeZone` in `hugo.yaml` only covers hand-written dates
  that omit it.
- `public/` and `resources/_gen/` are build output and gitignored. `scripts/blog
  check` writes to `public/`.
- The theme is a git submodule (`themes/PaperMod`); CI checks out with
  `submodules: recursive`.

## Operational notes

- The workflow has `contents: write` for one reason: GitHub disables scheduled
  workflows after 60 days with no *commits* (workflow runs don't count), which
  would silently stop the publisher during a gap between writing batches. The
  keepalive step commits a timestamp only when the repo has been quiet for 45+
  days. Remove that step and drop the permission back to `read` if you'd rather
  watch for GitHub's disable-notification email yourself.
- Cron runs are best-effort and can be delayed under load; posts publish late,
  never early. `workflow_dispatch` force-builds if you need a post out now.
- **Cross-post links are a scheduling constraint.** A post that links to another
  post must publish *after* its target, or the link 404s until the target lands.
  `schedule --drafts` orders by filename, which is not publication order — pass
  the files explicitly when one post references another.
- Every post is drawn from private project notes. Client names, infrastructure
  hostnames/IPs, credentials, pricing and dataset/model specifics get scrubbed or
  generalized before a draft is written, not after.
