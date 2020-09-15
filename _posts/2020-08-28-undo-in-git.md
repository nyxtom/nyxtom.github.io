---
title: Undo in Git
published: true
description: Using a combination of git commands like amend, rebase, reset, and reflog can help you fix your commits
tags: [git,productivity,webdev,beginners]
---

Occasionally, perhaps not all that occasionally, you might make a mistake in your commit workflow. You forgot to fix a lint error, your tests didn't pass so you need to fix that first, or you forgot to add a file, maybe you weren't ready for that commit just yet, maybe you didn't mean to push yet. Whatever the case may be, mistakes happen and you need a way to undo whatever those mistakes were. Here are a few easy workflows you can follow when these mistakes happen.

## How to make a change to the previous commit

Let's say you have some code you want to add when you make your first commit. So you type `git status` and:

```shell
nyxtom@enceladus: git status
On branch master

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        index.css
        index.html
        load.js
        load2.js

nothing added to commit but untracked files present (use "git add" to track)
```

You go ahead and add these files with git add.

```shell
nyxtom@enceladus$ git add index.css index.html load.js
~/libvar/map
nyxtom@enceladus$ git status
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   index.css
        new file:   index.html
        new file:   load.js

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        load2.js

~/libvar/map
nyxtom@enceladus$ git commit -m 'Adding initial project'
[master (root-commit) 9e5fc8f2] Adding initial project
 3 files changed, 187 insertions(+)
 create mode 100644 index.css
 create mode 100644 index.html
 create mode 100644 load.js
```

But whoops! Turns out there was a problem in your previous commit. You want to change a line of code. So you make your change to the file and use `git add -p`

```shell
nyxtom@enceladus$ git add -p
diff --git a/load.js b/load.js
index 888a7fbd..f0cc06b3 100644
--- a/load.js
+++ b/load.js
@@ -28,6 +28,8 @@ function setup() {
     let text = [''];

     function draw() {
+        context.clearRect(0, 0, window.innerWidth, window.innerHeight);
+
         let offset = 100;
         let tokens = parse(text);
         tokens.forEach(token => {
(1/1) Stage this hunk [y,n,q,a,d,e,?]? y
```

Instead of making another commit, you can just amend the previous commit! Use `git commit --amend`.

```shell
nyxtom@enceladus$ git commit --amend -m 'Adding initial project'
[master 5923dfe9] Adding initial project
 Date: Fri Aug 28 11:40:45 2020 -0500
 3 files changed, 188 insertions(+)
 create mode 100644 index.css
 create mode 100644 index.html
 create mode 100644 load.js
```

Great! Now we have fixed that last commit with this change!

## How to reset a commit

Now let's say you forgot to add a file, so you go ahead and make another commit like so:

```shell
nyxtom@enceladus$ git add load2.js
~/libvar/map (master)+
nyxtom@enceladus$ st
On branch master
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        new file:   load2.js

~/libvar/map (master)+
nyxtom@enceladus$ git commit -m 'Test'
[master a1bf0355] Test
 1 file changed, 147 insertions(+)
 create mode 100644 load2.js
```

But let's say you don't want that commit yet. Have you run `git push` yet? No? Okay great. You can unstage the last commit at HEAD with the following: `git reset --soft`

```
nyxtom@enceladus$ git reset --soft HEAD^
~/libvar/map (master)+
nyxtom@enceladus$ st
On branch master
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        new file:   load.js
```

Now the previous commit is still staged. Meaning it's ready to be committed again. If you want to unstage it go ahead and type `git restore --staged <file>`

## How to rebase and squash commits

Let's say you made a few commits like the one below.

```shell
nyxtom@enceladus$ git log --oneline
9f44611c (HEAD -> master) Fix tests
2dd82c2b Better comments
5171bfac Fix spacing
ffda820e [#3203] Fixes input validation prior to user api requests
```

You realize that the last **3** commits actually belong to the original bug fix commit. Fear not! You can use `git rebase` for this. Let's rebase the last 4 commits.

```shell
nyxtom@enceladus$ git rebase -i HEAD~4
```

You'll notice that when you rebase with `-i` it is doing so interactively. This means that changes you make in this mode can be done a commit at a time (provided that you are making changes to multiple commits). The editor that pops up will provide you with some options like below:

```
pick ffda820e [#3203] Fixes input validation prior to user api requests
pick 5171bfac Fix spacing
pick 2dd82c2b Better comments
pick 9f44611c Fix tests

# Rebase 5923dfe9..9f44611c onto 9f44611c (4 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message
# x, exec <command> = run command (the rest of the line) using shell
# b, break = stop here (continue rebase later with 'git rebase --continue')
# d, drop <commit> = remove commit
# l, label <label> = label current HEAD with a name
# t, reset <label> = reset HEAD to a label
# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
# .       create a merge commit using the original merge commit's
# .       message (or the oneline, if no original merge commit was
# .       specified). Use -c <commit> to reword the commit message.
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
# However, if you remove everything, the rebase will be aborted.
#
```

Since we are concerned with merging the last 3 commits into the first one, we can change all those `pick` to `fixup`. This will "squash" the commit into the next one in the history and ignore that log message. There are a number of options you can choose from here so I suggest looking through each in detail.

```
pick ffda820e [#3203] Fixes input validation prior to user api requests
fixup 5171bfac Fix spacing
fixup 2dd82c2b Better comments
fixup 9f44611c Fix tests
```

My built-in editor is `vim` so when I type `:wq` the rebase will finish and the following is output to the terminal.

```shell
Successfully rebased and updated refs/heads/master.
```

Great! Now type `git log --oneline` and you should only see a single commit!

```shell
nyxtom@enceladus$ git log --oneline
40122c65 (HEAD -> master) [#3203] Fixes input validation prior to user api requests
```

## How to rebase and then push those changes again

Let's say you are working on a branch for a pull request and you already pushed to the remote branch with `git push`. You made some changes, undid the commit history with `git rebase`. Unfortunately, now your commit tree is out of sync with the remote commit history. Well you can always `git push --force`. This will force the remote history to match your current history.

What happens if someone else has made remote changes? You can actually run `--force-with-lease` to ensure you don't override someone else's changes. It's not very well know, but it's definitely useful!

## How to undo a git rebase

Let's say you messed up your rebase and you need to fix what you did and get back to a previous state. Commits are never really lost in git so you can use `git reflog` to take a look at the commit history prior to the rebase.

```shell
nyxtom@enceladus$ git reflog
40122c65 (HEAD -> master, origin/master) HEAD@{0}: checkout: moving from 8115f1e263eeeee4bf8f75e90bbb6cca1a78cfbe to master
8115f1e2 HEAD@{1}: checkout: moving from master to 8115f1e263eeeee4bf8f75e90bbb6cca1a78cfbe
40122c65 (HEAD -> master, origin/master) HEAD@{2}: rebase (finish): returning to refs/heads/master
40122c65 (HEAD -> master, origin/master) HEAD@{3}: rebase (fixup): [#3203] Fixes input validation prior to user api requests
0b378bf2 HEAD@{4}: rebase (fixup): # This is a combination of 3 commits.
e3dc38ab HEAD@{5}: rebase (fixup): # This is a combination of 2 commits.
ffda820e HEAD@{6}: rebase (start): checkout HEAD~4
9f44611c HEAD@{7}: commit: Fix tests
2dd82c2b HEAD@{8}: commit: Better comments
5171bfac HEAD@{9}: commit: Fix spacing
ffda820e HEAD@{10}: commit: [#3203] Fixes input validation prior to user api requests
```

Great! Let's move back to the commit just right before that rebase happened.

```shell
nyxtom@enceladus$ git reset 9f44611c
nyxtom@enceladus$ git log --oneline
9f44611c (HEAD -> master) Fix tests
2dd82c2b Better comments
5171bfac Fix spacing
ffda820e [#3203] Fixes input validation prior to user api requests
5923dfe9 Adding initial project
```

Great! You're back in business!

## Conclusion

If you ever run into trouble, know that all is literally **not lost** forever. You can even use `git fsck --lost-found` if you find yourself in a situation where you have orphan commits that aren't actually attached to the commit tree.
