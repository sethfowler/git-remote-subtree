# git-remote-subtree

A git remote helper that allows you to treat a remote branch as if it were a
copy of a local branch that differed only in the contents of one subtree.

The idea is that hide the functionality of `git-subtree` behind a protocol which
works transparently, doesn't pollute the commit history of the repo, and doesn't
interfere with normal git functionality. Just as a failing escalator becomes
stairs, any issue with `git-remote-subtree` should yield a monorepo that
continues to work perfectly, with no loss of data or commits.

# Proposed approach

Say the user sets up a remote like so:

```
git remote add subRepo subtree::subRepoDir::superRepo/branch::http://example.com/normalRepo
```

This creates a git remote called `subRepo` which wraps a normal git repo at
`http://example.com/normalRepo`. If you fetch a branch `subRepo/subBranch`, what
you get is `normalRepo/subBranch`, but it is as if all of the files are moved
into a single top-level directory `subRepoDir/`, and `subRepoDir` is grafted
into the tree at `superRepo/branch`. If `subRepoDir` already exists in
`superRepo/branch`, the effect is as if its existing contents were replaced by
the contents of `normalRepo/subBranch`. If `subRepoDir` already exists in
`superRepo/branch` and *the contents are exactly the same*, then
`subRepo/subBranch` will have the same SHA as `superRepo/branch`, and pulling
one into the other will be a no-op.

Similarly, pushing from `superRepo/branch` to `subRepo/subBranch` behaves as if
the contents of `subRepoDir` were at the top level, and everything else was
thrown away. If the contents of `subRepoDir` are the same as
`normalRepo/subBranch`, then pushing is a no-op.

This setup means that the functionality of `git-subtree` can be implemented
totally by pushing to and pulling from a `subtree::` remote. Because all the
magic is inside the remote helper, the main repo remains clean, and all other
git functionality works just as you would expect.

This should be fairly simple to implement while preserving history. A hand-wavy
algorithm for fetching is as follows:

- Do a fetch on `normalRepo` and `superRepo` into our hidden repo.

- See if the tree object of the oldest commit in `normalRepo/subBranch` is
  present in the local repo. If not, we know that we've never merged this branch
  in before, and we can skip some of the following work.

- Walk backwards in the commit graph from `normalRepo/subBranch` until we find a
  tree object that's in the local repo. See if one of the associated commits in
  the local repo is the same in every other respect except for the parent
  commits and the fact that the tree is rewritten. If so, this is the last
  common commit. If not, keep walking backwards; if we run out of commits, we've
  never merged this branch in before, and in the steps below we can just start
  at the current commit in the local repo.

- Create a temporary branch in our hidden repo pointing at the version of the
  common commit in our local repo.

- Cherry-pick commits from our local repo until we either hit a commit that
  modifies the tree object for `subRepoDir` (which we won't cherry-pick), or we
  run out of commits.

- Cherry-pick all of the commits from `normalRepo/subBranch` onto our temporary
  branch, with the tree rewritten appropriately. We know they'll apply cleanly
  because the state of `subRepoDir` in our temporary repo is clean with respect
  to `normalRepo`.

- The temporary repo contains the data we'll return from the fetch. Repeat as
  necessary for the other branches.

This is obviously quite expensive, so in practice we'll want to cache some
information to speed this up.

Pushing is a bit simpler; once we find the common commit we just need to
transform each commit in our local repo that touches `subRepoDir` into a
corresponding commit on `normalRepo/subBranch`.

It might sound like there's a lot to implement here, but actually
`git-subhistory` in particular is fairly close to what's needed here, and
translating it into e.g. Python would get us 80% of the way there.

# Resources and related work

[How to Write a New Git Protocol](https://rovaughn.github.io/2015-2-9.html)

[Mastering Git Subtrees](https://medium.com/@porteneuve/mastering-git-subtrees-943d29a798ec#.us0rtft89)

[git-subtree docs](https://raw.githubusercontent.com/git/git/master/contrib/subtree/git-subtree.txt)

[git-subhistory](https://github.com/laughinghan/git-subhistory)

[git-subrepo](https://github.com/ingydotnet/git-subrepo)

[Which commit has this blob?](http://stackoverflow.com/questions/223678/which-commit-has-this-blob)
