# Git & Version Control

> **18 questions** — 10 theory, 6 practical, 2 experience

- Git internals: three-tree architecture, branches as pointers, commits as a DAG, SHA-1 hashing
- Rebase vs merge: history readability, conflict effort, bisect usability, shared branch safety
- Interactive rebase: squash, reword, reorder, edit, drop commits
- Cherry-picking: moving commits between branches, ranges, -x flag, risks of duplicate commits
- Bisect: finding regressions via binary search, manual and automated workflows
- Reflog: recovering from reset --hard, failed rebase, dropped stashes, expiry
- Branching models: trunk-based development, GitFlow, GitHub Flow — team size, CI maturity, tagging and release workflows
- PR merge strategies: squash-merge vs merge-commit vs rebase-merge tradeoffs
- Force-push: dangers on shared branches, --force-with-lease, branch protection
- Merge conflict resolution: conflict markers, --ours/--theirs, rerere, minimizing conflicts with small PRs and frequent rebasing
- Monorepo workflows: sparse checkout, shallow/partial clone, CODEOWNERS, path-based CI filtering — and why submodules are rarely the answer

---

## Foundational

<details>
<summary>1. How does Git store data internally — what is the three-tree architecture (working directory, staging area, repository), how are commits structured as a directed acyclic graph, what does it mean that branches are just pointers to commits, and why does Git use SHA-1 hashing for content addressing?</summary>

**Three-tree architecture:**

Git manages three "trees" (snapshots of file state):

1. **Working directory** — the actual files on disk you edit. This is what you see in your editor.
2. **Staging area (index)** — a file (`.git/index`) that holds the snapshot you're building for the next commit. `git add` moves changes from working directory to the index. This lets you craft commits selectively — you don't have to commit everything you've changed.
3. **Repository (`.git/objects/`)** — the permanent object database. `git commit` takes the index snapshot and writes it as a commit object here.

`git status` compares these three trees: working dir vs index (unstaged changes), index vs HEAD commit (staged changes).

**Git's object model:**

Git stores four types of objects:

- **Blob** — file contents (no filename, just content)
- **Tree** — a directory listing, mapping filenames to blob SHAs (and subtrees)
- **Commit** — points to a tree (the project snapshot), plus metadata: author, message, timestamp, and parent commit(s). (Annotated tags are a fourth object type — named pointers to commits.)

**Commits as a DAG:**

Each commit points to one or more parent commits (zero for the initial commit, two for a merge commit). This forms a directed acyclic graph — directed because edges point from child to parent, acyclic because you can never follow parent links and arrive back at the same commit. The DAG is what enables branching, merging, and history traversal.

**Branches are just pointers:**

A branch is a 41-byte file in `.git/refs/heads/` containing one SHA — the commit it points to. Creating a branch is nearly free (no copying). When you commit, Git advances the current branch pointer to the new commit. `HEAD` is a pointer to the current branch (a "symbolic ref"), which is how Git knows which branch to advance.

```bash
cat .git/refs/heads/main
# e.g., a1b2c3d4e5f6...

cat .git/HEAD
# ref: refs/heads/main
```

**Why SHA-1 content addressing:**

Every object is stored and referenced by the SHA-1 hash of its content. This gives Git:

- **Integrity** — if any bit changes, the hash changes, so corruption is immediately detectable
- **Deduplication** — identical content produces the same hash, so Git stores it once regardless of how many files or commits reference it
- **Efficient comparison** — comparing two trees means comparing hashes, not file contents

Git is transitioning to SHA-256 for stronger collision resistance, but the content-addressing model stays the same.

</details>

<details>
<summary>2. Why would you choose rebase over merge (or vice versa) — how does each strategy affect history readability, conflict resolution effort, the usefulness of git bisect, and what are the dangers of rebasing commits that have already been pushed to a shared branch?</summary>

**Merge** creates a merge commit that joins two branches. It preserves the exact history of when work happened in parallel.

**Rebase** replays your commits on top of the target branch, creating new commits with new SHAs. It produces a linear history as if the work happened sequentially.

**Comparison:**

| Aspect | Merge | Rebase |
|---|---|---|
| **History readability** | Retains branch topology — useful for seeing when parallel work happened, but can produce a tangled graph | Linear, clean history — easier to scan and understand |
| **Conflict resolution** | Resolve once in the merge commit | Resolve per replayed commit — more effort if many commits conflict, but each conflict is smaller and more focused |
| **git bisect** | Merge commits can cause bisect to walk down a side branch, making it harder to isolate regressions | Linear history makes bisect straightforward — every commit is on the main line |
| **Reversibility** | Safe — original commits unchanged, merge commit is additive | Destructive — rewrites commit SHAs, original commits are gone (recoverable via reflog for 90 days) |

**The golden rule of rebase:**

Never rebase commits that have been pushed to a shared branch that others are working on. Rebase rewrites SHAs, so anyone who based work on the old commits now has divergent history. They'll see confusing duplicate commits or conflicts when they pull.

**Practical recommendation:**

- **Rebase** your local feature branch onto `main` before merging — keeps history clean
- **Merge** when combining long-lived branches or when the parallel development history is valuable
- For most teams: rebase locally, merge (or squash-merge) the PR into `main`

</details>

<details>
<summary>3. Why do different teams need different branching models — compare trunk-based development, GitFlow, and GitHub Flow in terms of team size, CI maturity, release cadence, and risk tolerance, and explain what signals tell you a team has outgrown its current model?</summary>

**Trunk-based development:**

Everyone commits to `main` (or very short-lived branches that merge within hours). Releases are cut from `main` via tags or release branches.

- **Best for:** Teams with strong CI/CD, automated testing, feature flags, and continuous deployment
- **Team size:** Works at any scale (Google, Meta use this), but requires CI maturity
- **Release cadence:** Continuous — deploy multiple times per day
- **Risk tolerance:** Low risk per change (small, frequent merges), but requires confidence in your test suite and rollback capability

**GitHub Flow:**

Branch off `main`, open a PR, review, merge back to `main`. Simple — one long-lived branch plus short-lived feature branches.

- **Best for:** Teams deploying from `main` with moderate CI maturity
- **Team size:** Small to mid-size (2-20 developers)
- **Release cadence:** On every merge to `main`, or on a schedule
- **Risk tolerance:** Moderate — PR reviews catch issues, but no staging branch

**GitFlow:**

Long-lived `main` and `develop` branches, plus `feature/*`, `release/*`, and `hotfix/*` branches. Strict promotion path: feature -> develop -> release -> main.

- **Best for:** Teams with formal release cycles, multiple versions in production, or compliance requirements
- **Team size:** Larger teams with dedicated release management
- **Release cadence:** Scheduled (weekly, monthly, quarterly)
- **Risk tolerance:** Very conservative — multiple gates before code reaches production

**Signals a team has outgrown its model:**

- **GitFlow feels too heavy:** Merge conflicts between `develop` and `main` are constant, release branches live for weeks, hotfixes are painful to back-port. Move to GitHub Flow or trunk-based.
- **GitHub Flow is too loose:** No clear release process, unclear what's deployed, broken `main` blocks everyone. Either add more CI discipline (move toward trunk-based) or add release branches (move toward GitFlow).
- **Trunk-based without the infrastructure:** Broken `main` is frequent, no feature flags so half-done features leak to users, no automated rollback. The model isn't wrong — the tooling isn't ready. Invest in CI/CD before adopting trunk-based.
- **Long-lived feature branches:** Any model where branches live more than a few days signals integration problems. The fix is always smaller, more frequent merges — not a different branching model.

</details>

## Conceptual Depth

<details>
<summary>4. When and why would you cherry-pick commits instead of merging a branch — what problems does cherry-picking solve, what does the -x flag do and why should you use it, how do you cherry-pick a range of commits, and what are the risks of duplicate commits appearing in your history?</summary>

**When to cherry-pick:**

Cherry-pick applies a specific commit's changes to your current branch without merging the entire source branch. Use it when:

- **Hotfixing:** A bug fix landed on `develop` but you need it on `main` or a release branch immediately, without pulling in all other `develop` changes
- **Backporting:** Applying a fix to an older release branch that will never be merged forward
- **Selective extraction:** A feature branch has one useful commit mixed with work-in-progress you don't want yet

**The `-x` flag:**

```bash
git cherry-pick -x abc1234
```

This appends `(cherry picked from commit abc1234)` to the commit message. Always use it — it creates a traceable link between the original and the cherry-picked commit. Without it, there's no record that these two commits are related, which causes confusion during code review and when investigating history.

**Cherry-picking a range:**

```bash
# Cherry-pick commits A through D (A is exclusive, D is inclusive)
git cherry-pick A..D

# Cherry-pick commits A through D, including A
git cherry-pick A^..D
```

The `A..D` syntax is exclusive of A, which trips people up. Use `A^..D` to include A itself.

**Risks of duplicate commits:**

Cherry-picking creates a new commit with a different SHA but identical changes. If you later merge the source branch into the target branch, Git sees two commits that do the same thing. Usually Git handles this gracefully (the changes are already applied, so the merge is a no-op for those lines), but it can cause:

- **Confusing history** — `git log` shows the same change twice with different SHAs
- **Merge conflicts** — if the surrounding code changed between the original and the cherry-pick, Git may conflict on the merge
- **Noise in blame** — the cherry-picked commit shows up as a separate change

**Best practice:** Cherry-pick for hotfixes and backports where the branches won't be merged later. If the branches will eventually merge, prefer a proper merge or rebase instead.

</details>

<details>
<summary>5. How does git bisect find regressions using binary search — why is this approach faster than scanning commits linearly, what makes a codebase bisect-friendly (and what breaks it), and when would you use manual bisect vs automated bisect with a test script?</summary>

**How bisect works:**

You give Git a "good" commit (where the bug didn't exist) and a "bad" commit (where it does). Git checks out the commit halfway between them and asks you to test. Based on your answer (good or bad), it eliminates half the remaining commits and repeats.

For 1,000 commits between good and bad, linear scanning checks up to 1,000 commits. Binary search checks ~10 (`log2(1000)`). This is dramatically faster for large ranges.

```bash
git bisect start
git bisect bad              # current commit has the bug
git bisect good v2.1.0      # this tag was known-good
# Git checks out the middle commit
# You test, then:
git bisect good             # or git bisect bad
# Repeat until Git identifies the first bad commit
git bisect reset            # return to your original branch
```

**What makes a codebase bisect-friendly:**

- **Every commit compiles and passes tests** — if intermediate commits are broken, you can't test them and must `git bisect skip`, which reduces efficiency
- **Atomic commits** — each commit is a single logical change, so when bisect finds the bad commit, you can understand what it changed
- **No squashed mega-commits** — a 500-line squash commit that bisect lands on tells you nothing useful

**What breaks bisect:**

- Commits that don't build (WIP commits, broken merges)
- Commits that require manual setup changes (DB migrations, env variable changes) without documentation
- Non-deterministic tests — a flaky test makes bisect give wrong answers

**Manual vs automated bisect:**

```bash
# Automated bisect with a test script
git bisect start HEAD v2.1.0
git bisect run npm test -- --grep "user login"
```

Use **manual bisect** when the regression is visual, involves user interaction, or doesn't have a reliable automated test. Use **automated bisect** (`git bisect run`) when you can write a script or test command that exits 0 for good and non-zero for bad — this is faster and eliminates human error.

**Gotcha with `bisect run`:** The script must exit with code 125 to signal "skip this commit" (can't test, e.g., doesn't compile). Exit 0 = good, exit 1-124/126-127 = bad, exit 128+ = abort bisect.

</details>

<details>
<summary>6. What does the reflog track that the regular commit log doesn't — what types of operations does it record, why is it your safety net for recovering from destructive operations like reset --hard and failed rebases, and when does the reflog expire so you can no longer recover?</summary>

**What the reflog tracks:**

`git log` shows the commit ancestry chain — the DAG of commits reachable from a ref. The **reflog** records every time a ref (branch pointer or HEAD) changes, including operations that aren't part of the commit history:

- `commit` — normal commits
- `reset` — including `reset --hard` that "loses" commits
- `rebase` — every step of a rebase, including the original position before rebase started
- `checkout` / `switch` — branch switches
- `merge` — merge operations
- `pull` — fetch + merge/rebase
- `stash` — stash and stash drop
- `amend` — commit amends (the pre-amend commit is recorded)

```bash
git reflog
# e.g.:
# a1b2c3d HEAD@{0}: reset: moving to HEAD~3
# f4e5d6c HEAD@{1}: commit: add user validation
# 7890abc HEAD@{2}: rebase (finish): returning to refs/heads/feature
```

**Why it's your safety net:**

When you `reset --hard` or a rebase goes wrong, the commits aren't deleted — they're just unreachable from any branch. The reflog still points to them. You can find the SHA of the commit you want and `git reset` or `git checkout` back to it. Nothing is truly lost until the reflog entry expires and garbage collection runs.

**Expiration:**

- **Reachable entries** (commits still on a branch): expire after **90 days** by default
- **Unreachable entries** (orphaned commits): expire after **30 days** by default
- After expiry, `git gc` can actually delete the objects

These defaults are configurable:

```bash
git config gc.reflogExpire "90 days"
git config gc.reflogExpireUnreachable "30 days"
```

**Key limitation:** The reflog is local only. It's not pushed to remotes. If you clone a fresh copy, there's no reflog history. This is why having a remote as a backup matters — even if reflog is gone locally, the remote may still have the commits on a branch.

</details>

<details>
<summary>7. What are the tradeoffs between squash-merge, merge-commit, and rebase-merge as PR merge strategies — how does each affect commit history cleanliness, blame/bisect accuracy, the ability to revert a PR as a unit, and what strategy works best for different team sizes and workflows?</summary>

| Aspect | Squash-merge | Merge-commit | Rebase-merge |
|---|---|---|---|
| **What it does** | Combines all PR commits into one commit on `main` | Creates a merge commit joining the branch into `main`, preserving all individual commits | Replays PR commits onto `main` individually (new SHAs), no merge commit |
| **History** | Cleanest — one commit per PR | Preserves full branch history; merge commits make the graph non-linear | Linear history with all individual commits |
| **Blame accuracy** | Points to the squash commit — you lose which specific change within the PR caused an issue | Points to the exact original commit | Points to the exact commit (rebased copy) |
| **Bisect** | Bisect lands on the squash commit — tells you which PR, not which change | Bisect can walk into the branch and find the exact commit | Bisect works well — linear history, individual commits |
| **Revert** | Easy — `git revert <squash-sha>` undoes the entire PR in one command | Easy — `git revert -m 1 <merge-sha>` undoes the entire PR | Hard — you must revert each individual commit separately, in reverse order |
| **Author attribution** | Single commit may lose co-author information (unless Co-authored-by trailers are added) | Preserves all individual authors | Preserves individual authors |

**Recommendations by team context:**

- **Squash-merge** works best for most teams. Clean history, easy reverts, each commit on `main` maps to one PR. The tradeoff (losing granular blame) rarely matters in practice — if you need the detail, the PR link is in the commit message.
- **Merge-commit** is better when you need full commit granularity on `main` — useful for large PRs from long-lived branches or when detailed bisect matters more than a clean log.
- **Rebase-merge** gives you linear history with full commit detail, but losing single-command reverts is a meaningful cost. Best for teams that enforce clean, atomic commits on every PR (rare in practice).

**Practical default:** Squash-merge with a commit message template that includes the PR number. Override to merge-commit for large feature branches where individual commit history adds value.

</details>

<details>
<summary>8. Why is force-pushing dangerous on shared branches — what exactly happens to other developers' work when you force-push, how does --force-with-lease protect against overwriting someone else's commits, and what branch protection rules should you configure to prevent accidental force-pushes to main?</summary>

**What happens when you force-push:**

`git push --force` replaces the remote branch's history with your local history, regardless of what the remote currently contains. If another developer pushed commits after your last fetch, those commits are removed from the remote branch. The other developer's local copy still has them, but:

- Their next `git pull` will show a diverged history
- If they don't realize what happened, they might re-push (duplicating their work) or lose track of their commits entirely
- Anyone who based work on the now-removed commits has orphaned history

**`--force-with-lease` protection:**

```bash
git push --force-with-lease
```

This tells Git: "force-push, but only if the remote branch is at the SHA I last saw." If someone pushed new commits since your last fetch, the push is rejected. It's a compare-and-swap operation — safe force-push.

**When `--force-with-lease` still fails you:**

If you run `git fetch` before pushing, your local tracking ref gets updated. Now `--force-with-lease` sees the remote as "matching" even though you haven't reviewed the new commits. The protection only works if you haven't fetched since the other person pushed.

A stricter variant:

```bash
git push --force-with-lease=origin/feature:expected-sha
```

This explicitly checks against a specific SHA, regardless of what you've fetched.

**Branch protection rules to configure:**

On GitHub (similar on GitLab):

- **Require pull request reviews** — no direct pushes to `main`
- **Require status checks to pass** — CI must be green before merge
- **Do not allow force pushes** — explicitly blocks `--force` and `--force-with-lease` on protected branches
- **Do not allow deletions** — prevents accidental branch deletion
- **Require linear history** (optional) — enforces squash or rebase merges, prevents merge commits
- **Include administrators** — apply these rules even to repo admins, so no one can bypass them accidentally

**The combination that matters most:** "no force pushes" + "require PR reviews" on `main` and any release branches. This makes `--force-with-lease` a tool for feature branches only, where it belongs.

</details>

<details>
<summary>9. What are the key challenges of working in a monorepo with Git — why do large repositories cause performance problems, and how do sparse checkout, shallow clone, and partial clone each address different aspects of this problem? When would you use each one, and why are Git submodules rarely the right answer for monorepo workflows?</summary>

**Why large repos are slow:**

Git operations scale with repository size in several ways:

- **`git status` / `git diff`** scan every tracked file to detect changes — slow with millions of files
- **`git clone`** downloads the entire history of every file — huge for repos with years of history and many packages
- **`git checkout`** writes all tracked files to disk — slow if the working tree is massive
- **Index operations** — the `.git/index` file lists every tracked file; large indexes slow down everything

**Three solutions for different problems:**

| Solution | What it does | Problem it solves | When to use |
|---|---|---|---|
| **Sparse checkout** | Only checks out a subset of files to the working directory. The full history is still cloned — you just don't materialize files you don't need. | Slow `status`/`diff` due to massive working tree, developers who only work on one package | Developer workstations where you only touch a few packages |
| **Shallow clone** (`--depth N`) | Clones only the last N commits of history, omitting older history entirely | Slow clone due to massive history | CI/CD pipelines that only need to build the current state, not traverse history |
| **Partial clone** (`--filter=blob:none`) | Clones the full commit graph but skips downloading file contents (blobs) until accessed | Slow clone due to many/large files, but you still need full history for log/blame | Developers who need history navigation but don't want to wait for a full clone |

```bash
# Sparse checkout — only work on packages/backend
git clone --sparse <repo-url>
cd repo
git sparse-checkout set packages/backend shared/

# Shallow clone — CI only needs recent history
git clone --depth 1 <repo-url>

# Partial clone — full history, lazy blob download
git clone --filter=blob:none <repo-url>
```

**Combining them:** You can use partial clone + sparse checkout together for the best developer experience — fast clone, minimal working tree, full history available when needed.

**Why submodules are rarely the answer:**

Submodules embed one Git repo inside another, each with its own `.git`. The problems:

- **Version pinning is manual** — the parent repo pins a specific SHA of each submodule. Updating means committing new SHA pins, which is an extra step developers forget.
- **Broken clone workflow** — `git clone` doesn't fetch submodules by default (`--recurse-submodules` needed). New team members constantly hit "empty directory" issues.
- **Atomic cross-package changes are impossible** — changing code in the parent and a submodule requires two separate commits in two repos, coordinated manually.

Submodules make sense for truly independent repos you consume as a dependency (e.g., a vendored library). For a monorepo where packages change together frequently, they add friction without solving the core performance issues that sparse/partial clone handle better.

</details>

<details>
<summary>10. How do you approach merge conflict resolution beyond just picking sides — explain what the conflict markers mean (including the base version in diff3 mode), when --ours and --theirs are appropriate, what rerere does and when to enable it, and what practices (small PRs, frequent rebasing) minimize conflicts in the first place?</summary>

**Understanding conflict markers:**

Default conflict markers show two sides:

```
<<<<<<< HEAD
const timeout = 5000;
=======
const timeout = 10000;
>>>>>>> feature-branch
```

This tells you HEAD has `5000` and the incoming branch has `10000`, but you don't know what the value was *before* either change. Enable diff3 to see the common ancestor:

```bash
git config --global merge.conflictstyle diff3
```

Now conflicts show three sections:

```
<<<<<<< HEAD
const timeout = 5000;
||||||| merged common ancestors
const timeout = 3000;
=======
const timeout = 10000;
>>>>>>> feature-branch
```

The middle section shows the original value (`3000`). Now you can see that HEAD changed it from 3000 to 5000, and the feature branch changed it from 3000 to 10000. This context makes resolution much easier — you understand the *intent* of each change, not just the result.

**When `--ours` and `--theirs` are appropriate:**

```bash
# During a merge — accept our version for a specific file
git checkout --ours package-lock.json
git add package-lock.json

# Accept theirs
git checkout --theirs package-lock.json
git add package-lock.json
```

Use these for files where manual merging is pointless — lock files, generated files, or when you know one side's changes fully supersede the other. Never use them blindly on source code — you'll silently drop changes.

**Note:** During a rebase, `--ours` and `--theirs` are swapped compared to merge. In a rebase, "ours" is the branch you're rebasing *onto* (e.g., `main`), and "theirs" is the commit being replayed (your changes). This confuses everyone.

**rerere (reuse recorded resolution):**

```bash
git config --global rerere.enabled true
```

When enabled, Git records how you resolve each conflict. If the same conflict appears again (common during long rebases or repeated cherry-picks), Git automatically applies the same resolution. It doesn't commit — it stages the resolution and lets you verify.

Enable it globally — there's no downside. It's especially valuable for:

- Rebasing a feature branch onto `main` repeatedly during development
- Cherry-picking the same changes to multiple release branches

**Practices that minimize conflicts:**

- **Small PRs** — fewer lines changed means fewer opportunities for overlap
- **Frequent rebasing** — rebase your feature branch onto `main` regularly (daily if `main` is active) so conflicts are small and incremental rather than a massive merge at the end
- **Clear code ownership** — when different developers work in different parts of the codebase, conflicts are rare. CODEOWNERS helps formalize this.
- **Avoid reformatting commits** — whitespace and formatting changes in the same PR as logic changes create conflicts that look scary but are trivial. Do formatting changes in separate PRs.
- **Short-lived branches** — branches that live for days, not weeks, accumulate fewer conflicts

</details>

## Practical — Daily Git Operations

<details>
<summary>11. Walk through an interactive rebase workflow — show the exact commands to squash the last 5 commits into 2 meaningful ones, reword a commit message, reorder commits, edit a commit to split it into two, and drop a commit entirely. Explain what happens if a conflict arises mid-rebase and how you resolve it or abort.</summary>

**Starting an interactive rebase:**

```bash
git rebase -i HEAD~5
```

This opens your editor with the last 5 commits (oldest first):

```
pick a1b2c3d add user model
pick d4e5f6g add validation logic
pick h7i8j9k fix typo in validation
pick l0m1n2o add user controller
pick p3q4r5s fix controller import
```

**Squashing 5 commits into 2 meaningful ones:**

Change `pick` to `squash` (or `s`) for commits you want to fold into the one above:

```
pick a1b2c3d add user model
pick d4e5f6g add validation logic
squash h7i8j9k fix typo in validation
pick l0m1n2o add user controller
squash p3q4r5s fix controller import
```

Result: commits 2+3 become one commit, commits 4+5 become another. Git opens the editor for each group to let you write a combined commit message.

**Rewording a commit message:**

```
reword a1b2c3d add user model
pick d4e5f6g add validation logic
```

Git will pause and open the editor for just the commit message — file contents stay unchanged.

**Reordering commits:**

Simply move lines around in the editor. Git replays them in the new order:

```
pick l0m1n2o add user controller
pick a1b2c3d add user model
```

**Editing a commit to split it into two:**

```
edit a1b2c3d add user model and migration
```

Git stops at that commit. Now split it:

```bash
# Undo the commit but keep changes staged
git reset HEAD~1

# Stage and commit the model separately
git add src/models/user.ts
git commit -m "add user model"

# Stage and commit the migration separately
git add migrations/
git commit -m "add user migration"

# Continue the rebase
git rebase --continue
```

**Dropping a commit:**

Change `pick` to `drop` (or delete the line entirely):

```
pick a1b2c3d add user model
drop d4e5f6g temporary debug logging
```

**Handling conflicts mid-rebase:**

When a conflict occurs, Git pauses and tells you which files conflict:

```bash
# See what's conflicted
git status

# Resolve conflicts in your editor, then:
git add <resolved-files>
git rebase --continue

# Or abort the entire rebase and go back to where you started:
git rebase --abort

# Skip just this commit (rare — usually means the commit is no longer needed):
git rebase --skip
```

**Gotcha:** If you get conflicts on multiple commits during a rebase, you must resolve each one sequentially. This is the tradeoff vs merge, where you resolve everything once (as covered in question 2).

</details>

<details>
<summary>12. Show how to use the reflog to recover from three common disasters — recovering commits after a reset --hard, recovering your branch after a failed interactive rebase, and recovering a dropped stash. Show the exact commands for each scenario and explain how you identify the right reflog entry.</summary>

The reflog records every HEAD movement locally (as covered in question 6). Here's how to use it for the three most common recovery scenarios.

**Scenario 1: Recovering commits after `reset --hard`**

You accidentally ran `git reset --hard HEAD~3`, losing your last 3 commits.

```bash
# View the reflog — look for entries BEFORE the reset
git reflog

# Output:
# a1b2c3d HEAD@{0}: reset: moving to HEAD~3     <-- the destructive reset
# f4e5d6c HEAD@{1}: commit: add error handling    <-- this is what you lost
# 8901234 HEAD@{2}: commit: add validation
# abc5678 HEAD@{3}: commit: add user model

# Move back to the commit before the reset
git reset --hard HEAD@{1}
# Or use the SHA directly:
git reset --hard f4e5d6c
```

**How to identify the right entry:** Look for the reflog entry just before the `reset` line. The SHA on that line is where your branch was before the damage.

**Scenario 2: Recovering after a failed interactive rebase**

You started a rebase, conflicts got messy, you aborted or got into a bad state.

```bash
git reflog

# Output:
# d4e5f6g HEAD@{0}: rebase (abort): returning to refs/heads/feature
# 1234567 HEAD@{1}: rebase (pick): add validation
# 89abcde HEAD@{2}: rebase (start): checkout main
# f4e5d6c HEAD@{3}: commit: add error handling     <-- pre-rebase state

# If abort didn't restore correctly, or you finished a rebase you regret:
git reset --hard HEAD@{3}
```

**How to identify the right entry:** Look for `rebase (start)` — the entry just before it is your branch's pre-rebase state. Alternatively, check the branch-specific reflog:

```bash
git reflog show feature
# Shows only movements of the 'feature' branch pointer
```

**Scenario 3: Recovering a dropped stash**

You ran `git stash drop` or `git stash clear` and need the stash back.

```bash
# Stash drops don't appear in HEAD's reflog — use fsck to find dangling commits
git fsck --no-reflogs | grep "dangling commit"

# Output:
# dangling commit abc1234def567...
# dangling commit 789abc012def...

# Inspect each to find your stash
git show abc1234def567

# Once you find the right one, re-apply it
git stash apply abc1234def567

# Or create a branch from it to inspect safely
git branch recovered-stash abc1234def567
```

**Alternative — if the drop was recent, check the stash reflog:**

```bash
git reflog show stash

# Output:
# abc1234 stash@{0}: WIP on feature: add validation  <-- current top stash
# (dropped entries won't show here, but fsck finds them)
```

**How to identify the right entry:** `git show` on each dangling commit — stash commits have a characteristic format with "WIP on \<branch\>" or "On \<branch\>:" in the message. The commit will contain your stashed changes as a diff.

**Time limit:** These dangling objects survive until `git gc` prunes them (default: 2 weeks for unreachable objects, or 30 days via reflog expiry). Act quickly.

</details>

<details>
<summary>13. Walk through resolving a merge conflict step by step — show what the conflict markers look like (including diff3 format), demonstrate resolving with --ours and --theirs for specific files, configure and use rerere to auto-resolve recurring conflicts, and show the commands to complete the merge after resolution.</summary>

The conceptual background for conflict markers, `--ours`/`--theirs`, and rerere was covered in question 10. This answer focuses on the practical step-by-step workflow.

**Step 1: Start the merge and see what conflicts**

```bash
git merge feature-branch
# Auto-merging src/config.ts
# CONFLICT (content): Merge conflict in src/config.ts
# CONFLICT (content): Merge conflict in src/services/user.ts
# Automatic merge failed; fix conflicts and then commit the result.

git status
# Both modified: src/config.ts
# Both modified: src/services/user.ts
```

**Step 2: Understand the conflict (diff3 format)**

With diff3 enabled (see Q10), the conflict shows three sections:

```typescript
// src/services/user.ts
<<<<<<< HEAD
async function getUser(id: string): Promise<User> {
  return db.users.findById(id);
||||||| merged common ancestors
async function getUser(id: number): Promise<User> {
  return db.users.findById(id);
=======
async function getUser(id: number): Promise<User | null> {
  return db.users.findById(id) ?? null;
>>>>>>> feature-branch
}
```

The base had `id: number` with `Promise<User>`. HEAD changed the param to `string`. Feature branch changed the return to `User | null`. The correct resolution combines both:

```typescript
async function getUser(id: string): Promise<User | null> {
  return db.users.findById(id) ?? null;
}
```

**Step 3: Use --ours/--theirs for generated files**

```bash
# Lock file conflicts are noise — just take ours and regenerate
git checkout --ours package-lock.json
npm install  # regenerate the lock file
git add package-lock.json
```

**Step 4: Mark resolved files and complete the merge**

```bash
# After manually fixing src/services/user.ts and src/config.ts:
git add src/services/user.ts src/config.ts

# Verify no conflicts remain
git status
# All conflicts fixed but you are still merging.

# Complete the merge
git commit
# Git opens editor with a pre-filled merge commit message
```

**Step 5: Set up rerere for recurring conflicts**

```bash
# Enable rerere globally (one-time setup)
git config --global rerere.enabled true

# Now, every conflict you resolve gets recorded.
# Next time the same conflict appears (e.g., rebasing the same branch again):
git merge feature-branch
# Resolved 'src/config.ts' using previous resolution.

# rerere stages the resolution but does NOT commit — verify first:
git diff --staged src/config.ts
git add src/config.ts
git commit
```

**If rerere applies a wrong resolution:**

```bash
git checkout --conflict=merge src/config.ts  # reset to conflicted state
git rerere forget src/config.ts              # delete the recorded resolution
# Now resolve manually again
```

**Useful commands during conflict resolution:**

```bash
# See what the merge is trying to do (before resolving)
git log --merge --oneline        # commits unique to each side
git diff --name-only --diff-filter=U  # list only conflicted files

# See the three versions of a conflicted file
git show :1:src/config.ts  # base (common ancestor)
git show :2:src/config.ts  # ours (HEAD)
git show :3:src/config.ts  # theirs (feature-branch)
```

</details>

## Practical — Repository & Workflow Configuration

<details>
<summary>14. Set up a monorepo for efficient development — show how to configure sparse checkout so a developer only clones the packages they work on, write a CODEOWNERS file that assigns different teams to different directories, and demonstrate path-based CI filtering (only run backend tests when backend/ changes). Show the exact commands and config files.</summary>

**Sparse checkout setup:**

The basic sparse checkout commands were shown in Q9. For monorepo development, combine sparse checkout with partial clone and manage context switching between packages:

```bash
# Initial setup: partial clone + sparse checkout (fast clone, minimal working tree)
git clone --sparse --filter=blob:none https://github.com/org/monorepo.git
cd monorepo

# Set your working packages
git sparse-checkout set packages/backend packages/shared configs/

# Switch context: start working on auth package too
git sparse-checkout add packages/auth

# Check what you currently have checked out
git sparse-checkout list

# Drop a package you're done with (blobs stay cached locally)
git sparse-checkout set packages/auth packages/shared configs/
```

Root files (`package.json`, `tsconfig.json`, etc.) are always included. Blobs for non-checked-out paths are fetched lazily only when accessed (e.g., via `git log -p` or `git blame`).

**CODEOWNERS file:**

Place at `.github/CODEOWNERS` (GitHub) or `CODEOWNERS` at the repo root:

```
# Default — catches anything not matched below
* @org/platform-team

# Backend packages
packages/backend/           @org/backend-team
packages/auth/              @org/backend-team @org/security-team

# Frontend packages
packages/frontend/          @org/frontend-team
packages/dashboard/         @org/frontend-team

# Shared code — requires review from both teams
packages/shared/            @org/backend-team @org/frontend-team

# Infrastructure
terraform/                  @org/infra-team
.github/                    @org/infra-team

# Specific critical files
packages/backend/src/billing/   @org/billing-team @org/backend-team
```

Rules are evaluated top-to-bottom; the **last matching pattern wins**. The billing directory override works because it comes after the broader `packages/backend/` rule.

**Path-based CI filtering (GitHub Actions):**

```yaml
# .github/workflows/backend-tests.yml
name: Backend Tests

on:
  pull_request:
    paths:
      - 'packages/backend/**'
      - 'packages/shared/**'    # shared code affects backend
      - 'package.json'          # dependency changes
      - 'tsconfig.base.json'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          sparse-checkout: |
            packages/backend
            packages/shared
          # Only check out what the tests need — faster CI

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - run: npm ci
      - run: npm test --workspace=packages/backend
```

```yaml
# .github/workflows/frontend-tests.yml
name: Frontend Tests

on:
  pull_request:
    paths:
      - 'packages/frontend/**'
      - 'packages/shared/**'
      - 'package.json'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test --workspace=packages/frontend
```

**Path filtering with `paths-ignore` (alternative):**

```yaml
on:
  pull_request:
    paths-ignore:
      - 'docs/**'
      - '**.md'
      - '.vscode/**'
```

Use `paths` when you want to explicitly opt-in to triggering. Use `paths-ignore` when most changes should trigger but you want to exclude documentation-only changes.

**GitLab CI equivalent:**

```yaml
backend-tests:
  rules:
    - changes:
        - packages/backend/**
        - packages/shared/**
  script:
    - npm ci
    - npm test --workspace=packages/backend
```

</details>

## Practical — History & Search

<details>
<summary>15. Show how to force-push safely and configure branch protection — demonstrate the difference between --force and --force-with-lease (and when --force-with-lease can still fail), show how to configure branch protection rules on GitHub/GitLab (required reviews, status checks, no force-push), and what to do when you need to force-push to a shared branch as a last resort.</summary>

The conceptual explanation of force-push dangers and `--force-with-lease` was covered in question 8. This answer focuses on the practical commands and configuration.

**`--force` vs `--force-with-lease` in action:**

```bash
# Dangerous — overwrites remote unconditionally
git push --force origin feature

# Safe — only overwrites if remote matches your last known state
git push --force-with-lease origin feature

# Safest — explicitly specify the expected remote SHA
git push --force-with-lease=origin/feature:abc1234 origin feature
```

**When `--force-with-lease` silently fails to protect you:**

```bash
# You rebase your branch
git rebase main

# A teammate pushes to the same branch
# ...

# You fetch (maybe your IDE auto-fetches, or you ran git pull on another branch)
git fetch

# Now --force-with-lease sees the remote as "matching" your local tracking ref
# because fetch updated it. The push succeeds, overwriting your teammate's work.
git push --force-with-lease origin feature  # DANGER: still succeeds
```

**Mitigation:** Set `push.useForceIfIncludes` to require that your local branch actually includes the remote's commits:

```bash
git config --global push.useForceIfIncludes true
```

This makes `--force-with-lease` reject the push if the remote has commits you haven't integrated, even if you've fetched them.

**GitHub branch protection setup (Settings > Branches > Branch protection rules):**

The rules themselves were covered conceptually in Q8. Here's the exact UI configuration:

```
Branch name pattern: main

[x] Require a pull request before merging
    [x] Require approvals: 1
    [x] Dismiss stale pull request approvals when new commits are pushed

[x] Require status checks to pass before merging
    [x] Require branches to be up to date before merging
    Required checks: ci/tests, ci/lint

[x] Do not allow force pushes
[x] Do not allow deletions
[x] Require linear history          (optional — enforces squash or rebase merge)
[x] Include administrators
```

**GitLab equivalent (Settings > Repository > Protected branches):**

```
Branch: main
Allowed to merge: Maintainers
Allowed to push: No one            (forces MR workflow)
Allowed to force push: No
Code owner approval required: Yes
```

**Last resort: force-pushing to a shared branch**

When you absolutely must (e.g., accidentally committed secrets to `main`):

1. **Communicate first** — notify the team in Slack/chat that you're about to force-push, and tell everyone to stop pushing
2. **Have everyone stash or note their current branch state**
3. **Do the force-push:**

```bash
# Remove the sensitive commit
git rebase -i HEAD~3  # or use git filter-branch / git-filter-repo

# Force-push with explicit SHA check
git push --force-with-lease=origin/main:<expected-sha> origin main
```

4. **Everyone else resets to the new remote:**

```bash
git fetch origin
git reset --hard origin/main
# If they had local commits on top:
git cherry-pick <their-commit-shas>
```

5. **Post-mortem** — document what happened and ensure branch protection is tightened to prevent recurrence

</details>

<details>
<summary>16. Configure PR merge strategy settings on a repository — show how to set up squash-merge as the default on GitHub/GitLab, configure the default squash commit message template, explain when you'd override the default for specific PRs (e.g., merge-commit for a long-lived feature branch), and how to enforce a consistent strategy across a team.</summary>

**GitHub setup (Settings > General > Pull Requests):**

```
[x] Allow squash merging          (enable)
    Default commit message: Pull request title and description

[ ] Allow merge commits            (disable to enforce squash-only)
[ ] Allow rebase merging           (disable to enforce squash-only)

[ ] Always suggest updating pull request branches
[x] Automatically delete head branches
```

By only enabling squash merging, the merge button only offers squash — no manual discipline needed.

**If you want flexibility (allow merge-commit as override):**

```
[x] Allow squash merging           (default)
[x] Allow merge commits            (for exceptional cases)
[ ] Allow rebase merging

Default to: Squash and merge       (dropdown under merge options)
```

**Squash commit message template:**

GitHub uses the PR title as the commit subject and the PR description as the body by default (when configured as "Pull request title and description"). A common convention:

```
feat: add user authentication (#142)

- Add JWT token generation and validation
- Add login/logout endpoints
- Add auth middleware for protected routes

Co-authored-by: Alice <alice@example.com>
```

The PR number `(#142)` is automatically appended by GitHub. Configure your PR template (`.github/pull_request_template.md`) to structure descriptions consistently:

```markdown
## What
Brief description of the change.

## Why
Context and motivation.

## Testing
How this was verified.
```

**GitLab setup (Settings > Merge Requests):**

```
Merge method: Squash commits       (or "Merge commit with semi-linear history")
Squash commits when merging:        Encourage (or Require)

Default merge commit message template:
%{title} (%{reference})

%{description}
```

**When to override squash-merge:**

- **Long-lived feature branch with many logical changes** — if the branch has 50 commits representing 8 distinct features, a single squash commit loses valuable context. Use merge-commit to preserve the individual commits.
- **Release branch merges** — merging `release/v2.0` back to `main` should use a merge-commit so the full release history is preserved on `main`.
- **Preserving bisect granularity** — if a large PR might introduce a subtle bug, keeping individual commits makes bisect useful (as covered in question 7).

On GitHub, the PR author or reviewer selects the merge strategy from the dropdown on the merge button — no admin action needed for per-PR overrides.

**Enforcing consistency across a team:**

1. **Repo settings** — disable strategies you don't want (strongest enforcement)
2. **CI check** — add a GitHub Action that validates commit message format on merge to `main`:

```yaml
# .github/workflows/commit-lint.yml
name: Commit Lint
on:
  pull_request:
    types: [opened, edited, synchronize]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: wagoid/commitlint-github-action@v5
        with:
          configFile: .commitlintrc.json
```

```json
{
  "extends": ["@commitlint/config-conventional"],
  "rules": {
    "type-enum": [2, "always", ["feat", "fix", "refactor", "docs", "chore", "test", "ci"]],
    "subject-max-length": [2, "always", 72]
  }
}
```

3. **Team documentation** — document the merge strategy choice and the reasoning in the contributing guide. People follow conventions better when they understand why.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>17. Tell me about a time you recovered from a Git disaster in a team setting — what happened (reset --hard, bad rebase, lost commits, force-push to main), how did you diagnose and fix it, and what safeguards did you put in place afterward?</summary>

**What the interviewer is looking for:**

- Calm, systematic debugging under pressure
- Understanding of Git internals (not just "I Googled it")
- That you improved the process afterward — not just fixed the immediate problem
- Communication during the incident (did you warn the team, or silently fix it?)

**Suggested structure (STAR+):**

1. **Situation** — set the scene briefly: team size, what branch, what was at stake
2. **What went wrong** — be specific about the Git operation that caused the disaster
3. **Diagnosis** — how you figured out what happened (reflog, comparing SHAs, checking remote state)
4. **Recovery** — exact steps you took to fix it, including any communication to the team
5. **Safeguards** — what you put in place to prevent recurrence

**Example outline to personalize:**

> A teammate force-pushed to `main` after a rebase, overwriting 3 commits from other developers that had been merged that morning. We noticed because CI started failing on tests that had previously passed.
>
> I checked the reflog on the CI runner (which had recently fetched) and identified the 3 missing commit SHAs. Confirmed by comparing `git log origin/main` with what the teammate's local branch showed. The teammate's local history was missing the 3 commits entirely — they'd rebased `main` onto their feature branch (backwards) and force-pushed.
>
> Recovery: I used one of the other developers' local repos (which still had the correct `main`) to `git push --force-with-lease` the correct history back. Verified all 3 commits were restored, CI turned green.
>
> Safeguards afterward: Enabled branch protection on `main` (no force pushes, require PR reviews, include admins). Added a team guideline about only force-pushing to personal feature branches. Set up a Slack notification for any force-push events via GitHub webhooks.

**Key points to hit:**

- Show you understand **why** the disaster happened at a Git level (not just "someone did something wrong")
- Emphasize **communication** — "I immediately told the team to stop pushing to main while I investigated"
- The safeguards should be **systemic** (branch protection, automation), not just "we told people to be more careful"

</details>

<details>
<summary>18. Describe a time you set up or changed a team's branching strategy — what was the previous workflow, what problems drove the change, how did you get the team to adopt it, and what was the outcome?</summary>

**What the interviewer is looking for:**

- Ability to identify process problems and propose solutions
- Change management skills — technical solutions need human buy-in
- Understanding of branching model tradeoffs (covered in question 3)
- Measurable improvement, not just "we changed it and it felt better"

**Suggested structure:**

1. **Previous workflow** — what was the team doing? Name the model (or describe the ad-hoc process)
2. **Pain points** — what concrete problems drove the change? (merge conflicts, slow releases, broken main, unclear what's deployed)
3. **Proposed change** — what did you propose and why? Reference the tradeoffs.
4. **Adoption** — how did you get the team on board? (RFC, demo, gradual rollout, tooling)
5. **Outcome** — what improved? Use metrics if possible (deploy frequency, time to merge, incidents)

**Example outline to personalize:**

> Our team of 8 was using a GitFlow-like model with `develop`, `staging`, and `main` branches. The problems: release branches lived for 2+ weeks, merge conflicts between `develop` and `main` were constant, and hotfixes required cherry-picking across 3 branches — which often went wrong.
>
> I proposed moving to GitHub Flow (branch from `main`, PR back to `main`, deploy from `main`). The key insight: our product was a SaaS platform with continuous deployment — we didn't need the release branch ceremony GitFlow provides. I wrote a short RFC comparing the two models with our specific pain points mapped to how GitHub Flow solves them.
>
> Adoption: I ran a 30-minute demo showing the new workflow, updated the contributing guide, configured branch protection on `main` (required reviews, status checks), and set squash-merge as the default. We ran both models in parallel for one sprint — new features used GitHub Flow, in-flight work finished under GitFlow.
>
> Outcome: Deploy frequency went from weekly to multiple times per day. Merge conflicts dropped significantly (no more develop-to-main drift). Time from PR open to deploy went from days to hours. The team's biggest concern (no staging branch) was addressed by adding preview environments per PR.

**Key points to hit:**

- Show the change was **driven by real problems**, not just preference
- Demonstrate **empathy for the team** — you didn't just mandate a change, you addressed concerns and made adoption smooth
- The outcome should reference **concrete improvements**, not just feelings
- If something didn't go perfectly, mention it — "we underestimated how much CI needed to improve first" shows self-awareness

</details>
