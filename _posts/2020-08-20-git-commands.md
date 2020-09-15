---
title: Git Commands
published: true
description: If you're getting started to development and git, learn about these top git commands for your daily use
tags: [git,productivity,beginners]
---

In this article, we're going to go over my top 10 git commands I use almost every day. If you're new to programming, or just getting familiar with git, then I highly recommend becoming familiar with a few of these commands. There are some great GUIs out there, but nothing beats learning the command line.

## git status

> `git status` will display the difference between the index and the current HEAD commit, paths that have differences between the working tree and the index file, and paths in the working tree that are not yet tracked. [Docs](https://git-scm.com/docs/git-status)

```
On branch master
Your branch is up to date with 'origin/master'.

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   todo.md
```

You'll use this one so much that I would recommend creating an alias for it in your `~/.bashrc` file with:

```
alias st='git status'
```

## git add

> `git add` updates the index using the current content found in the working tree, to prepare the content staged for the next commit. It typically adds the current content of existing paths as a whole, but with some options it can also be used to add content with only part of the changes made to the working tree files applied, or remove paths that do not exist in the working tree anymore [docs](https://git-scm.com/docs/git-add)

I frequently use the `-p` option to stage only parts of a file instead of the entire changeset when I have multiple things I've done in a single commit. It's helpful when you want to split up your work into multiple commits.

## git commit

> `git commit` will create a new commit containing the current contents of the index and the given log message describing the changes. The new commit is a direct child of HEAD, usually the tip of the current branch, and the branch is updated to point to it [docs](https://git-scm.com/docs/git-commit)

```
git commit -m 'Message'
```

If you don't want to open up the built-in editor (aka Vim for me), you can use `-m` to inline your message. Since I frequently have commit [hooks](https://git-scm.com/docs/githooks) I usually let `vim` open up to edit my message. Remember to `:wqa!`

## git diff

> `git diff` will show changes between the working tree and the index or a tree, changes between the index and a tree, changes between two trees, changes resulting from a merge, changes between two blob objects, or changes between two files on disk. [docs](https://git-scm.com/docs/git-diff)

I frequently will use this one to perform diffs between branches as well. `git diff branch1..branch2`. You can also take a diff and output it to a file `git diff > patch.diff`. You can pass that file along and later apply it with `git apply patch.diff`.

## git stash

If you're working on something and you need to store it away temporarily, use `git stash`. You can use `git stash` like a clipboard of sorts and even name it with `git stash -m name`. Later you can apply it with `git stash apply stash^name`. If you aren't naming your stash you can manipulate what's in the stash with `git stash pop`, `git stash list`. [doc](https://www.git-scm.com/docs/git-stash)

## git log

`git log` has loads of options you can use to manipulate what you are looking at. Here are a few I've used that could be helpful [doc](https://www.git-scm.com/docs/git-log)

* `git log --graph``: Draw a text-based graphical representation of the commit history on the left hand side of the output. This may cause extra lines to be printed in between commits, in order for the graph history to be drawn properly
* `git log --format=<format>` or `git log --pretty=<format>`: Pretty-print the contents of the commit logs in a given format, where <format> can be one of oneline, short, medium, full, fuller, reference, email, raw, format:<string> and tformat:<string>. When <format> is none of the above, and has %placeholder in it, it acts as if --pretty=tformat:<format> were given.
* `git log -c`: With this option, diff output for a merge commit shows the differences from each of the parents to the merge result simultaneously instead of showing pairwise diff between a parent and the result one at a time. Furthermore, it lists only files which were modified from all parents.

## git push

> Updates remote refs using local refs, while sending objects necessary to complete the given refs. [doc](https://www.git-scm.com/docs/git-push)

## git pull

Git pull is actually a combination of two commands in git. `git fetch` and `git merge`.

> Incorporates changes from a remote repository into the current branch. In its default mode, git pull is shorthand for git fetch followed by git merge FETCH_HEAD.

> More precisely, git pull runs git fetch with the given parameters and calls git merge to merge the retrieved branch heads into the current branch. With --rebase, it runs git rebase instead of git merge. [doc](https://www.git-scm.com/docs/git-pull)

## git checkout

> Switch branch or restore working tree files. Updates files in the working tree to match the version in the index or the specified tree. If no pathspec was given, git checkout will also update HEAD to set the specified branch as the current branch. [doc](https://www.git-scm.com/docs/git-checkout)

`git checkout -b <branch>` is the usual way you will want to checkout a new branch. Otherwise you can also checkout an existing branch with `git checkout <branch>` provided the branch is available and the tree has been fetched already.

`git checkout -f <filename>` you can use this to restore the state of a file or a pattern of files; you can also use the added options in the case of unmerged entries such as (`--ours` or `--theirs`)

## git blame

Git blame is a handy utility for determining what revision and which author is to blame for each line in a file. Many editors have plugins available to display this right into the editor. [doc](https://www.git-scm.com/docs/git-blame)

While there are a number of options, the one I use most is `git blame <file>` but to be perfectly honest the built in git blame commands in Vim or VS Code are much better suited to navigating on the fly.

Git blame can be helpful to lookup where code was moved, how it got there, and in what specific changesets it occurred.

## 🎉 BONUS: git reset

Git reset has three primary forms of resetting the state of the working tree. It comes in the form of the flags `--soft` `--hard` and `--mixed`. [doc](https://www.git-scm.com/docs/git-reset)

* `--soft` Does not touch the index file or the working tree at all (but resets the head to <commit>, just like all modes do). This leaves all your changed files "Changes to be committed", as git status would put it.
* `--hard` Resets the index and working tree. Any changes to tracked files in the working tree since <commit> are discarded.
* `--mixed` Resets the index but not the working tree (i.e., the changed files are preserved but not marked for commit) and reports what has not been updated. This is the default action.

Typically, if I made a mistake in my last commit and I want to undo that change before I go and push to remote I will restage that commit as follows.

`git reset --soft HEAD^`. This will unstage the last commit I made and keep it in the working changes. I will make my changes and recommit. If you don't want to use this approach and you simply want to make changes on top of an existing commit, you can use `git commit --amend` to amend on top of the last commit.

## 🎉 BONUS++: git bisect

If you are having trouble figuring out when a bug was committed and you don't know the exact commit that caused it, then take a look at `git bisect`.

> This command uses a binary search algorithm to find which commit in your project’s history introduced a bug. You use it by first telling it a "bad" commit that is known to contain the bug, and a "good" commit that is known to be before the bug was introduced. Then git bisect picks a commit between those two endpoints and asks you whether the selected commit is "good" or "bad". It continues narrowing down the range until it finds the exact commit that introduced the change. [doc](https://www.git-scm.com/docs/git-bisect)

You do this by running through a series of commands:

* `git bisect start`
* `git bisect bad HEAD` (if current HEAD has the bug)
* `git bisect good v1.0` (commit where everything was good)
* `git bisect reset` (rest the current bisect session)

You can also shorthand the first 3 commands with `git bisect start HEAD HEAD~10` (for last 10 commits is where the bug is somewhere).

## 🎉🎉🎉 Expert Level: [git rebase](https://git-scm.com/docs/git-rebase)

Be careful with this command, but if you are really looking to make adjustments to your commit history, take a look at `git rebase`. I've used `git rebase` when I need to clean up my commit history in my branch before I'm ready to merge. It's really quite useful if you end up having a number of commits that look something like this:

```
47d07f83 Lint
953b4fcb Fix tests
36da7d28 Lint error
2d85a6ab Fix background colors in tmux to match nova
0227a7f5 Fix window/tab navigation in tmux
```

Want to get rid of those last 3 commits but they need to be removed/merged/squashed/reordered? Use `git rebase -i HEAD~5` where `-i` will let you look at the revision history and `HEAD~5` is approximately 5 commits from the HEAD of the tree. This will drop you into an editor to make changes to those commits. You can even make additional edits to an existing commit in this mode and amend with `git commit --amend`. Once you're done here, use `git rebase --continue`.

**Side note:** it's really easy to mess up the commit history this way so I'd recommend reserving this command for only managing your branch commits at times. If you end up revising the commit history and it includes merged commits then the parent tree might end up being out of sync and that's a mess you don't want to get caught in. Make sure you know what you're doing :)

## Conclusion

That's it! It was supposed to be 10 but I added 3 more in there for good measure. There are loads of great git commands and if you are using [GitHub](https://github.com/) you can even take advantage of additional commands with [hub](https://hub.github.com/). Hub will add additional commands to interact directly with GitHub such as browse issues, create a gist, create repositories, or generate pull requests.
