# Testing commands

## Test PRs

Make sure you are in the base branch of the PR (mostly develop).
You also aren't allowed to have any not checked in changes.

Then run:

```bash
git fetch origin pull/<PRnumber>/head && git checkout FETCH_HEAD
```

This only works for PRs which don't depend on another PR which hasn't been applied up to now.
For the other case you either need to locally merge the PRs in a temporary one or merge them in a fork on Github.

When you are done with testing the PR just run this to get back to the develop branch:

```bash
git switch develop
```
