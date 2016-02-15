# Introduction

Kai Wang (kaiwang@) and I (Jói Sigurðsson, joi@) experimented with something we called “shift-based development” in the first half of Q1 2013 as a way to work closely together on a componentization that was hard to do in parallel.

I work from Iceland, which is 7 or 8 hours ahead of Kai’s location (Mountain View, California), depending on the time of year (daylight savings time is not observed in Iceland). Kai and I were working on componentizing the Prefs subsystem of Chromium, and it was obvious early on that if we tried to develop in parallel, we would step on each others’ toes very regularly and be forced to do often difficult merges (due to e.g. moving files, renaming things, and so on). The idea came up, since my normal work day ends right around the time Kai’s starts, why not do the development in serial instead. Although I often work an extra hour or two later in the evening, that’s normally for responding to code review requests and email and not so much for coding, so this way we’d be completely free of getting in each others’ way.

The way we implemented this was we set up a bare git repository, and at the end of the day, we would push whatever we were working on to this repository, and let the other know the status of things. This could be a single working branch, or (more often) a pipeline of branches, plus a branch representing the SVN revision we were based off of. To make this work, we were both using the unmanaged git workflow.

The idea was, the next “shift” would pull down the pipeline of branches and continue where the last left off, doing development on the last change in the pipeline if it was incomplete, landing whatever changes could be landed, or starting a new change in the pipeline if all of them were ready for review or ready to land.

One limitation we ran into was that only the owner of an issue in Rietveld could upload a new patch set. To work around this limitation, I added a feature to Rietveld where you can add a COLLABORATOR=xyz@chromium.org line to your change description, which will allow that person to also upload patches and edit the change description (see [Rietveld patch](https://code.google.com/p/rietveld/source/detail?r=a37a6b2495b43e5fdd38292602d933714b7e8ddd)).

In my opinion this was moderately successful. We were probably less productive than we would have been if each of us had been working on completely unrelated things, but certainly more productive than if we had tried to work together on componentizing Prefs in parallel.

With more practice, I think this way of working together could be quite successful. It was also challenging and fun and could be a worthwhile thing to try for folks separated by close to a full working day or more. See details below if you'd like to try.

# Details

The following instructions assume Linux is being used, but should be easily adaptable to other OSes. If you find mistakes in the instructions, please feel free to correct and clarify this Wiki page.

## Setup

On one of our Linux boxes, we set up a bare git repository using [these instructions](http://git-scm.com/book/en/Git-on-the-Server-Setting-Up-the-Server). I'm sure you could also use an existing git service such as github.

Let's assume the IP address of the Linux box hosting the bare
repository is `12.34.56.78`, the Linux user that provides access to the
bare repository is named `gitshift`, and the repository is located at
`/home/gitshift/git/gitshift.git`.

Each developer participating in the shift-based development provides
their RSA public key (e.g. `~/.ssh/id_rsa.pub`) and we add it to
`/home/gitshift/.ssh/authorized_keys`; that way as long as you are using
`ssh-agent` and have run `ssh-add`, you won't need to type in a password
every time you issue a git command that affects the repository.

Issue these commands to add a remote for this repository to your local
git repo, and to mirror its initial state.  You can call it whatever
you like, shiftrepo is just an example.

```
$ git remote add shiftrepo gitshift@12.34.56.78:/home/gitshift/git/gitshift.git
$ git fetch shiftrepo
```

You should now be able to do a `git push` of some dummy branch; try
it out, e.g.

```
$ git checkout -b shifttest master
$ echo boo > boo.txt
$ git add boo.txt && git commit -m .
$ git push shiftrepo
```

The shared repository is just a place where you share your branches;
it is not a place where you do actual work.  The actual work should be
done in your separate local repositories, and you still use e.g. git
pull (which in our git svn repositories behind the scenes does a `git svn fetch` etc.

Shift-based collaboration won't work well (at least not with a
pipeline of branches) unless you are using an "unmanaged" git checkout.

## Example Working Rules

For the branches we collaborated on, we set up some working
rules. This may be a good starting set and it worked well enough for us,
but others could adapt these rules:

a) We had a naming convention for pipelines of branches. E.g. you
might have branches named shiftrepo/p0-movemore and shiftrepo/p1-sync,
where p1-sync's upstream branch is p0-movemore, and p0-movemore's
upstream branch is an SVN revision, generally a version that was LKGR
at some point.  You can find the git commit hash of this revision by
running

```
git log -1 --grep=src@ | head -1 | cut -d " " -f2
```

and the SVN revision number by running

```
git log -1 --grep=src@ | grep git-svn-id | cut -d@ -f2 | cut -d " "  -f1
```

We followed a naming convention of one or two alphabetic characters
followed by the sequence number of the branch, followed by a dash and
a descriptive name for what's going on in that particular branch. The
one or two alphabetic characters indicated the rough over-arching
topic (e.g. p for Prefs), and the stuff after the dash can be more
descriptive.

b) When pushing a pipeline of branches to shiftrepo (where branch A
depends on branch B and so forth) we made sure to first git pull in
each dependent branch in sequence, so that each branch in shiftrepo is
building straight on top of the previous branch.

c) At the end of our shift, we did `git push shiftrepo branchname`
for each branch.

d) At the start of our shift, we did `git fetch shiftrepo` and then
for each branch we were collaborating on we did `git checkout branchname && git merge shiftrepo/branchname`. Note that the first command checks
out the local branch, and the second merges the shiftrepo/ branch into
it. This does not make the shiftrepo/ branch a parent of the local
branch.

e) Also at the start of each shift, we updated the local upstream
branches for each branch to match the upstream relationships that the
person ending their shift had on his end.  One case is if the oldest
branch in the pipeline has been merged to a new LKGR, then we did this:

```
git branch --set-upstream oldestBranchName `git log -1 --grep=src@ oldestBranchName | head -1 | cut -d " " -f2`
```

and the other case is if new branches were created during the last shift, e.g. p4-foo was added, then for each we need to do like this:

```
git branch --set-upstream p4-foo p3-bar
```

f) For managing old branches, we removed the oldest branch in a
pipeline when several conditions were met:

> i) The old branch has been checked in.

> ii) The SVN revision of the check-in of the old branch is equal to
> or older than LKGR, i.e. that change in SVN is included when you
> sync to LKGR.

> iii) That LKGR has been merged into the old branch, and we've done
> `git pull` in the next branch after it.

g) We only used the CQ to commit stuff. The fear was (and this hasn't
really been validated as true or false) that there might be some
gotchas if we used `git cl dcommit` instead.

h) At the end of our shift, we communicated by email/IM/Hangout to let
the other know the status of the work, next steps remaining for any
currently-open branches, and to discuss what might make sense for the
next branches to work on.

## Random Commands

To push a local branch to shiftrepo: `git push shiftrepo localbranchname`

To push all "matching" branches (i.e. push the latest copy of
any local branch that has previously been pushed to shiftrepo): `git push shiftrepo`

To delete a branch from shiftrepo, it's weird: `git push shiftrepo :branchname`
