---
name: rectifier
description: Review code changes with rectifier, imbik's code review agent, through the `rectifier` CLI. Use it for a review of a branch, of commits or of uncommitted changes before a pull request. Also use it for the state or the findings of a rectifier review, to keep reviewing while the user edits, and to go through the findings of the account's Deep Scans.
---

# rectifier

`rectifier` sends changes in a git checkout to rectifier's review service and prints the review as markdown. The
review runs on the service. Nothing needs pushing, and no pull request is needed.

## Set up

1. Make sure that the CLI is installed: `rectifier --version`. If it is missing, install it with
   `uv tool install imbik-rectifier`, or `pipx install imbik-rectifier`. It needs Python 3.10 or later, and git.
2. Make sure that the CLI is signed in: `rectifier whoami`. If it is not, run `rectifier login`. It opens the
   user's browser on rectifier's sign-in page and waits up to 10 minutes. Tell the user to check that the code and
   the host name on the page match the terminal, then to approve. With no browser on this machine, run
   `rectifier login --no-browser`, and give the user the link it prints.
3. If a command says that the CLI is too old, upgrade it with `uv tool upgrade imbik-rectifier`, or
   `pipx upgrade imbik-rectifier`, and then run `rectifier skill install` again to update this file.

The CLI needs the network, and `rectifier login` writes its token under the user's home folder. If your commands run
in a sandbox that blocks the network, such as the default sandbox of Codex or Cursor, ask the user to approve running
`rectifier` outside the sandbox.

Never ask the user to paste a token into the chat. If `RECTIFIER_CLI_TOKEN` is set, the CLI uses it, and
`rectifier login` is not needed.

## Review a change

| Command | Reviews |
| --- | --- |
| `rectifier review` | The branch since it left the default branch |
| `rectifier review --last 3` | The last 3 commits, as one change |
| `rectifier review abc123..def456` | A range, as one change |
| `rectifier review --uncommitted` | Staged, unstaged and untracked changes, against HEAD |
| `rectifier review --uncommitted --base origin/main` | The uncommitted changes and the commits since `origin/main` |

- Use a full review unless the user asks for a light one. `--full` asks for a full review, and `--lite` for a light
  one. A line on stderr says which kind runs, such as `starting full review`.
- A review takes several minutes. Run it in the background, or run it with `--no-wait`. `--no-wait` prints the
  review's job and exits. Then `rectifier status <job> --wait` waits for the end and prints the review.
- `rectifier review` reads the whole branch every time. After you commit fixes, review only the new commits with
  `--last N`.
- If the output says that no new review ran, these commits were reviewed in the last hour, and the output shows
  that review. Do not add `--force` unless the user asks.
- `--uncommitted` sends nothing the first time. It lists the files that it sends, prints the same command with
  `--confirm <code>`, and exits with code 3. Show the list to the user. If the user agrees and the list holds no
  secrets, run the command with the code.
- If the list holds files that must not be sent, or files that are not part of the change, leave them out with
  `--path`, a git pathspec: `--path src/` sends only that folder, and `--path ':!scratch/'` leaves one out. Give it
  again for more. Run the command again, and show the user the new list.
- Do not use `--post` unless the user asks. It posts the review on the branch's open pull request.
- `-C <dir>` reviews the checkout in `<dir>`. `--json` prints JSON instead of markdown.

### Exit codes

| Exit code | Meaning |
| --- | --- |
| 0 | Nothing blocks |
| 1 | A finding blocks |
| 2 | The review did not run or did not complete. The error is on stderr. |
| 3 | The command waits for the user to agree to the files it sends (`--confirm`) |
| 4 | `rectifier status` without `--wait` only: the review has not ended yet |
| 130 | The command was stopped with Ctrl-C. A review it was waiting for continues. |

### Act on the findings

The review prints each finding with its severity, its file and line, and a "Message to fix with an agent". It also
saves the review under `.rectifier/reviews/` in the checkout, which git ignores.

- Check each finding against the current code before you change anything.
- If the finding is real, follow its message to fix it.
- If the finding is wrong or not worth fixing, change nothing, and tell the user why.

## Look up reviews

- `rectifier status <job>` says where a review is: waiting, running and how far, or ended. After the end, the command
  prints the review. Rectifier keeps a result for an hour after the review ends.
- `rectifier reviews` lists the account's reviews, with the newest first. `--repo org/name`, `--pr N` and
  `--status` narrow the list. `--json` prints JSON.
- `rectifier report --pr N` prints the findings that Rectifier posted on pull request N of this repository, laid out
  as a `rectifier review` report but without the impact or the message for an agent. It reads the newest review of the
  pull request that completed. Use it when the user asks you to fix Rectifier's findings on a pull or merge request.
  `--repo org/name` names another repository, and `--json` prints JSON. Exit code 1 means a finding blocks.
  Reading a review's findings is asked again a few times after a 429, 502, 503, 504 or dropped connection; when reading
  many reviews, read them one at a time rather than in parallel.
- Once you have checked a finding of `rectifier report`, or the user tells you what it is, record it with
  `rectifier verdict <review> <finding> real|not-real|out-of-scope|already-known [--note "..."]`. `<review>` is the
  review's id, which the report's last line names (`review.id` in the JSON), and `<finding>` the finding's id
  (`id` in the JSON). `out-of-scope` is a real defect in code or of a kind the user does not want reviewed; the other
  labels mean what they mean for `scan-verdict` below. A later answer on the same finding replaces the earlier one.
  A finding of a review whose findings Rectifier no longer keeps cannot be answered: exit code 2.

## Go through Deep Scan findings

A Deep Scan reads a whole repository, not a change. Use these commands when the user asks you to go through or fix
the findings of the account's Deep Scans.

- `rectifier scans` lists the account's finished Deep Scans, with the newest first: the repository, when the scan
  finished, its findings by severity, the commit it read, and the scan's id. `--json` prints JSON.
- `rectifier scan-report <id>` prints the findings of one scan: severity, title, file and lines, what it is, why it is
  a defect, and a link to the code.
- `rectifier scan-report --all --json` prints the findings of the newest finished scan of each repository, as
  `{"scans": [{"scan", "report"}], "skipped"}`. Each `report` has the commit the scan read (`sha`) and its
  `findings`. A non-empty `scanned` (the paths read of a large repository), `left_out` (files not read as generated,
  vendored, built or docs) or a `marks_read` of false means the scan did not read the whole repository.
- The scan read the code at its commit. Before you change anything, check each finding against the current code.
- Once you have checked a finding, or the user tells you what it is, record it with
  `rectifier scan-verdict <scan> <finding> real|not-real|wont-fix|already-known [--note "..."]`. `<finding>` is the
  finding's id, which `scan-report` prints (`id` in the JSON). `wont-fix` is a real defect the user will not fix (of
  low value to them, in test or dead code, or of a kind they do not care about); `already-known` is a real defect they
  already knew of. A later answer on the same finding replaces the earlier one. A finding of a report that expired
  or was deleted cannot be answered: exit code 2. A CLI older than 0.6.9 refuses `wont-fix`; upgrade it (step 3 of
  Set up), or send `out-of-scope`, which the service stores as `wont-fix`.
- Rectifier deletes a scan's report after 30 days unless the account's plan keeps reports. An expired or deleted
  report cannot be read: `rectifier scan-report <id>` exits 2. `--all` does not read a report that had expired when
  it listed the scans: it reads the newest older scan of that repository whose report is kept, if there is one.
  `skipped` holds only a report that expired or was deleted after the list was read. A 429, 502, 503, 504 or
  dropped connection is asked again a few times first; any other failure, or one that lasts, stops `--all` with exit
  code 2 and no output.

## Review while you edit: `rectifier review --watch`

`--watch` reviews your branch and your uncommitted changes again each time you stop editing for 30 s. It prints each
finding as one line on stdout, so you can act on it while you work. Use it for a task that takes more than a few
minutes. For one review of a finished change, use `rectifier review` without `--watch`.

### Start the watch

1. Run the watch once in a shell, in the checkout:
   `rectifier review --watch -C <checkout> --base origin/main`
   It lists the files it will send, prints a code, and exits with code 3.
2. Read the list. Make sure that it holds only your own changes, with no secrets and no files that must not leave your
   machine. Look for untracked links and caches, such as a `.venv` linked into the checkout. Add them to
   `.git/info/exclude`, then run step 1 again.
3. Start the watch with the code:
   `rectifier review --watch -C <checkout> --base origin/main --confirm <CODE>`
   The code stays valid for this checkout and base. If the watch stops, run the same command again. It remembers what
   it printed and does not print it again.
4. Make sure that a `watching <checkout> against <base>` line comes within a few seconds. If it does not come, the
   output does not reach you. See "Claude Code" below.

Start the watch before the first edit, in the checkout where the edits happen. If the work is committed and pushed to
a pull request before the watch starts, the watch can leave that commit to the pull request's review and start no
review for it (`review covered by #<n>`).

`--watch` is open only to some accounts. If a line says that `--watch` is not open for this account, stop the watch.
Then, when the task is done, run `rectifier review` for one review of the change.

Use one watch for each checkout, and one branch for each checkout. A second watch in the same checkout exits with code
2. Do not switch branches in a watched checkout. Stop the watch when you finish the task or leave the checkout.

### Read the lines

Each line starts with a word that tells you what it is. `<job>` is the review's id.

- `suspected <job>/<id> <severity> <file>:<line> <title>`: an early finding that the review has not checked yet.
  These lines come minutes before the review ends.
- `finding <job>/<n> <severity> <file>:<line> <title> (details: ...)`: a final finding of the review.
- `confirmed <job>/<id> -> <job>/<n>`: the review confirmed the suspected line as that final finding.
- `withdrawn <job>/<id>`: the review did not confirm the suspected line.
- `unchecked <job>/<id>`: the review gave no verdict on the suspected line. If it ends with `(stopped for a newer
  checkpoint)`, a newer review of your later code replaced this one.
- `review <job> done: N new findings, ...`: the review ended. `read since <sha>` means that it read only what changed
  since an earlier review. `read the whole branch` means that it read all of the branch.
- `review <job>: full review` or `review <job>: light review`: the kind of review that runs. This line is on stderr.
- `review covered by #<n>: ...`: you pushed a commit that a pull request review already reads. The watch starts no
  review for it. Read that commit's findings on the pull request.
- `review not sent: ...` or `review <job> failed: ...`: no result came for this checkpoint. Your next change sends a
  new review.

A line can end with `(read <sha>, N commits behind HEAD)`. This means that the review read older code than your
checkout holds now. A line can also end with `(these lines changed since <sha>)`. This means that the code near the
finding changed after the review read it. It does not mean that the finding is fixed. Often the lines changed for a
different reason, or the fix is not complete. Read the code again before you skip the finding.

### Act on the lines

- Act on a `suspected` line when it comes. Do not wait for `confirmed`. Most suspected lines are confirmed, and the
  review can take several more minutes to end.
- If a later `withdrawn` line names a suspected line that you already fixed, keep the fix if the problem was real.
- Before you fix a finding, check it against your current code. Your code can have changed since the review read it.
- Each review reads one checkpoint. A finding can be true for an older checkpoint and gone in your current code.
- A review takes some minutes. Keep working while it runs. A new change starts a new review, and it can replace a
  review that has run less than 3 min.

### Claude Code

Follow the watch with the Monitor tool, not with a Bash command in the background. Monitor gives you each line as an
event when it prints.

- Command: `rectifier review --watch -C <checkout> --base origin/main --confirm <CODE> 2>&1`
- `timeout_ms`: 600000 (10 min). A headless run (`claude -p`) allows no more. An interactive session allows up to 1800000.
- When the Monitor ends at its timeout, start it again with the same command. Do this every time, for as long as the
  task goes on. The watch goes on where it stopped.
- To filter the lines, use `/usr/bin/grep --line-buffered`, for example
  `| /usr/bin/grep --line-buffered -v "^review .*: sent"`. Do not use a bare `grep`. In the Claude Code shell, `grep`
  can be a wrapper that holds each line back until the next line comes. Then a line can reach you minutes late, or
  only when the watch stops.
- Stop the watch with TaskStop on the Monitor.

### Codex, Cursor and other agents

If your agent cannot get lines as events, start the watch in the background and send its output to a file outside the
checkout:

`rectifier review --watch -C <checkout> --base origin/main --confirm <CODE> > "${TMPDIR:-/tmp}/rectifier-watch.log" 2>&1 &`

Read the new lines of that file after each edit and before you say that a task is done, for example with `tail -n 40`.
Make sure that the file has a `watching` line. If the process ends, start it again with the same command. Stop it by
its process id when you finish.
