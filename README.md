# Eigen fork for Insight Toolkit (ITK)

This project is a fork of the Eigen3 source hosted at https://gitlab.com/libeigen/eigen.git.

This project contains changes required to embed Eigen into ITK. This
includes changes made primarily to the build system to allow it to be embedded
into another source tree as well as a header to facilitate mangling of the
symbols to avoid conflicts with other copies of the library within a single
process.

# What is the branch naming convention?

Each branch is namged with the following pattern `for/itk-eigen-X-SHA{7}`

where:
- `X` is the upstream version of the forked project
- SHA{7}

Additional documentation on developing ITK see: https://docs.itk.org/en/latest/contributing/index.html

# Workflow for Updating



## Setup repositories
    1. Clone (or reuse) the fork and ensure the tree is clean:
        git clone git@github.com:InsightSoftwareConsortium/eigen.git 
        cd eigen
    2. Wire remotes:
        git remote add upstream https://gitlab.com/libeigen/eigen.git
    3. Fetch:
        git fetch origin
        git fetch upstream

## Create a new branch following the convention

4. Fetch the new upstream Eigen release tag and define workflow variables:

```
# Set the upstream Eigen release tag to update to (e.g. 3.4.0 or 5.0.1)
ITK_EIGEN_TARGET_TAG=5.0.1

git fetch upstream refs/tags/${ITK_EIGEN_TARGET_TAG}:refs/tags/${ITK_EIGEN_TARGET_TAG}

# Extract version from Eigen/Version at the target tag
MAJOR=$(git show refs/tags/${ITK_EIGEN_TARGET_TAG}:Eigen/Version | grep 'EIGEN_MAJOR_VERSION' | awk '{print $3}')
MINOR=$(git show refs/tags/${ITK_EIGEN_TARGET_TAG}:Eigen/Version | grep 'EIGEN_MINOR_VERSION' | awk '{print $3}')
PATCH=$(git show refs/tags/${ITK_EIGEN_TARGET_TAG}:Eigen/Version | grep 'EIGEN_PATCH_VERSION' | awk '{print $3}')
XYZ="${MAJOR}.${MINOR}.${PATCH}"
echo "XYZ [${XYZ}]"

SHA=$(git rev-parse --short refs/tags/${ITK_EIGEN_TARGET_TAG})
echo "SHA [${SHA}]"

NEW_BRANCH="for/itk-eigen-${XYZ}-${SHA}"
echo "NEW_BRANCH [${NEW_BRANCH}]"
```

5. Fetch the prior ITK fork branch and inspect overlay commits to replay:

```
# Set the current ITK fork branch, located in UpdateFromUpstream.sh.
# (e.g. for/itk-eigen-3.3.9-abcdef1)
ITK_CURRENT_BRANCH=for/itk-20260501-879885e1

git fetch upstream
git fetch origin ${ITK_CURRENT_BRANCH}:${ITK_CURRENT_BRANCH}
git log --oneline upstream/master..${ITK_CURRENT_BRANCH}
```

6. Determine the base commit where the ITK overlay patches begin, then branch and
   rebase onto the new upstream release (resolve conflicts as needed):

```
# Find the commit where ITK_CURRENT_BRANCH diverged from upstream.
# Using upstream/master directly as the rebase base is unreliable because
# upstream/master may have advanced since ITK_CURRENT_BRANCH was created.
REBASE_BASE=$(git merge-base ${ITK_CURRENT_BRANCH} upstream/master)
echo "REBASE_BASE [${REBASE_BASE}]"

git checkout -b ${NEW_BRANCH} ${ITK_CURRENT_BRANCH}
git rebase --onto refs/tags/${ITK_EIGEN_TARGET_TAG} ${REBASE_BASE} ${NEW_BRANCH}
```

   If the rebase encounters conflicts that are difficult to resolve, abort it and
   cherry-pick the overlay commits individually instead:

```
git rebase --abort

# List the overlay commits (newest first) from the log output in step 5
# and cherry-pick them oldest-first onto the new base:
git checkout -b ${NEW_BRANCH} refs/tags/${ITK_EIGEN_TARGET_TAG}
git cherry-pick <oldest-overlay-sha> ... <newest-overlay-sha>
```

7. Verify:

```
git log  --oneline refs/tags/${ITK_EIGEN_TARGET_TAG}..HEAD
git diff --stat    refs/tags/${ITK_EIGEN_TARGET_TAG}..HEAD
```

8. Publish the branch (ITK's `UpdateFromUpstream.sh` consumes this branch):

```
git push origin ${NEW_BRANCH}:${NEW_BRANCH}
```

9. In ITK, update `Modules/ThirdParty/Eigen3/UpdateFromUpstream.sh` with the NEW_BRANCH, commit change then run the script.
