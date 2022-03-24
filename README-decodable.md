# Decodable Flink

Decodable Flink is an internal "fork" for Apache Flink.
Learn more about general motivation and proposed standards
[RFC - Decodable Internal Flink “Fork” (DIFF)](https://docs.google.com/document/d/10GWdEyHkOWfKt09fRe2Gva7RkmpQ_32OeqMKhsnUdM8/view).

## Development
We maintain 2 development branches
* Default branch is ``decodable-main``
  * Code changes to ``decodable-main`` is incorporated on all internal flink artifacts and distributions.
  * Build on _decodable-main_ is used for flink distribution used for Decodable pipelines.
* Branch for Decodable Preview is ``decodable-preview-main``
  * ``decodable-preview-main`` is a temporary feature branch for preview until we merge it to ``decodable-main``
  * Code changes to ``decodable-preview-main`` is incorporated in flink artifacts and distributions used for Decodable Preview flink runtime only.
  * ``decodable-preview-main`` also incorporates changes in ``decodable-main`` at the time of latest rebase.
  * Build on _decodable-preview-main_ is used for flink distribution user for Decodable preview cluster.

## Decodable Flink Repo Management

Decodable-Flink repo is a detached mirror of Apache Flink.

### Syncing Decodable-Flink with Apache Flink
Decodable-Flink repo sync (against master) needs to be performed before we upgrade Flink version.
(This can be done independent of flink version upgrade too).

Follow these steps to sync the repo:
* Create a Jira task for sync.
* Get the latest changes from public flink repo.
* Run git diff against public flink repo to confirm no differences.
  * (Take a screenshot as evidence in PR for change merge)
* Push the changes to internal repo.
* Create a PR against ``master`` branch, review & merge.
  * Use `Merge Commit` since we want to preserve commits from public flink repo.

**NOTE:**   Internal flink repo master and public flink repo master should always code-match.

```shell
git remote add public-flink git@github.com:apache/flink.git
git fetch public-flink
git checkout public-flink/master
git checkout -b <BRANCH-SYNC-FLINK>
git diff public-flink/master
git push --set-upstream origin <BRANCH-SYNC-FLINK>
```

* If there are merge conflicts, resolve them by using "ours" merge strategy.
    Note that "ours" merge technically means we are accepting changes coming from public flink repo.
    But since those changes are already in our branch locally, we call it ours (vs origin).
* Verify no diff with public flink repo by running git diff against public flink repo again.
  ```shell
  git checkout <BRANCH-SYNC-FLINK>
  git merge master --strategy=ours
  git diff public-flink/master
  git push    
  ```

### How to upgrade Flink Version
* Follow the procedures above to sync the repo first.
* Create a Jira task for Flink version upgrade.
* Fetch the tags from public repo & push the specific release tag to internal repo.
* Rebase the code to release tag of the upgrade version.
* Push changes to internal repo.
* Create a PR against _decodable-main_, review & merge.
  * Use `Merge Commit` since we want to preserve commits from public flink repo.
* Follow the procedure for _decodable-preview-main_.

####_Procedure to upgrade version - sample git cmds here for version 1.14.4 (from existing version 1.14.3)_
```shell
git checkout decodable-main
git fetch public-flink
git push origin release-1.14.4
git checkout -b <BRANCH-UPGRADE-FLINK>
git rebase --onto release-1.14.4 release-1.14.3 <BRANCH-UPGRADE-FLINK>
git push --set-upstream origin <BRANCH-UPGRADE-FLINK>
```
* This step requires lot of due diligence -b e extra careful in resolving the conflicts here. 
  There may be new changes in the upgraded flink code that may not be compatible with our existing internal commits.

* Verify all necessary decodable changes are applied on top on upgraded version commit.
  ```shell
  git log  
  ```
  * you should see DE-XXXX commits on top of commit for upgraded flink version

####_Procedure to upgrade preview to latest decodable-main - sample git cmds here for version 1.14.4 (from existing version 1.14.3)_
**NOTE:**   Following this sample procedure _decodable-preview-main_, 
            by virtue of _decodable-main_ being on flink version 1.14.4, will upgrade to 1.14.4
            and will get all internal changes to _decodable_main_.
```shell
git checkout decodable-preview-main
git rebase origin/decodable-main
git push -f origin
```

**NOTE:**   There may be opportunity in the future to script-automate these. 
            Currently, I think, we need to do this couple times to feel comfortable with this procedure.
            There are validation steps, that I think, we need to do manual inspection/due diligence until
            we figure out how to automate.

## Decodable Flink builds
* We use github actions for ci/cd to produce build artifacts for all main and feature branches:
  * CI build for all feature branches
  * CD build for _decodable-main_
  * CD build for _decodable-preview-main_
* Feature branch builds are available in s3 path
  [s3://decodable-flink-dist/ci-build/<branch-name>](https://s3.console.aws.amazon.com/s3/buckets/decodable-flink-dist?prefix=ci-build%2F) after successful ci runs.
  * _Sample build for DE-1519 branch & commit head fe81f211c85_
  
  ``s3://decodable-flink-dist/ci-build/DE-1519/decodable-flink-1.14.3-fe81f211c85.tgz``
* Production builds are available in s3 folder after successful cd runs.
  * decodable-main: [s3://decodable-flink-dist/dist/<flink version>](https://s3.console.aws.amazon.com/s3/buckets/decodable-flink-dist?prefix=dist%2F)
    * _Sample flink distribution_ 
    
    ``s3://decodable-flink-dist/dist/1.14.4/decodable-flink-1.14.4-665820d0076.tgz``
  * decodable-preview-main: [s3://decodable-flink-dist/preview-dist/<flink version>](https://s3.console.aws.amazon.com/s3/buckets/decodable-flink-dist?prefix=preview-dist%2F)
    * _Sample flink distribution_
    
    ``s3://decodable-flink-dist/preview-dist/1.14.4/decodable-preview-flink-1.14.4-7e06eb14574.tgz``

## Miscellaneous Operations
#### How to downgrade flink version
This is not a regular operation. If you need to downgrade version create Revert PR on github from the merged PR that upgraded the version
