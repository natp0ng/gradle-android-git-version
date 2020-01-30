# gradle-android-git-version
[![CircleCI](https://circleci.com/gh/gladed/gradle-android-git-version/tree/master.svg?style=svg)](https://circleci.com/gh/gladed/gradle-android-git-version/tree/master)

A gradle plugin to calculate Android-friendly version names and codes from git tags.

If you are tired of manually updating your Android build files for each release, or generating builds that you can't trace back to code, then this plugin is for you!

## Installation and Use

### 1. Add the plugin to your project

Add `plugins` block to the top of your `app/build.gradle` (or equivalent):

```groovy
plugins {
    id 'com.gladed.androidgitversion' version '0.4.12'
}
```

### 2. Set `versionName` and `versionCode` from plugin results
```groovy
android {
    // ...
    defaultConfig {
        // ...
        versionName androidGitVersion.name()
        versionCode androidGitVersion.code()
```

### 3. Use a git tag to specify your version number (see [Semantic Versioning](http://semver.org))
```bash
$ git tag 1.2.3
$ gradle --quiet androidGitVersion
androidGitVersion.name	1.2.3
androidGitVersion.code	1002003
```

For more advanced usage, read on...

## Tag format

Tags should look like `1` or `3.14` or `1.12.5`. There must be at least one tag of this format reachable from the current commit, or the generated version name will be `"unknown"`.

Any suffix after the version number in the tag (such as `-release4` in `1.2.3-release4`) is included in the version name, but is ignored when generating the version code.

In some cases, you'll have more than one version being generated in a single repo. In this case you can supply a prefix like "lib-" to separate sets of tags. See `prefix` below for more.

## Intermediate Versions

Builds from non-tagged commits will generate a `name()` something like this:

`1.2.3-2-93411ff-fix_issue5-dirty`

In the example above, the components are:

| 1.2.3 | -2 | -93411ff | -fix_issue5 | -dirty |
| --- | --- | --- | --- | --- |
| Most recent tag | Number of commits since tag | Commit SHA prefix | Branch name | Dirty indicator |

Branches listed in `hideBranches` won't appear.

"-dirty" will only appear if the current branch has uncommitted changes.

You can customize this layout with `format` (see below).

## Version Codes

Version codes are calculated relative to the most recent tag. For example, version 1.2.3 will have
a version code of `1002003`.

You can customize the scheme used to generate version codes with `codeFormat` (see below).

## Methods

`name()` returns the current version name.

`code()` returns the current version code.

`flush()` flushes the internal cache of information about the git repo, in the event you have a gradle task that makes changes to the repo.

## Tasks

`androidGitVersion` prints the name and code, as shown above.

`androidGitVersionName` prints only the name.

`androidGitVersionCode` prints only the code.

## Usage Notes

* To save on bandwidth, CI servers may default to a "shallow" clone of your repo having no revision history, or may omit the pulling of tags. This hides the commit history and will prevent this plugin from accurately identifying the name or version code. This will result in an error not seen on your local build machine:

    ```
    Building app with versionName [untagged_814fca9-812f...9b8], versionCode [0]
    ```

    To address this, configure your CI project to pull enough history (e.g. `git fetch --depth=100 --tags`) to reach recent commits and tags of interest.

* `git worktree` is not supported.

## Configuration Properties

You can configure how names and codes are generated by adding an `androidGitVersion` block to your project's `build.gradle` file, before the `android` block. For example:

```groovy
androidGitVersion {
    abis = ["armeabi":1, "armeabi-v7a":2 ]
    baseCode 200000
    codeFormat 'MNNPPP'
    commitHashLength = 8
    format '%tag%%.count%%<commit>%%-branch%%...dirty%'
    hideBranches = [ 'develop' ]
    onlyIn 'my-library'
    prefix 'lib-'
    tagPattern(/^R[0-9]+.*/)
    untrackedIsDirty = false
}

android {
    // ...
}
```

### abis (map of String:int)
`abis` indicate how [ABI platforms](https://developer.android.com/ndk/guides/abis.html)
are mapped to integer codes. These integer codes are inserted into the `A` place in `codeFormat`.

The default `abis` are:

    ['armeabi':1, 'armeabi-v7a':2, 'arm64-v8a':3, 'mips':5, 'mips64':6, 'x86':8, 'x86_64':9 ]

### baseCode (int)
`baseCode` sets a floor for all generated version codes (that is, it is added to all generated version codes). Use this when you have already released a version with a code, and don't want to go backwards.

The default is `baseCode 0`, enforcing no minimum version code value.

### codeFormat (string)
`codeFormat` defines a scheme for building the version code. Each character corresponds to a reserved decimal place in the resulting code:

- `M` for the Major version number (1.x.x)
- `N` for the Minor version number (x.1.x)
- `P` for the Patch version number (x.x.1)
- `R` for the Revision number (e.g. x.x.x-rc1)
- `B` for the build number (revisions since last tag)
- `A` for the ABI platform code [1]
- `X` for a blank place filled with 0

[1] if you use `A` you must call `variants` to tell the plugin about your project's build variants. For example:

    android {
        ...
        androidGitVersion {
            codeFormat = 'AMNNPPP'
            variants applicationVariants
        }
    }

Note that changing the version code scheme for a released Android project can cause problems if your new version code does not
[increase monotonically](http://developer.android.com/tools/publishing/versioning.html). Consider `baseCode` if you are changing code formats from a prior release.

Android version codes are limited to a maximum version code of 2100000000. As a result, codeFormat only allows you to specify 9 digits.

The default is `codeFormat 'MMMNNNPPP'`, leaving 3 digits for each portion of the semantic version. A shorter code format such as `MNNPPP` is **highly recommended**.

### commitHashLength (int)
`commitHashLength` specifies how many characters to use from the beginning of commit SHA hash (e.g. `commit` in `format`). Values from 4 to 40 are valid.

The default length is 7 (similar to `git describe --tags`).

### hideBranches (list of strings/regexes)
`hideBranches` identifies branch names to be hidden from the version name for intermediate (untagged) commits.

For example, if 'master' is your development branch, `hideBranches ['master']` results in build names like `1.2-38-9effe2a` instead of `1.2-38-master-9effe2a`.

Note that each element of hideBranches is interpreted as a regex pattern, for example, `[ 'master', 'feature/.*' ]`.

The default is `hideBranches [ 'master', 'release' ]`, meaning that intermediate builds will not show these branch names.

### format (string)
`format` defines the form of the version name.

Parts may include:
 - `tag` (the last tag)
 - `count` (number of commits, if any, since last tag)
 - `commit` (most recent commit prefix, if any, since the last tag)
 - `branch` (branch name, if current branch is not in `hideBranches`)
 - `dirty` (inserting the word "dirty" if the build was made with uncommitted changes).
 - `describe` (the string generated by [git.describe](https://download.eclipse.org/jgit/site/5.6.0.201912101111-r/apidocs/org/eclipse/jgit/api/DescribeCommand.html), similar to the output from `git describe --tags`. **Warning**: This command works differently from the rest of this plugin and result in different `tag`, `count`, or `commit` parts than those used to generate the version code.)

Parts are delimited as `%<PARTNAME>%`. Any other characters appearing between % marks are preserved.

Parts are sometimes omitted (such as a branch name listed by `hideBranches`). In this case the entire part will not appear.

The default is `format "%tag%%-count%%-commit%%-branch%%-dirty%"`

### onlyIn (string)
`onlyIn` sets a required path for relevant file changes. Commits that change files in this path will count, while other commits will be ignored for versioning purposes.

For example, consider this directory tree:
```
+-- my-app/
    +-- .git/
    +-- build.gradle
    +-- app/
    |   +-- build.gradle
    |   +-- src/
    +-- lib/
        +-- build.gradle
        +-- src/
```
If `my-app/lib/build.gradle` is configured with `onlyIn 'lib'`, then changes to files in other paths (like `my-app/build.gradle` or `my-app/app/src`) will not affect the version name.

The default is `onlyIn ''`, including all paths.

### prefix (string)
`prefix` sets the required prefix for any relevant version tag. For example, with `prefix 'lib-'`, the tag `lib-1.5` is used to determine the version, while tags like `1.0` and `app-2.4.2` are ignored. When found, the prefix is removed from the front of the final version name.

The default is `prefix ''`, matching all numeric version tags.

### tagPattern (string/regex)
`tagPattern` limits the search for the most recent version tag to those that match the pattern. For example, `tagPattern(/^v[0-9]+.*)` limits matches to tags like `v1.6`.

If both `prefix` and `tagPattern` are used, the `prefix` strings should be included in the `tagPattern`.
 
The default is `tagPattern(/^$prefix[0-9]+.*/)`, finding all tags beginning with the prefix (if specified) and a digit.

### untrackedIsDirty (boolean)
When `untrackedIsDirty` is true, a version is considered dirty when any untracked files are detected in the repo's directory.

The default is `untrackedIsDirty false`; only tracked files are considered when deciding on "dirty".

## Deprecated Configuration Properties

See [DEPRECATED.md](DEPRECATED.md).

## License

All code here is Copyright 2015-2019 by Glade Diviney, and licensed under the [Apache 2.0 License](http://www.apache.org/licenses/LICENSE-2.0).
