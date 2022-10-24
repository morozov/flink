# Decodable Flink

Decodable Flink is an internal "fork" for Apache Flink.
Learn more about general motivation and proposed standards
[RFC - Decodable Internal Flink “Fork” (DIFF)](https://docs.google.com/document/d/10GWdEyHkOWfKt09fRe2Gva7RkmpQ_32OeqMKhsnUdM8/view).

## Decodable Flink Repo Management

We mirror Flink's branching and version management scheme
* The default branch is `decodable-main` (from `flink/master`). 
  Its version is typically the next version of Flink we are planning to use at decodable. The purpose of this branch is
  to have an up-to-date development snapshot available, and to make sure our custom patches against upstream Flink are
  compatible.
  Whenever there's a need, we rebase from the latest `flink/master` and force-push to this branch.
* we have a branch for each minor Flink version that is in use at decodable, e.g. `decodable-flink-1.15` (branch) with `1.15-SNAPSHOT` (version).
    * Each release is based on a `decodable-flink-x.y` branch, with a tag `decodable-flink-1.15.1-deco1` 
      pointing to a commit with the Flink  version, e.g. `1.15.1-deco1`. 
      For setting the Flink version: `cd tools; NEW_VERSION=1.14.4-deco2 ./releasing/update_branch_version.sh`
    * The `-deco1` suffix allows us to release multiple decodable versions per Flink release. Also, the name is distinguishable from Flink releases,
      to make sure we are not accidentally mixing upstream releases with decodable releases.

`decodable-main` contains all the latest patches we are maintaining on top of Flink.
When forking off a new Flink minor version (e.g. `decodable-flink-1.16`), we cherry-pick
all patches on to of upstream `flink/release-1.16`.

All branches deploy `*-SNAPSHOT` versions to `decodable-mvn-snapshots-local` and regular releases release to `decodable-mvn-releases-local`.

The `decodable-preview-main` is not in use anymore.

For the future, we plan to periodically back up this git repository (as a repository with all branches) to S3, in case of accidential force-pushes.


## Approving changes

Every change in this repository has to go through an approval process. For compliance reasons (GDPR & SOC2), we need to follow this rule:

> System changes are approved by at least 1 independent person prior to deployment into production.

Any change that requires pushing a new branch to this repo has do be done through a GitHub issues ticket and a personal fork of this repo.
So in order to propose a new branch to be pushed to this repo, do the following
1. Push the branch to your personal fork of this repo
2. Open an issue with a link to the branch, ask for somebody to approve it
3. Push the branch to the repo.


## Using a local development version of Flink in a local decodable build

1. Make sure the Flink version is set to a SNAPSHOT version, such as `1.15.2-SNAPSHOT`.
2. Build Flink locally using Maven, with a command such as: 
  ```
  mvn325 clean install -DskipTests -Dcheckstyle.skip -Dspotless.skip
  ```
  (you can try to use any `mvn` version for building Flink, but officially, for building Flink, you have to use Maven 3.2.5. `mvn325` is a symlink on my local machine to a Maven 3.2.5 directory)

3. Build a Flink tgz archive:
  ```
  $ cd decodable-flink/flink-dist/target/flink-1.15.2-SNAPSHOT-bin
  $ ls 
  flink-1.15.2-SNAPSHOT
  $ tar czf flink-1.15.2-SN.tgz flink-1.15.2-SNAPSHOT
  ```
4. Once the build has succeeded, refer to the Flink build in the decodable gradle files:
 - set the Flink version in `dependencies.gradle` to the version you've defined in 1., such as `1.15.2-SNAPSHOT`.
 - make sure Gradle can access the local builds by putting `mavenLocal()` into `subprojects { repositories { ...` in `build.gradle`. (Put the `mavenLocal()` definition last to avoid resolution errors.)
 - reference to the Flink zip in `decodable-flink-app/build.gradle`, in the `flinkDist` task. Therefore, replace the S3Download task with a task like this
```
task flinkDist(type: Copy) {
  from '/Users/rmetzger/Projects/decodable-flink/flink-dist/target/flink-1.15.2-SNAPSHOT-bin/flink-1.15.2-SN.tgz'
  rename 'flink-1.15.2-SN.tgz', 'flink.tgz'
  into "${project.buildDir}/flink-dist/"
}
 ```
5. Build decodable.

Note: As an alternative to doing steps 1. - 3. locally, we could also build an archive & deploy a snapshot version with GitHub Actions, and download those files from AWS for the local build. 

