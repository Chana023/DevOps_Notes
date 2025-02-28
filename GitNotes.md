# Git Notes

## Clone a bare git repositry

`git clone --bare <repository-url> <directory-name>`

Cloning bare repos is useful for:
- Creating a central repo on a server 
- Setting up a mirror of another repositry
- Taking back ups of a repositry

---

## Basic Commands

### Repo Set up and Configuration

- `git init`                        Create a new git repositry
- `git clone <url>`                 Clone a repo
- `git config user.name`            Set or view username
- `git config user.email`           Set of view email

---

### Branching and Merging

- `git branch`                      List branches
- `git branch <branch-name>`        Create a new branch
- `git branch <branch-name>`        Create a new branch
- `git checkout <branch>`           Switch to a branch
- `git checkout -b <branch>`        Create and switch to a new branch
- `git merge <branch>`              Merge a branch into the current branch
- `git rebase <branch>`             Reapply commits on top of another branch

---

### Remote Repositry commands

- `git remote -v`                   List remote repositories
- `git remote add <name> <url>`     Add a remote repository
- `git fetch <remote>`              Download objects and refs from remote
- `git pull`                        Fetch and integrate with current branch
- `git push <remote> <branch>`      Update remote refs and objects

---

### History and State

- `git log`                         Show commit logs
- `git log --oneline`               Show commit history in compact format
- `git show <commit>`               Show commit details
- `git reset <file>`                Unstage a file
- `git reset --hard HEAD`           Discard all local changes
- `git revert <commit>`             Create new commit that undoes changes

---

### Stashing

- `git stash`                        Save changes that aren't ready to commit
- `git stash pop`                    Apply stashed changes and remove from stash
- `git stash list`                   List stashed changes

---

## Advanced Commands