---
layout: post
title:  git cheatsheet
date:   2018-01-21 11:18:00 +0200
categories: cheatsheets
---
## general things
- initialize repo: `git init`
- add remote origin to existing repo: `git remote add origin {remote_url}`
- clone in new directory existing repo `git clone {remote_url}`
- add all files: `git add .`
- commit changes: `git commit -m "{message}"`
- commit changes without message (conflicts): `git commit --no-edit`
- push (with setup): `git push --set-upstream origin master`
- get from remote repo and merge: `git pull`: (`git pull origin master`, `git pull` is made up from `git fetch` and `git merge`)
- get from remote repo: `git fetch` (the pulled "results" will be "saved" in `remotes/{remote_name}/{remote_branch_name}` e.g. `remotes/origin/master`, plus you can specify specific remote repo to fetch)

### selecting commits
- current commit `HEAD` or `branch_name`, or `{specific commit's hash}`
- two commits back `HEAD~2` or `branch_name~2`, or `{specific commit's hash}`

## looking at things / searching

#### list of branches `git branch`
- look at list of local branches `git branch`
- look at all existing branches (also remote) and see the last commits in them `git branch -av`
- look at list of remote branches `git branch -r`

#### what is changed `git status`
- look at whats changed since last commit: `git status`

#### what differences there are `git diff`
- look at changes between current situation (only tracked files - not new) and what was in last commit: `git diff`
- look at changes between current situation (only tracked files - not new) and what was in specific commit: `git diff {commit's hash}`
- look at changes between current situation (only tracked files - not new) and what was in last commit for specific file: `git diff {file}`
- look at changes between multiple and older commits `git diff {older_hash or HEAD~x} {newer_hash or HEAD~x}`
- look at changes that are not pushed yet (staged) in staging area from previous commit `git diff --staged`
- look at changes between current situation (only tracked files - not new) and what was in last commit in side by side comparison: `git difftool`

#### look and change previous commits `git log`
- look at all the commits `git log`
- search in commits by messages `git log --grep="{search_text}"`
- look at commits and changes in commits `git log -p`
- look at last n commits (not all of them as in just `-p` case): `git log -p -n {n}`
- If you know the time period then you can use a time period to between commits before a particular date using `--since="2 weeks ago"` and `--until="1 day ago"`.
- get simple one-line-per-each output with hash and title for each of commits `git log --oneline`
- get extra info about which commit is oneline and where are you: `git log --decorate --oneline`
- get all commits in correct order for you with simplified lines and titles: `git log --all --decorate --oneline`
- no merge notifications in commits log - pass `-m` option

#### look at changes inside one commit `git show`
- look at whats changed in last commit `git show`
- look at whats changed in specific commit `git show {commit's hash}`

#### look at who is done the change `git blame`
- look in which commitments the file has changes, what has changed and who changed it `git blame {file_name}`
- look in which commitments the file has changes, what has changed and who changed it for just some specific lines: `git blame -L {line_from},{line_to} {file_name}`

## changing things

#### changing unpublished things
- change last commit that is unpublished `git commit --amend`

#### reset changes to specific commit `git reset`
- reset changes from staging area (when already added) to working area: `git reset`
- reset the HEAD (current commit) to specific commit (and preserve changes as local uncommited changes): `git reset {commit's_hash} {optional_file}`
- reset the HEAD to last commit's data (and preserve changes as local uncommited changes): `git reset HEAD .`
- reset the HEAD to two commits from last data (and preserve changes as local uncommited changes): `git reset HEAD~2 .`
- reset the HEAD to specific commit (and discard all changes since then): `git reset --hard {commit's_hash}`
- reset the HEAD to specific commit (and preserve all uncommited local changes): `git reset --keep {commit's hash}`

#### revert to some commit (undo commit) `git revert` (bad practice if reverting to already published comments)
- revert last commit: `git revert`
- revert a commit without having to edit commit file: `git revert HEAD --no-edit`
- revert to two commits back: `git revert HEAD...HEAD~2`

#### make changes in this commit (switch to working directory or switch between branches) `git checkout`
- will replace everything in working directory with last commits content: `git checkout .`
- switch to working in some specific branch: `git checkout {branch_name}`
- create new branch and switch to it `git checkout -b {new_branch_name}`
- discard all changes in file since commit `git checkout {HEAD or commit's hash} {file_name}`
- make decisions of conflicts - just use your local file as true: `git checkout --ours {file_name}`
- make decisions of conflicts - just use remotes file as true: `git checkout --theirs {file_name}`

#### make changes merging two branches `git merge`
- merge changes: `git merge {branch_name_or_repo_to_merge} {branch_name_or_repo_into_what_to_merge}` (e.g. `git merge remotes/{remote_name}/{remote_branch_name} master` after `git fetch`)
- use special mergetool for manual editing: `git mergetool`

#### branching shenanigans `git branch`
- create new branch `git branch {new_branch_name}`
- create new branch from another branch `git branch {new_branch_name} {starting_branch_name}`
- delete local branch `git branch -d {branch_name}`

#### change commit history, spelling, order, edc. `git rebase`
- open interactive edit panel `git rebase --interactive --root` (`--root` allows to edit all commits in history - including first one)
- open interactive edit panel and edit last n of commits `git rebase --interactive HEAD~{n}`


## recipes

### possible problem while merging
1. `git pull` (or `git merge`) gets merging conflict (it will tell conflict files)
2. fix the conflict:
  - manual fix (either of cases will be OK):
    - mergetool and merge manually `git mergetool`
    - edit file manually
  - automatic fix (either of cases will be OK):
    - make decisions of conflicts - just use your local file as true: `git checkout --ours {file_name}`
    - make decisions of conflicts - just use remotes file as true: `git checkout --theirs {file_name}`
3. add changes to staging
4. commit changes

### merging one branch into another (this case development into master)
1. you are in development branch
2. go to master branch `git checkout master`
3. merge it `git merge development`
4. if needed - destroy development branch `git branch -d development`

### splitting commit (bad practice if commit is published - only locally)
1. open interactive rebase options for last commit (this example): `git rebase --interactive HEAD~1` and switch commits to edit to edit mode
2. add to staging first par of changes: `git add {filename a}`
3. commit those changes: `git commit -m "{a type changes title}"`
4. add to staging second par of changes: `git add {filename b}`
5. commit those changes: `git commit -m "{b type changes title}"`
6. finish the process `git rebase --continue`
