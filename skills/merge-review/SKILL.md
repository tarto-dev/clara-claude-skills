---
name: merge-review
description: Reviews an existing GitLab merge request end-to-end — content/correctness, form, language, and coding standards — loads the linked Mantis ticket, and drafts a structured review note plus an approve / request-changes recommendation. Drafts only by default; posts the note and approves only on explicit user go-ahead. Tuned for Drupal/PHP + GitLab. Use when the user gives an MR (URL or number) to review as a reviewer, or invokes /clara:merge-review.
---

# Clara — Merge Review

Reviewer-side review of someone's merge request. Produce a review another human could post as-is: a verdict, a per-axis table, and concrete comments. **Outward-facing** — draft everything by default; post the note and approve **only** on explicit user go-ahead, never on your own initiative.

## Load the MR

From the URL or number, pull everything with `glab` (the `-R` and host come from `https://<host>/<group/project>/-/merge_requests/<id>`):

```bash
GITLAB_HOST=<host> glab mr view <id> -R <group/project>      # title, description, author, branches, approvals
GITLAB_HOST=<host> glab mr diff <id> -R <group/project>      # the diff under review
GITLAB_HOST=<host> glab api projects/<id_or_path>/merge_requests/<id>/commits   # commit list (for form/langue checks)
```

Read the changed lines **and** enough surrounding code to judge them. Never review a hunk in isolation.

## Load the ticket

Judge content against intent. Extract the Mantis id from the MR description or the branch name (`fix/<id>-slug`), then:

```bash
scripts/mantis-issue.sh <id>    # summary, description, steps-to-reproduce, notes
```

Needs `MANTIS_URL` + `MANTIS_TOKEN` in the env. If unset or it exits non-zero (2 = no creds, 3 = API error), ask the user to paste the ticket rather than reviewing intent blind.

## Review axes

Apply the **`clara:review` dimensions** — Correctness, Security, Performance & cacheability, Standards, Tests — and the **Blocker/Major/Minor/Nit** severity scale from that skill. Run the project's linters when present (via `lando`/`ddev`/`docker compose exec`). Below are the axes specific to reviewing someone else's MR:

- **Contenu** — Does the diff deliver what the ticket asks, end to end? Root cause vs symptom. This is `clara:review` Correctness, judged against the loaded ticket.
- **Forme** — Commit & MR hygiene:
  - **Atomic commits** — one logical change per commit; flag mixed-concern commits.
  - **Conventional commits** — `type(scope: #ticket): summary`.
  - **Ticket number required in every commit.** If missing, the author can add the Mantis ref as an MR comment; with no Mantis attached at all it's tolerated but should be rare — flag it either way.
  - MR title convention, real ticket link, description complete (Contexte / Cause racine / Correctif / Vérification), branch `fix|feat/<id>-slug`, staging scoped (no stray files).
- **Langue** — Two registers, never mixed:
  - **French** — the MR title/description and your review note (team language).
  - **English** — all code: identifiers, code comments, docblocks, and (Drupal) user-facing strings through `t()`. Non-English code or comments → raise a comment tagged **« à revoir »** at **Minor** — *non-blocking for the verdict*; the "threads must be closed before merge" rule guarantees it gets fixed.
  - Typos / grammar in either register → comment.
- **Standards** — `clara:review` Standards (match the file's local idiom and the `Drupal,DrupalPractice` standard; run `phpcs` / `php -l` rather than eyeballing), **plus the house rule: every function is typed (params + return) and carries a docblock (HEREDOC)** — flag any function missing either.

## Verdict

Mechanical, from the highest severity found:

- Any **Blocker/Major** → **request-changes**.
- Only **Minor/Nit**, or none → **approve** (note nits in the comment).

Language/style remarks tagged **« à revoir »** are Minor *by policy*: each one opens a thread (which must be closed before merge can proceed), so it doesn't flip the verdict to request-changes on its own. Reserve request-changes for real Contenu/Sécurité/Standards defects.

Never recommend approve on anything you couldn't verify — say what's unverified and default to request-changes.

## Draft the review note

Build a single structured note, **in the MR's language** (French here), ready to post verbatim:

```markdown
## Revue — <approve | demande de modifs>

| Axe | Verdict |
|---|---|
| Contenu | ✅ / ⚠️ / ❌ + une ligne |
| Forme | … |
| Langue | … |
| Standards | … |
| Tests | … |

### Remarques
- **[Blocker]** <symptôme> (`<fichier>:<ligne>`)
  <pourquoi, et le correctif exact — copiable, jamais « investiguer X »>
- **[à revoir]** Commentaire en français (`<fichier>:<ligne>`) — le code et les commentaires sont en anglais. Non bloquant, mais le thread doit être résolu avant merge.
- **[Nit]** …

### Vérifié
- ✅ <check empirique qui a tourné>
- ✅ phpcs / php -l / tests le cas échéant
```

Show this note to the user. **Do not post it.** State the recommended verdict and what's unverified.

## Post & approve — only on explicit go-ahead

When (and only when) the user explicitly validates, act on the platform:

```bash
# 1. post the review note
GITLAB_HOST=<host> glab mr note <id> -R <group/project> -m "$NOTE"

# 2a. approve  — only if the verdict is approve AND the user said so
GITLAB_HOST=<host> glab mr approve <id> -R <group/project>

# 2b. request-changes — post the note and leave it unapproved (GitLab has no "request changes" verb)
GITLAB_HOST=<host> glab mr revoke  <id> -R <group/project>   # revoke a prior approval if one exists
```

Confirm what you posted (note URL, approval state). Never merge, never mark Ready.

## Non-goals

- Draft by default. Posting the note and approving each require explicit user go-ahead — approval in one review doesn't carry to the next.
- Never approve what you couldn't verify, and never on Blocker/Major findings.
- Never merge or mark the MR Ready.
- Don't flag pre-existing code the MR didn't touch — note it as out-of-scope instead.
