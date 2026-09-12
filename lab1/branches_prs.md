# Hands-On Lab: Branches & Pull Requests

**Goal:** By the end, every participant has created a branch, made a change, opened a PR, gotten it reviewed, resolved a merge conflict, and merged it.
**Time:** 10 min
**Requirements:** GitHub account, Git installed, terminal. No coding language needed — everything is plain text/Markdown so nobody gets blocked by setup.

---

## 0. Facilitator Setup (do this once, before the session)

1. Create a new GitHub repo, e.g. `git-pr-lab`, public or org-internal.
2. Add these starter files:

**`team-roster.md`**
```markdown
# Team Roster

| Name | Role | Fun Fact |
|------|------|----------|
```

**`shared-notes.md`**
```markdown
# Shared Notes

Add your notes below this line.

---
```

**`CONTRIBUTING.md`**
```markdown
# How to Contribute

1. Create a branch: `git checkout -b feature/<your-name>-<short-task>`
2. Make your change.
3. Commit: `git commit -m "feat: add <what you did>"`
4. Push: `git push origin feature/<your-name>-<short-task>`
5. Open a Pull Request into `main`.
6. Get one approval before merging.
```

---

## 1. Everyone: Clone and Configure (5 min)

```bash
git clone https://github.com/<org>/git-pr-lab.git
cd git-pr-lab
git config user.name "Your Name"
git config user.email "you@example.com"
```

---

## 2. Exercise A — Solo Branch + PR (15 min)

Everyone does this in parallel; no conflicts because each person edits a new row.

```bash
git checkout main
git pull origin main
git checkout -b feature/<yourname>-add-to-roster
```

Open `team-roster.md` and add one row for yourself:

```markdown
| Moora | Backend Dev | Once debugged prod at 2am in pajamas |
```

```bash
git add team-roster.md
git commit -m "feat: add <yourname> to team roster"
git push origin feature/<yourname>-add-to-roster
```

Go to GitHub → **Open a Pull Request** → base `main` ← compare `feature/<yourname>-add-to-roster`.
- Add a short description
- Assign a reviewer (pair people up)

**Reviewer task:** open the PR, leave one comment (even a nitpick like "typo" or "nice!"), then **Approve**.

**Author task:** once approved, **merge** the PR (use "Squash and merge" — explain why: keeps `main` history clean, one commit per feature).

**Debrief question for the room:** "What happened to your branch after merge?" → introduce `git branch -d` / deleting the remote branch via GitHub's "Delete branch" button after merge.

---

## 3. Exercise B — Deliberate Merge Conflict (15–20 min)

This is the important one — most people never practice conflict resolution until it happens for real, under pressure.

Pair up participants (A and B). Both start from the same point:

```bash
git checkout main
git pull origin main
```

**Person A:**
```bash
git checkout -b feature/a-update-notes
```
Edit `shared-notes.md`, change the line right after `Add your notes below this line.` to:
```
Person A says: standups are at 9am.
```
```bash
git add shared-notes.md
git commit -m "feat: add standup time note"
git push origin feature/a-update-notes
```
Open a PR and **merge it immediately** (skip review for speed here).

**Person B (started before A merged, doesn't know about A's change yet):**
```bash
git checkout -b feature/b-update-notes
```
Edit the *same line* in `shared-notes.md` to:
```
Person B says: standups are at 10am, not 9.
```
```bash
git add shared-notes.md
git commit -m "feat: correct standup time note"
git push origin feature/b-update-notes
```
Open a PR into `main`. GitHub will show: **"This branch has conflicts that must be resolved."**

### Resolve it locally:
```bash
git checkout feature/b-update-notes
git fetch origin
git merge origin/main
```
Git will show a conflict in `shared-notes.md`:
```
<<<<<<< HEAD
Person B says: standups are at 10am, not 9.
=======
Person A says: standups are at 9am.
>>>>>>> origin/main
```
Edit the file to resolve (decide together — this is the point: a conflict is a conversation, not just a mechanical fix):
```
Standups are at 9am (confirmed).
```
```bash
git add shared-notes.md
git commit
git push origin feature/b-update-notes
```
Refresh the PR on GitHub — conflicts are gone, ready to merge. Get it approved and merge.
