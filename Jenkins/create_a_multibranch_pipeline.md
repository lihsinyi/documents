# Branch Sources
## Configures
### Github
* set Credentials
* Repository HTTPS URL
### Behaviours
It can discover branches, pull requests, and tags with filters.
Jenkins can merge the target branch by using `Discover pull requests from origin`, so we don't need to write the merge code as below in the script:
```
sh """
    git merge origin/main
"""
```

## Issues
### Issue - `merge: origin/main - not something we can merge`
```
sh """
    git merge origin/main
"""
```
When using `Discover branches` with the above code in the script, and getting an error `merge: origin/main - not something we can merge`,
it doesn't work by adding `git fetch origin main` (it will throw `fatal: could not read Username for 'https://github.com': No such device or address`)
the solution is adding `Advanced clone behaviors` and enabling `Fetch tags`, so Jenkins can fetch all refspecs.
I guess adding `Specify ref specs` might work.

|Advanced clone behaviours / Fetch tags|the SCM checkout command|
|-|-|
| [ ] | `/usr/bin/git fetch --no-tags --force --progress -- https://github.com/my/repo.git +refs/heads/my_branch:refs/remotes/origin/my_branch # timeout=10` |
| [x] | `/usr/bin/git fetch --tags --force --progress -- https://github.com/sifive/perfsight.git +refs/heads/*:refs/remotes/origin/* # timeout=10` |

### A branch can only be applied by the first matched branch source, even if the branch is filtered out.
if 2 branch sources discover branches, and the first branch source has a filter, all branches will be applied by the first branch.
