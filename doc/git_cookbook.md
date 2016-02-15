# Git Cookbook

A collection of git recipes to do common git tasks.

See also [Git Tips](git_tips.md).

[TOC]

## Introduction

This is designed to be a cookbook for common command sequences/tasks relating to
git, git-cl, and how they work with chromium development. It might be a little
light on explanations.

If you are new to git, or do not have much experience with a distributed version
control system, you should also check out
[The Git Community Book](http://book.git-scm.com/) for an overview of basic git
concepts and general git usage. Knowing what git means by branches, commits,
reverts, and resets (as opposed to what SVN means by them) will help make the
following much more understandable.

## Excluding file(s) from git-cl, while preserving them for later use

Since git-cl assumes that the diff between your current branch and its tracking
branch (defaults to the svn-trunk if there is no tracking branch) is what should
be used for the CL, the goal is to remove the unwanted files from the current
branch, and preserve them in another branch, or a similar.

### Method #1: Reset your current branch, and selectively commit files.

1.  `git log`  See the list of your commits. Find the hash of the last commit
     before your changes.
1.  `git reset --soft abcdef` where abcdef is the hash found in the step above.
1.  `git commit <files_for_this_cl> -m "files to upload"` commit the files you
     want included in the CL here.
1.  `git checkout -b new_branch_name origin/trunk` Create a new branch for the
    files that you want to exclude.
1.  `git commit -a -m "preserved files"` Commit the rest of the files.

### Method #2: Create a new branch, reset, then commit files to preserve

This method creates a new branch from your current one to preserve your changes.
The commits on the new branch are undone, and then only the files you want to
preserve are recommitted.

1.  `git checkout -b new_branch_name` This preserves your old files.
1.  `git log` See the list of your commits. Find the hash of the last commit
    before your changes.
1.  `git reset --soft abcdef` Where abcdef is the hash found in the step above.
1.  `git commit <files_to_preserve> -m "preserved files"` Commit the found files
    into the `new_branch_name`.

Then revert your files however you'd like in your old branch. The files listed
in step 4 will be saved in `new_branch_name`

### Method #3: Cherry pick changes into review branches

If you are systematic in creating separate local commits for independent
changes, you can make a number of different changes in the same client and then
cherry-pick each one into a separate review branch.

1.  Make and commit a set of independent changes.
1.  `git log`  # see the hashes for each of your commits.
1.  repeat checkout, cherry-pick, upload steps for each change1..n
    1.  `git checkout -b review-changeN origin` Create a new review branch
        tracking origin
    1.  `git cherry-pick <hash of change N>`
    1.  `git cl upload`

If a change needs updating due to review comments, you can go back to your main
working branch, update the commit, and re-cherry-pick it into the review branch.

1.  `git checkout <working branch>`
1.  Make changes.
1.  If the commit you want to update is the most recent one:
    1.  `git commit --amend <files>`
1.  If not:
    1.  `git commit <files>`
    1.  `git rebase -i origin`  # use interactive rebase to squash the new
        commit into the old one.
1.  `git log`  # observe new hash for the change
1.  `git checkout review-changeN`
1.  `git reset --hard`  # remove the previous version of the change
1.  `cherry-pick <new hash of change N>`
1.  `git cl upload`

## Sharing code between multiple machines

Assume Windows computer named vista, Linux one named penguin.
Prerequisite: both machine have git clones of the main git tree.

```shell
vista$ git remote add linux ssh://penguin/path/to/git/repo
vista$ git fetch linux
vista$ git branch -a   # should show "linux/branchname"
vista$ git checkout -b foobar linux/foobar
vista$ hack hack hack; git commit -a
vista$ git push linux  # push branch back to linux
penguin$ git reset --hard  # update with new stuff in branch
```

Note that, by default, `gclient sync` will update all remotes. If your other
machine (i.e., `penguin` in the above example) is not always available,
`gclient sync` will timeout and fail trying to reach it. To fix this, you may
exclude your machine from being fetched by default:

    vista$ git config --bool remote.linux.skipDefaultUpdate true

## Reverting and undoing reverts

Two commands to be familiar with:

*   `git cherry-pick X` -- patch in the change made in revision X (where X is a
    hash, or HEAD~2, or whatever).
*   `git revert X` -- patch in the **inverse** of the change made.

With that in hand, say you learned that the commit `abcdef` you just made was
bad.

Revert it locally:

```shell
git checkout origin   # start with trunk
git show abcdef       # grab the svn revision that abcdef was
git revert abcdef
# an editor will pop up; be sure to replace the unhelpful git hash
# in the commit message with the svn revision number
```

Commit the revert:

```shell
# note that since "git svn dcommit" commits each local change separately, be
# extra sure that your commit log looks exactly like what you want the tree's
# commit log to look like before you do this.
git log          # double check that the commit log is *exactly* what you want
git svn dcommit  # commit to svn, bypassing all precommit checks and prompts
```

Roll it forward again locally:

```shell
# go back to your old branch again, and reset the branch to origin, which now
# has your revert.
git checkout mybranch
git reset --hard origin


git cherry-pick abcdef  # re-apply your bad change
git show                # grab the rietveld issue number out of the old commit
git cl issue 12345      # restore the rietveld issue that was cleared on commit
```

And now you can continue hacking where you left off, and since you're reusing
the Reitveld issue you don't have to rewrite the commit message. (You may want
to go manually reopen the issue on the Rietveld site -- `git cl status` will
give you the URL.)

## Retrieving, or diffing against an old file revision

Git works in terms of commits, not files. Thus, working with the history of a
single file requires modified version of the show and diff commands.

```shell
# Find the commit you want in the file's commit log.
git log path/to/file
# This prints out the file contents at commit 123abc.
git show 123abc:path/to/file
# Diff the current version against path/to/file against the version at
# path/to/file
git diff 123abc -- path/to/file
```

When invoking `git show` or `git diff`, the `path/to/file` is **not relative the
the current directory**. It must be the full path from the directory where the
.git directory lives. This is different from invoking `git log` which
understands relative paths.

## Checking out pristine branch from git-svn

In the backend, git-svn keeps a remote tracking branch that points to the the
commit tree representing the svn repository. The name of this branch is
configured during `git svn init`. The git-svn remote branch is often named
`origin/trunk` for Chromium, and `origin/master` for WebKit.

If you want to checkout a "fresh" branch, you can base it directly off the
remote branch for svn.

    git checkout -b fresh origin/trunk  # Replace with origin/master for webkit.


To find out what your git-svn remote branch name is, you can examine your
`.git/config` file and look for the `svn-remote` entry. It will look something
like this:

```
[svn-remote "svn"]
        url = svn://svn.chromium.org/chrome
        fetch = trunk/src:refs/remotes/origin/trunk
```

The last line (`fetch = trunk/src:refs/remotes/origin/trunk`), says to make
`trunk/src` on svn into `refs/remote/origin/trunk` in the local git checkout.
Which means, the name of the svn remote branch name is `origin/trunk`. You can
use this branch name for all sorts of actions (diff, log, show, etc.)

## Making your `git svn {fetch,rebase}` go fast

If you are pulling changes from the git repository in Chromium (or WebKit), but
your your `git svn` commands still seem to pull each change individually from
svn, your repository is probably setup incorrectly. Make sure the entries in
your `.git/config` look something like this:

```
[remote "origin"]
        url = https://chromium.googlesource.com/chromium/src.git
        fetch = +refs/heads/*:refs/remotes/origin/*
[svn-remote "svn"]
        url = svn://svn.chromium.org/chrome
        fetch = trunk/src:refs/remotes/origin/trunk
```

Here, `git svn fetch` will update the hash in refs/remotes/origin/trunk as per
the `fetch =` line under `svn-remote`. Similarly, `git fetch` will update the
**same** tag under `refs/remotes/origin`.

With this setup, `git fetch` will use the faster git protocol to pull changes
down into `origin/trunk`. This effectively updates the high-water mark for
`git-svn`. Later invocations of `git svn {find-rev, fetch, rebase}` will be be
able to skip pulling those revisions down from the svn server. Instead, it
will just run a regex over the commit log in `origin/trunk` and parse all the
`git-svn-id` lines. To rebuild the mapping. Example:

```
commit 016d28b8c4959a3d28d2fbfb4b86c0361aad74ef
Author: mpcomplete@chromium.org <mpcomplete@chromium.org@0039d316-1c4b-4281-b951-d872f2087c98>
Date:   Mon Jul 19 19:09:41 2010 +0000

    Revert r42636. That hack is no longer needed now that we removed the compact
    location bar view.

    BUG=38992

    Review URL: http://codereview.chromium.org/3036004

    git-svn-id: svn://svn.chromium.org/chrome/trunk/src@52935 0039d316-1c4b-4281-b951-d872f2087c98
```

Will be parsed to map svn revision r52935 (on Google Code) to commit
016d28b8c4959a3d28d2fbfb4b86c0361aad74ef. The parsing will generate a lot of
lines that look like `rXXXX = 01234ABCD`. It should generally take a minute or
so when doing an incremental update.

For this to work, two things must be true:

*   The svn url in the `svn-remote` clause must exactly match the url in the
    git-svn-id pulled form the server.
*   The fetch from origin must write into the exact same branch that specified
    in the fetch line of `svn-remote`.

If either of these are not true, then `git svn fetch` and friends will talk to
svn directly, and be very slow.

## Reusing a Git mirror

If you have a nearby copy of a Git repo, you can quickly bootstrap your copy
from that one then adjust it to point it at the real upstream one.

1.  Clone a nearby copy of the code you want: `git clone coworker-machine:/path/to/repo`
1.  Change the URL your copy fetches from to point at the real git repo:
    `git set-url origin http://src.chromium.org/git/chromium.git`
1.  Update your copy: `git fetch`
1.  Delete any extra branches that you picked up in the initial clone:
    `git prune origin`
