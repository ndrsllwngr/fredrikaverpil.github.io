---
layout: post
title: 'Git gotchas'
tags: [linux, bash, git]
---

Git: Generally Impossible To (remember?) ...

Anyways, some handy git tricks in this post.

<!--more-->

## Branch management

When you've accidentaly screwed up your local `master`, make it good again by resetting it into whatever is in `origin/master`:

```bash
$ git checkout -B master origin/master && git pull
```

Or reset your current branch to the one in origin:

```bash
$ git reset --hard origin/<your_branch>
```

Delete all branches which have been merged (except e.g. master, main, dev):

```bash
# Show a list of merged branches, which can be deleted
$ git branch --merged | egrep -v "(^\*|master|main|dev)" | xargs echo

# Delete the merged branches
$ git branch --merged | egrep -v "(^\*|master|main|dev)" | xargs git branch -d
```

## Search (using super fast grep)

Free text search throughout any git commit message:

```bash
$ git --no-pager log --regexp-ignore-case --grep <regexp>
```

Find a commit based on free text search of code:

```bash
$ git rev-list --all | xargs git --no-pager grep --color=never --extended-regexp --ignore-case <regexp>
```

Free text search in current code:

```bash
git --no-pager grep --ignore-case <regexp>
```

## Rebasing

### Better rebase?

Rebase that for some reason works more often than `git rebase`:

```bash
$ git pull --rebase origin master
```

The `git pull` is a shorthand for `git fetch` and `git merge`. The above command instead does a `git fetch` followed by a `git rebase`.

### Solving a complex rebase conflict

When I have a complex rebase conflict to solve, I like to squash all commits I've added in my private branch into one single commit. This makes it easier to deal with solving the conflict, in comparison with having to solve it for multiple commits.

Then, if the rebase conflict is still difficult to solve or maintain, I usually remove the files with the complex conflict from the commit, perform the rebase and manually add the removed files back on top of `HEAD`. During the last step, it's important to diff the files carefully so they do not contain wrong/old code.

#### Step 1. Interactive rebase with squash

Many thanks to [Erik Thorsell](https://erikthorsell.github.io) for showing me interactive rebase!

Get the commit hash of the start of the branch. There are a couple of ways to get this. You can simply git log and count your commits, or

```bash
$ git log --graph --decorate --pretty=oneline --abbrev-commit

* 8e2ba60 (HEAD -> add-post-on-git-gotchas) Update post
* 8dcb8ae Update post
* a19e1f3 Update post
* 0f4bdfb Create post
* fc7be40 (origin/master, origin/HEAD, master) Update about.md
* 8213989 Fix tags
* 350bb6c Add post on git
```

In the example above, the start of the branch is the commit with hash `fc7be40` and there were a total of 4 commits made since branching off. We can then squash the commits either by defining the commit sha where we branched off:

```bash
$ git rebase -i fc7be40
```

Your editor of choice (defined in `~/.gitconfig`) should open a `git-rebase-todo` file and you should see something like this:

```bash
pick 0f4bdfb Create post
pick a19e1f3 Update post
pick 8dcb8ae Update post
pick 8e2ba60 Update post

# Rebase fc7be40..8e2ba60 onto 8e2ba60 (4 commands)
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
# Note that empty commits are commented out
```

Edit the commits you want to squash. The example below shows how to squash all four commits into one:

```bash
pick 0f4bdfb Create post
s a19e1f3 Update post
s 8dcb8ae Update post
s 8e2ba60 Update post
```

Save the file and close it. A new file will open, with the commit message to use for this new commit. After saving and closing it, the `git rebase` command will finish and the commits were quashed into one commit. Note that the hash for the squashed commit is brand new.

We have, however, only rebased on top of `fc7be40`. Not on top of `origin/master`.

Run `git log` again to see what it looks like:

```bash
$ git log --graph --decorate --pretty=oneline --abbrev-commit

* 65d6e04 (HEAD -> add-post-on-git-gotchas) Create post
* fc7be40 (origin/master, origin/HEAD, master) Update about.md
* 8213989 Fix tags
* 350bb6c Add post on git
* b3fcc7d Rename post
```

Now you will have to push this to your private branch using `git push --force`, because you have effectively a completely new branch here, which you need to replace the old one with.

You can now go ahead with a `git rebase master` to rebase the squashed commit on top of master, which is hopefully a bit easier to handle now.

#### Step 2. Removing files from a commit

If you for some reason still have a very complex conflict to solve, you can completely remove files from your commit which will allow you to rebase without major issues.

Begin by resetting the branch to sit at where you originally started your branch, but keep all your changes as staged files. Taking the previous example, I will reset to `fc7be40 (origin/master, origin/HEAD, master) Update about.md`:

```bash
$ git reset --soft fc7be40
```

Now you can carefully unstage the changes which are giving you a hard time. In my case I will take [Poetry's](https://python-poetry.org/docs/basic-usage/#installing-with-poetrylock) lockfile (`poetry.lock`) as an example:

```bash
$ git reset HEAD poetry.lock
```

Repeat this for all files you wish to unstage from your commit.

I will then re-create my original commit from all the files which are _staged_:

```bash
$ git commit -m "Create post"
```

#### Step 3. Perform the rebase

Now, the rebase is hopefully a lot easier to manage:

```bash
$ git rebase master
```

Sort out any conflicts with `git rebase --continue` or `git rebase --skip`, like usual...

Then finally, when the rebase is completed, add the file changes back in which were to complicated to solve previously with e.g. `git add <file>` and finally a `git commit --amend` if you want to amend the final changes to the previous commit.

To push the final result, you must perform a `git push --force` since you have changed the conents of your branch to such an extent that you effectively are looking at a brand new branch (but with the same name as before).
