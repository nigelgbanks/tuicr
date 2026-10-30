# Gerrit

tuicr can review **Gerrit changes** the same way it reviews GitHub PRs and GitLab MRs: open a
change, read its diff, leave inline comments, and push the review back. A Gerrit change takes the
place of a pull request, and its change number is the "PR number" everywhere in the UI.

Gerrit ships no companion CLI (the other backends wrap `gh`, `glab`, `bkt`, and `az`), so tuicr
talks to the [Gerrit REST API](https://gerrit-review.googlesource.com/Documentation/rest-api.html)
directly over HTTPS.

## Setup

Gerrit authenticates the REST API with an **HTTP password**, not your account password. Generate
one in Gerrit under *Settings → HTTP Credentials*, then:

```bash
# bash/zsh
export GERRIT_USERNAME=jdoe
export GERRIT_PASSWORD=xxxxxxxx...

# PowerShell (persist for your user)
setx GERRIT_USERNAME "jdoe"
setx GERRIT_PASSWORD "xxxxxxxx..."
```

Without credentials tuicr still reads public changes anonymously, but `:submit` needs them.

## How tuicr recognizes a Gerrit remote

Gerrit is always self-hosted, so there is no reserved domain to key on the way `github.com` or
`bitbucket.org` work. tuicr treats a remote as Gerrit when any of these hold:

1. The remote uses Gerrit's canonical SSH port **29418** —
   `ssh://jdoe@review.internal:29418/platform/base`. This is the strongest signal and needs no
   configuration.
2. `GERRIT_URL` names the host.
3. The hostname contains **"gerrit"** — `https://gerrit.example.com/platform/base`. (The same
   heuristic the GitLab backend uses for self-hosted instances.)

If your Gerrit lives on a neutral hostname *and* you clone over HTTPS, set `GERRIT_URL` to the
server root:

```bash
export GERRIT_URL=https://review.internal
```

## Which server tuicr talks to

With `GERRIT_URL` unset, tuicr assumes the REST API is served over HTTPS at the root of the git
remote's own host — `ssh://jdoe@review.internal:29418/platform/base` implies
`https://review.internal`. That is the stock Gerrit layout and needs no configuration.

**When `GERRIT_URL` is set it wins**, for every Gerrit repo. It is the override for the
deployments a git remote cannot describe:

| Situation | Set |
| --- | --- |
| Gerrit under a path prefix | `GERRIT_URL=https://review.internal/gerrit` |
| REST API on a non-default port | `GERRIT_URL=https://review.internal:8443` |
| Plain HTTP, read-only | `GERRIT_URL=http://review.internal` |
| Web host differs from the SSH gateway in the remote | `GERRIT_URL=https://review.corp.com` |

That last row is the one you cannot express any other way: with a remote of
`ssh://gerrit-ssh.corp.com:29418/platform/base`, nothing in the URL names the web host.

A value without a scheme (`review.internal`, `review.internal:8443`) or an SSH URL pasted from
`git remote -v` (`ssh://jdoe@review.internal:29418`) is normalized onto HTTPS. A `29418` port is
dropped — that is Gerrit's *SSH* port and means nothing over HTTPS — but any other port you write
is kept. Use the full `http://…` form if your REST API is not on HTTPS.

**Plain HTTP is anonymous-only.** With `GERRIT_USERNAME`/`GERRIT_PASSWORD` set, a `GERRIT_URL` on
`http://` is refused before the request leaves tuicr. A Gerrit HTTP password is long-lived and
authorizes `git push` as well as the REST API, and HTTP Basic puts it on the wire in a trivially
reversible encoding — so an `http://` server can be browsed anonymously, but not authenticated to.
There is no override. If you need to submit reviews, put the REST API behind HTTPS or reach it
through a tunnel.

Note that `GERRIT_URL` is a process-global environment variable, so it names *one* server. If you
review on two Gerrits, scope it per checkout (direnv, a shell wrapper) or leave it unset where the
hostnames already carry the signal. Your review sessions are still keyed by the *remote's* host, so
setting the variable never changes which saved session a checkout resolves to.

The `/a/` prefix Gerrit uses for authenticated git clones (`https://host/a/my/project`) is
transport, not part of the project name, and is stripped automatically.

## Reviewing a change

Run tuicr from inside a **local clone** of the project (see the diff note below), then:

```bash
tuicr pr 3965                                            # by change number
tuicr pr https://gerrit.example.com/c/platform/base/+/3965   # by URL
tuicr pr https://gerrit.example.com/#/c/3965/                # legacy URL form
```

The **Pull Requests** tab lists open changes for the project your remote points at; `r` toggles
between all open changes and the ones **waiting on you**.

That second list is driven by Gerrit's **attention set** — the same signal behind the *Your Turn*
section of the Gerrit dashboard — as `attention:self -owner:self`. The alternative,
`reviewer:self`, matches every change you are a reviewer on including the ones you already voted
on and handed back to the author, which is the opposite of "needs my review". `-owner:self` drops
your own changes: the attention set holds those too when a reviewer replies, but that is your turn
to *answer*, not to review.

Both operators need credentials, so an anonymous session shows all open changes for either
setting of the toggle.

Inside tuicr, review as usual (`j`/`k` to move, `c` to comment, `v` for a range comment), then
`:submit` to push:

- **Comment** — posts your inline comments and your review-level comment as the change message.
- **Approve** — the same, plus a `Code-Review +2` vote.
- **Request changes** — the same, plus a `Code-Review -1` vote. (`-2` is a hard veto in Gerrit,
  stronger than what "request changes" means on the other forges, so tuicr does not cast it.)
- **Draft** — stores every inline comment as a Gerrit *draft* comment, and the review-level
  comment as a draft on Gerrit's `/PATCHSET_LEVEL` pseudo-file. Nothing is published until you
  hit **Reply** in the Gerrit web UI.

Publishing a review passes `drafts: KEEP`, so drafts you left in the Gerrit web UI are not swept
up by a tuicr submit.

Existing comments are fetched when you open the change and render inline on their lines. Comments
left on an earlier patch set still show, marked outdated, matching the Gerrit web UI. Comments on
Gerrit's magic paths (`/COMMIT_MSG`, `/MERGE_LIST`, `/PATCHSET_LEVEL`) have no line in the file
diff and are not rendered.

## Relation chains

A feature on Gerrit often arrives as a *relation chain*: several changes stacked on each other,
one commit each. Without a flag, `tuicr pr 3965` opens change 3965 on its own, as for any change.
Add `--chain` to open the whole open chain it belongs to instead:

```bash
tuicr pr 3965 --chain
```

- The diff runs from the parent of the bottom change to the top change, whichever change in the
  chain you named.
- The inline commit selector lists one commit per change, oldest at the bottom, each labelled
  with its change number. Select one to read exactly that change's diff, or a run of them.
- The chain comes from Gerrit's `related` endpoint and is rebuilt from parent links. Merged and
  abandoned changes are left out, so a merged bottom change becomes the chain's base. A change
  *above* that was built on an older patch set of the change below it is not in the history being
  reviewed, so it ends the chain there. A change *below* that sits on an older patch set stays in,
  since it is in that history, and its row in the selector says which patch set it is
  (`(patch set 1 of 2)`).
- Where two open changes share a parent, the chain stops below the fork.
- A change that is not in an open chain opens on its own, with a warning saying so, and so does
  one whose chain could not be read (the warning then says why). `--chain` is ignored, with a
  warning, on every other forge, and it is not available from the Pull Requests tab yet.
- `:e` reloads a chain view as a chain. If the chain cannot be read, the reload fails and the
  chain view stays as it was. If the chain is gone (the other changes were merged or abandoned),
  the change reopens on its own, in its own session.

**A chain view is read-only for now.** Each change in a chain is reviewed and voted on
separately, and tuicr does not yet route each comment to the change that owns its commit, so
`:submit` is refused there. You can still read the chain and leave local comments, which stay
in the session. To submit, open the change on its own (without `--chain`), or use the Gerrit
web UI.

The existing comments of the change you opened are all shown as outdated in a chain view (they
are hidden by default; `:comments all` shows them). Their line numbers belong to that change's own
patch set, while the chain's diff spans more commits, so a line can land on the wrong text.

A chain view is saved as a session of the change you named, keyed by the top change's revision,
and kept apart from the change's lone view: local comments and reviewed marks made in one never
show up in the other. When a new patch set arrives, tuicr usually starts its session from the
review of the previous one. It looks only at the review you saved last for that change, though, and
skips it when that was the other view, so after switching between the two, the next patch set
starts fresh. The earlier review is still on disk, under its own patch set.

## The local-clone requirement

Gerrit's `/revisions/{id}/patch` endpoint returns a patch ordered by git's rename-aware diff
queue, while its file list is keyed and sorted by path — the two cannot be paired positionally the
way tuicr's `pair_metadata_with_patch` requires, and tuicr does not read file paths out of patch
text. So the diff is built from a local clone with `git diff <base>..<head>`, which yields
authoritative `git diff --raw` metadata. Azure DevOps takes the same route.

A Gerrit change lives in `refs/changes/*`, which a normal clone does not carry, so tuicr fetches
the change's patch-set ref first (for a relation chain, the top change's ref, which carries every
change below it). That is a bare `git fetch <remote> refs/changes/NN/CCCC/P`: it writes
`FETCH_HEAD` only, creating no branch and moving no existing ref in your repo.

File content for context expansion is read from the local clone first, falling back to the REST
API when a revision isn't present locally.

## MVP limitations

This is the first Gerrit slice. Not yet supported:

- **Patch-set comparison.** tuicr reviews the current patch set against its parent commit. There
  is no "diff patch set 2 against 4" selector — the inline commit selector lists the changes of a
  relation chain, not the patch sets of one change.
- **Submitting from a relation chain.** A chain opens read-only; see
  [Relation chains](#relation-chains).
- **The `Code-Review` label is assumed.** Installs that rename it, or that need a different label
  (`Verified`, a custom gate), cannot vote from tuicr yet.
- **Replying to a thread.** Existing comments are read-only in tuicr, as on every other forge.
- **SSH transport.** Only the REST API is used; `ssh -p 29418 host gerrit review` is not called.
- **Gerrit 3.3+ for the `r` toggle.** The attention set was introduced in Gerrit 3.3; an older
  server rejects `attention:self` outright. The rest of the backend has no such floor.
- **Draft submits are not atomic.** Gerrit's `Create Draft` endpoint takes one comment per
  request, and there is no batch equivalent, so saving a draft review is one `PUT` per comment. If
  the connection drops part-way, the drafts before that point are already on the server while
  tuicr still holds every comment locally — nothing is lost, and the error names how many landed.
  Submitting again re-posts those, so delete them in Gerrit's web UI first, or publish from there.
  Drafts are private to you until published.

Publishing a review is a single request and needs no such care: it carries
`omit_duplicate_comments`, so if the response is lost after Gerrit committed it, submitting again
will not post the inline comments twice. A second review *message* can still appear that way.

## Troubleshooting

- **"Gerrit needs authentication for this request."** — set `GERRIT_USERNAME` and
  `GERRIT_PASSWORD`.
- **"Gerrit rejected the credentials…"** — `GERRIT_PASSWORD` must be the HTTP password from
  *Settings → HTTP Credentials*, not your account password.
- **"refusing to send GERRIT_USERNAME/GERRIT_PASSWORD over plaintext HTTP…"** — `GERRIT_URL` is an
  `http://` URL. Point it at HTTPS, or unset the two credential variables to browse anonymously.
- **The Pull Requests tab says there is no supported remote** — your Gerrit host shows none of the
  three signals above. Set `GERRIT_URL` to the server root.
- **Requests go to the wrong host** — the error names the server actually contacted. If that is
  your SSH gateway rather than the web host, set `GERRIT_URL`; if it is a *different* Gerrit than
  the one you meant, you have a stale `GERRIT_URL` exported in that shell.
- **`Unsupported operator attention:self` after pressing `r`** — your Gerrit predates 3.3 and has
  no attention set. The unfiltered list still works; toggle back with `r`.
- **"Could not find <base>..<head> in the local checkout."** — the patch-set fetch failed. Run
  `git fetch origin refs/changes/…` (the ref is named in the error) by hand and retry; that
  usually surfaces the real credential or network problem.
- **`applying label Code-Review: +2 is restricted`** — your account cannot cast `+2` on this
  project. Use Comment instead, and vote from the web UI.
