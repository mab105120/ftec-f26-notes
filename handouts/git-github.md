# Version Control with Git and GitHub

## Why this exists

Think about how you've actually kept "old versions" of something before — a model, a script, a writeup. Most people land on one of a few habits: saving `analysis_v2_final_FINAL.xlsx` next to `analysis_v2_final.xlsx`, emailing a file back and forth with a colleague so each version lives in someone's inbox, or just overwriting the file and hoping you never need what was there before. Say those out loud and they sound a little absurd — and they are. But they're also the default, because without a tool built for the problem, there's no better option.

**Version control** is that tool. It's a system that automatically keeps a complete, searchable history of every change made to a set of files: who changed what, when, and — because you write a short message with every change — why. The important word there is *history*. It's tempting to think of version control as a fancy backup, but a backup only protects the latest copy. What you actually want, and what version control gives you, is every copy, connected by a record of how you got from one to the next.

That history buys you two distinct things, and it's worth naming them separately because they solve different problems:

**An audit trail.** You can always answer "what did this look like last week, and exactly what changed since then?" This is the same discipline behind a trade blotter or an audit log in a financial system: every entry is timestamped, attributed to whoever made it, and never silently overwritten. If a regulator — or your professor — asks why a number changed, "we're not sure, someone edited the spreadsheet" is not an acceptable answer. Once software is involved, "we're not sure, someone edited the code" is exactly as unacceptable, and version control is what makes a real answer possible.

**Safe experimentation.** You can try something risky — a new approach, a fix you're not sure about — in complete isolation from the working version, and if it doesn't pan out, undo it at zero cost. No more commenting out the old code "just in case" and leaving dead code scattered through a file forever. This turns out to matter as much as the audit trail, and it's the idea behind branching, which comes later in this handout.

Git is the specific version control tool this course uses — by a wide margin the most common one in the industry, and the same tool used to submit every assignment this semester. GitHub is a separate thing: a website that hosts Git repositories online and adds collaboration features (like pull requests, covered below) on top of plain Git. Git is the tool; GitHub is where a copy of your work lives so other people — your classmates, your professor, in a real job your teammates — can see it and work with it too.

## Core concepts: repository, staging, and commits

Before touching a terminal, four terms need to be precise, because the rest of this handout builds directly on them.

A **repository** (usually shortened to "repo") is a folder that Git is tracking the full history of. Not every folder on your computer is a repo — you tell Git to start tracking one, and from that point forward it watches every file inside it for changes.

The **working directory** is just the files as they currently sit on your disk, including whatever you're mid-edit on right now. It's the messy, in-progress state — nothing here is part of the permanent history yet.

**Staging** is the act of marking specific changes as "ready to be included in the next snapshot." This is a deliberate middle step, and it exists for a reason worth sitting with: it lets a snapshot of your work be a *curated* decision, not just "whatever happened to be sitting on disk when I ran a command." Suppose you're working on the portfolio tracker and you've simultaneously (a) fixed a real bug in how gain/loss is computed and (b) left some unrelated scratch debugging code in another file. Staging lets you include only the bug fix in the next snapshot and leave the scratch code out, even though both currently exist in your working directory.

A **commit** is a saved snapshot of whatever was staged, permanently recorded with a message explaining why the change was made. This is the fundamental unit of Git's history — every entry in the audit trail described above is a commit. A good commit message says what changed and, more importantly, why: `"fix gain/loss calc to use average cost basis, not last purchase price"` is useful six months from now; `"fix bug"` is not.

Here's what that sequence looks like as actual commands, using a one-line notes file as a stand-in for a real project file:

```
git init                          # start tracking this folder as a repo
echo "first note" > notes.txt     # create a file — working directory only
git status                        # shows notes.txt as untracked
git add notes.txt                 # stage it
git status                        # now shows it staged, ready to commit
git commit -m "add initial notes" # take the snapshot, with a message
```

After that, editing the file again and running `git status` shows it as modified but not yet staged — you can inspect exactly what changed with `git diff` before deciding whether to stage and commit it. Committing a second time and then running `git log` shows the full history: both commits, each with its author, timestamp, and message. That log is the audit trail made concrete — not a concept anymore, but something you can actually read.

Why small, frequent commits rather than one giant commit at the end of a work session? Because the audit trail is only as useful as its granularity. If everything you did in three hours lands in one commit, "what changed and why" is answerable only for the whole three hours, not for any specific decision inside it. This is the same idea as fine-grained requirements and tests: when something breaks later, a history of small, well-described commits lets you isolate exactly when and why, instead of staring at one enormous diff.

## Branching: isolating work safely

A **branch** is an independent line of development — a way to make changes without touching the main line of history until you're deliberately ready to bring them together. This is what makes "safe experimentation" from the opening section concrete rather than a slogan: work happens on a branch, and the main branch is completely unaffected until you choose to merge.

The analogy worth holding onto: a branch is like drafting a redline of a term sheet. You're not editing the executed, agreed-upon original — you're working on a separate copy, and it only becomes part of the real document once someone reviews it and agrees to incorporate it. Until then, your redline and the original can coexist without either being at risk.

```
git branch                        # lists branches — starts with just one (main)
git checkout -b add-summary       # create a new branch and switch to it
# ...edit notes.txt, then...
git add notes.txt
git commit -m "add portfolio summary line"
git checkout main                 # switch back to main
# notes.txt here does NOT have the new line — the branch's work is isolated
git checkout add-summary          # switch forward again
# the change is still there, exactly as you left it
```

Nothing was lost or put at risk moving in either direction — that's the entire value proposition of a branch. If the change on `add-summary` turned out to be wrong, you could simply never merge it, or delete the branch outright, and `main` would be exactly as if it never happened.

Branches aren't magic, though. Two branches can independently edit the *same lines* of the *same file* in incompatible ways, and when that happens, Git has no way to guess which version you actually want. That situation is a merge conflict, and it's the next concept — not a failure, but an expected part of working with branches.

## Merge conflicts: Git asking a human to decide

A **merge conflict** happens when you try to combine two branches that changed the same lines of the same file in different ways, and Git can't automatically determine which change should win. This is worth stating plainly because it's easy to panic the first time it happens: a conflict is not a mistake and not a sign that something is broken. It's Git correctly refusing to silently guess on your behalf, and asking you — the person who understands the intent behind both changes — to make the call instead.

The contract-redlining analogy extends naturally here: imagine two people independently redlining the same clause of a contract, then trying to merge both redlines into one final document. Someone has to sit down and reconcile the two versions of that clause by hand. Git will happily auto-merge changes that touch *different* clauses (different lines) without bothering you at all — it only stops and asks when the same line is genuinely in dispute.

Here's the shape of it in practice. Suppose two branches, `edit-a` and `edit-b`, each independently change the same first line of `notes.txt`:

```
git checkout main
git merge edit-a      # clean merge — main now has edit-a's version of the line
git merge edit-b      # CONFLICT (content): Merge conflict in notes.txt
```

Opening the file at that point shows Git's conflict markers directly in the text:

```
<<<<<<< HEAD
(edit-a's version of the line)
=======
(edit-b's version of the line)
>>>>>>> edit-b
```

Read this literally: everything between `<<<<<<< HEAD` and `=======` is the version currently on your branch; everything between `=======` and `>>>>>>> edit-b` is the incoming version from the branch you're merging in. Resolving the conflict means editing the file by hand — pick one version, combine the two, or write something new entirely — and then deleting the conflict markers themselves, since they're not meant to stay in the file. Once that's done:

```
git add notes.txt
git commit                        # Git recognizes the conflict as resolved
```

That commit becomes a normal merge commit, no different from any other entry in the history, and `git log --oneline --graph` will show the two lines of work coming back together into one. There's no special recovery mode here and no risk of losing work — resolving a conflict is just editing a file like any other edit, and telling Git you're done.

## Pull requests and code review

Everything so far has happened purely on your own computer. To share work — with a classmate, a teammate, or an instructor — you need a **remote**: a copy of the repository hosted somewhere else, in this course's case on GitHub, which you sync with using `push` (send your local commits up) and `pull` (bring down commits made elsewhere).

```
git push origin add-summary       # send this branch's commits to GitHub
```

Once a branch is pushed, GitHub lets you open a **pull request (PR)**: a request to merge one branch into another, visible on GitHub, that other people can read, comment on, and approve before anything actually merges. It's worth being precise about what a PR is, because it's easy to think of it as just a submission mechanism. A PR is a *proposal* — it has its own comment thread, its own diff view showing exactly what changed line by line, and it stays open for discussion until someone decides it's ready to merge.

**Code review** is the practice of someone other than the author reading a PR before it merges — checking for correctness, clarity, and whether the change fits how the rest of the project works. This is a real practice on professional engineering teams, not something invented for this course: proposing a change, having someone else look at it, and then merging it once it's approved is close to the default workflow at most companies that build software.

This matters for this course in two very direct ways. First, every assignment this semester is submitted exactly this way: branch, commit, push, open a PR — and grading happens by reading that PR, the same way a professional reviewer reads a teammate's proposed change. Second, because a PR is visible by design — not a private file exchange — sharing code with a classmate isn't something that happens quietly. It shows up as a reviewable diff that anyone with access can see. This is precisely why this course's collaboration policy is stated the way it is: discussing concepts and approach out loud is fine, but sharing or viewing another student's code is not, and "I didn't think anyone would see it" was never actually true once the workflow runs through PRs.

The mechanical flow, once a branch is pushed:

1. On GitHub, open the "Open a pull request" flow from your pushed branch.
2. Give it a title and description explaining what the change does and why.
3. Reviewers read the diff view and can leave comments on specific lines.
4. Once approved, the PR is merged, and the change becomes part of the main branch — visible to everyone with access to the repository from that point forward.

## The workflow you'll use for every assignment

Everything above compiles into one repeatable loop, and it is worth memorizing because you will run it, unmodified, for every assignment this semester:

1. **Branch** — create a new branch for the work you're about to do, isolating it from the main line of history.
2. **Commit** — as you make progress, stage and commit your changes in small, well-described pieces.
3. **Push** — send your branch, with its commits, to GitHub.
4. **Pull request** — open a PR from that branch, which is what gets reviewed and graded.

There is no separate "submission" step beyond this — opening the PR *is* the submission. Getting comfortable with this loop now, before it's attached to a graded deliverable, is the entire point of practicing it hands-on this week.

## FAQs

**"Isn't this just Google Docs version history?"** Similar idea, meaningfully more control. Docs saves a snapshot roughly whenever it feels like it and gives you one linear history. Git lets you decide exactly when a snapshot happens and exactly what's included in it (staging), and branching lets multiple independent lines of change exist at the same time — something a single linear history can't represent at all.

**"What if I mess something up?"** This is exactly the safe-experimentation pitch from the beginning of this handout. Almost everything in Git is recoverable, and working on a branch specifically exists to make mistakes cheap — a branch you abandon or delete leaves the main line of work completely untouched. If you get stuck, it's fine to ask for help rather than trying to force your way out.

**"Do I have to use the command line?"** Graphical Git tools exist and plenty of professionals use them day to day, but this course teaches the command line specifically because it makes the underlying model — staging, commits, branches — visible instead of hiding it behind buttons. It's also the one tool guaranteed to work in any environment you'll encounter later in the course.

**"Did I break something by causing a merge conflict?"** No. A conflict means Git found two genuinely different edits to the same line and is asking a person to decide between them — it is the expected, correct outcome of two branches disagreeing, not an error state or a sign you did something wrong.

**"What's the difference between merge and rebase — don't real teams use rebase?"** Rebase is a real alternative to merging that rewrites a branch's history onto a new base instead of creating a merge commit, and it's common on teams where several people share one branch and care about keeping a clean, linear history. It isn't part of this course's workflow, so one sentence is enough for now — it's worth knowing it exists, and worth looking into further on your own if you're curious, but not something to worry about for assignments here.

## Check your understanding

Use these to test yourself, not as a substitute for the explanations above — if a question is hard to answer without re-reading a section, that's the section to reread.

- Why is version control described as being about *history*, not backup? What would you lose if you only kept the latest copy of everything?
- What problem does staging solve that committing everything in the working directory directly would not?
- Explain, in your own words, why a merge conflict is not a bug or a mistake.
- What is the actual difference between Git and GitHub?
- Walk through the branch → commit → push → PR loop from memory, and explain in one sentence what each step accomplishes.
