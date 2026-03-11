# Git & Version Control

> **31 questions** — 14 theory, 17 practical

- Git internals: three-tree architecture, branches as pointers, commits as a DAG, SHA-1 hashing
- Rebase vs merge: history readability, conflict effort, bisect usability, shared branch safety
- Interactive rebase: squash, reword, reorder, edit, drop commits
- Cherry-picking: moving commits between branches, ranges, -x flag, risks of duplicate commits
- Bisect: finding regressions via binary search, manual and automated workflows
- Stashing: git stash push/pop/apply, partial stashing (-p), stash list management, when stashing breaks down (multiple stashes, conflicts on pop)
- Reflog: recovering from reset --hard, failed rebase, dropped stashes, expiry
- Branching models: trunk-based development, GitFlow, GitHub Flow — team size, CI maturity, tagging and release workflows
- Tags: annotated vs lightweight, semantic versioning, tag-triggered CI/CD pipelines, signing tags
- PR merge strategies: squash-merge vs merge-commit vs rebase-merge tradeoffs
- Force-push: dangers on shared branches, --force-with-lease, branch protection
- Merge conflict resolution: conflict markers, --ours/--theirs, rerere, minimizing conflicts with small PRs and frequent rebasing
- Monorepo workflows: sparse checkout, shallow/partial clone, CODEOWNERS, path-based CI filtering — and why submodules are rarely the answer
- Git worktrees: parallel branches without stashing, shared .git directory, use cases (code review, hotfixes, CI)
- Git hooks: pre-commit, commit-msg, pre-push, husky/lint-staged, client-side vs server-side
- History rewriting: removing secrets with filter-repo (not filter-branch), force-push coordination, secret rotation after exposure
- Code search and blame: git log -S (pickaxe), -G regex, --follow renames, git blame -w, .git-blame-ignore-revs

---

## Foundational

<details>
<summary>1. How does Git store data internally — what is the three-tree architecture (working directory, staging area, repository), how are commits structured as a directed acyclic graph, what does it mean that branches are just pointers to commits, and why does Git use SHA-1 hashing for content addressing?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. Why would you choose rebase over merge (or vice versa) — how does each strategy affect history readability, conflict resolution effort, the usefulness of git bisect, and what are the dangers of rebasing commits that have already been pushed to a shared branch?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. Why do different teams need different branching models — compare trunk-based development, GitFlow, and GitHub Flow in terms of team size, CI maturity, release cadence, and risk tolerance, and explain what signals tell you a team has outgrown its current model?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>4. When and why would you cherry-pick commits instead of merging a branch — what problems does cherry-picking solve, what does the -x flag do and why should you use it, how do you cherry-pick a range of commits, and what are the risks of duplicate commits appearing in your history?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. How does git bisect find regressions using binary search — why is this approach faster than scanning commits linearly, what makes a codebase bisect-friendly (and what breaks it), and when would you use manual bisect vs automated bisect with a test script?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. What does the reflog track that the regular commit log doesn't — what types of operations does it record, why is it your safety net for recovering from destructive operations like reset --hard and failed rebases, and when does the reflog expire so you can no longer recover?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. What are the tradeoffs between squash-merge, merge-commit, and rebase-merge as PR merge strategies — how does each affect commit history cleanliness, blame/bisect accuracy, the ability to revert a PR as a unit, and what strategy works best for different team sizes and workflows?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. Why is force-pushing dangerous on shared branches — what exactly happens to other developers' work when you force-push, how does --force-with-lease protect against overwriting someone else's commits, and what branch protection rules should you configure to prevent accidental force-pushes to main?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. What are the key challenges of working in a monorepo with Git — why do large repositories cause performance problems, and how do sparse checkout, shallow clone, and partial clone each address different aspects of this problem? When would you use each one, and why are Git submodules rarely the right answer for monorepo workflows?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. What problem do Git worktrees solve that stashing and branch switching don't — how do worktrees share a single .git directory while maintaining separate working directories, and what are the practical use cases (code review, hotfixes, running CI locally) that make worktrees valuable?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. How does git stash work and when does it break down — explain the difference between stash push, pop, and apply, how partial stashing with -p lets you stash specific changes, how to manage a stash list when it grows, and why relying on multiple stashes often leads to conflicts on pop and lost work?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. What are the different types of Git hooks (pre-commit, commit-msg, pre-push), what is each best used for, how do client-side hooks differ from server-side hooks in terms of enforcement guarantees, and why do tools like husky and lint-staged exist instead of just using raw Git hooks?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. Why is rewriting history to remove secrets more complex than just deleting the file in a new commit — what does git filter-repo do that makes it the right tool (and why not filter-branch), how do you coordinate the force-push with your team, and why must you still rotate every exposed secret even after rewriting history?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. How do you approach merge conflict resolution beyond just picking sides — explain what the conflict markers mean (including the base version in diff3 mode), when --ours and --theirs are appropriate, what rerere does and when to enable it, and what practices (small PRs, frequent rebasing) minimize conflicts in the first place?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Daily Git Operations

<details>
<summary>15. Walk through an interactive rebase workflow — show the exact commands to squash the last 5 commits into 2 meaningful ones, reword a commit message, reorder commits, edit a commit to split it into two, and drop a commit entirely. Explain what happens if a conflict arises mid-rebase and how you resolve it or abort.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Show how to cherry-pick commits across branches — demonstrate cherry-picking a single commit, a range of commits, and using the -x flag. Then show how to handle a cherry-pick conflict and what the resulting history looks like compared to a merge.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Walk through a complete git bisect workflow — show the manual process (git bisect start, good, bad, reset) and then automate it with git bisect run using a test script. Show what a good bisect test script looks like and how to interpret the result when bisect identifies the offending commit.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. Show how to use the reflog to recover from three common disasters — recovering commits after a reset --hard, recovering your branch after a failed interactive rebase, and recovering a dropped stash. Show the exact commands for each scenario and explain how you identify the right reflog entry.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. Walk through resolving a merge conflict step by step — show what the conflict markers look like (including diff3 format), demonstrate resolving with --ours and --theirs for specific files, configure and use rerere to auto-resolve recurring conflicts, and show the commands to complete the merge after resolution.</summary>

<!-- Answer will be added later -->

</details>

## Practical — Repository & Workflow Configuration

<details>
<summary>20. Set up a branching model with a proper tagging and release workflow — show the commands for creating release branches, tagging releases with annotated tags using semantic versioning, and the workflow for hotfixing a released version. Explain how tag-triggered CI/CD pipelines work (what triggers the build, how the tag becomes the version), when you'd sign tags with GPG and why, and compare how this differs between trunk-based development and GitFlow.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. Configure Git hooks using husky and lint-staged in a Node.js project — show how to set up a pre-commit hook that runs ESLint and Prettier only on staged files, a commit-msg hook that enforces conventional commit format, and a pre-push hook that runs tests. Explain how to handle the case where a developer bypasses hooks with --no-verify.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. Set up a monorepo for efficient development — show how to configure sparse checkout so a developer only clones the packages they work on, write a CODEOWNERS file that assigns different teams to different directories, and demonstrate path-based CI filtering (only run backend tests when backend/ changes). Show the exact commands and config files.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. Set up Git worktrees for parallel development — show how to create a worktree for reviewing a colleague's PR while your main branch has uncommitted work, create a worktree for an emergency hotfix, list and clean up worktrees, and explain the gotchas (you can't check out the same branch in two worktrees).</summary>

<!-- Answer will be added later -->

</details>

## Practical — History & Search

<details>
<summary>24. Show how to force-push safely and configure branch protection — demonstrate the difference between --force and --force-with-lease (and when --force-with-lease can still fail), show how to configure branch protection rules on GitHub/GitLab (required reviews, status checks, no force-push), and what to do when you need to force-push to a shared branch as a last resort.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>25. Walk through removing a leaked secret from Git history using git filter-repo — show the exact commands to identify which commits contain the secret, rewrite history to remove it from all commits, coordinate the force-push with your team (what they need to do with their local clones), and the post-rewrite checklist (rotate secret, check CI caches, verify backups).</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. Show how to use Git's code search and blame tools effectively — demonstrate git log -S (pickaxe) to find when a string was added or removed, -G with a regex for pattern matching, --follow to track a renamed file, git blame -w to ignore whitespace changes, and how to set up .git-blame-ignore-revs to skip formatting commits in blame output.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. Configure PR merge strategy settings on a repository — show how to set up squash-merge as the default on GitHub/GitLab, configure the default squash commit message template, explain when you'd override the default for specific PRs (e.g., merge-commit for a long-lived feature branch), and how to enforce a consistent strategy across a team.</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>28. Tell me about a time you recovered from a Git disaster in a team setting — what happened (reset --hard, bad rebase, lost commits, force-push to main), how did you diagnose and fix it, and what safeguards did you put in place afterward?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>29. Describe a time you set up or changed a team's branching strategy — what was the previous workflow, what problems drove the change, how did you get the team to adopt it, and what was the outcome?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>30. Tell me about a time you dealt with Git performance issues in a large repository — what were the symptoms (slow clones, slow checkouts, large .git directory), what was the root cause, and what techniques did you use to fix it?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>31. Describe a time you introduced Git workflow tooling (hooks, CI filters, automation) to a team — what problem were you solving, how did you implement it, how did you handle pushback or edge cases, and what impact did it have on code quality or developer experience?</summary>

<!-- Answer framework will be added later -->

</details>
