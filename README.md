# gradle-android-git-version
A gradle plugin to calculate Android-friendly version names and codes from git tags.

## Usage

Add the plugin the top of your `app/build.gradle` (or equivalent):
```groovy
plugins {
    id "com.gladed.androidgitversion" version "0.2.2"
}
```

Set `versionName` and `versionCode` from plugin results:
```groovy
android {
    // ...
    defaultConfig {
        // ...
        versionName androidGitVersion.name()
        versionCode androidGitVersion.code()
```

Use git tag to specify a base version number (see [Semantic Versioning](http://semver.org))
```bash
$ git tag 1.2.3
$ git androidGitVersion
androidGitVersion.name	1.2.3
androidGitVersion.code	1002003
```

## Non-tagged versioning

For builds from commits that are not tagged, `name()` will return a build of this form:

`1.2.3-2-master-93411ff-dirty`

The components in the example above are as follows:
`1.2.3` | `-2` | `-master` | `-93411ff` | `-dirty`
- | - | - | - | -
From tag | Number of commits since tag | Branch name | Commit prefix | Dirty flag, present only if files have changes on-disk

## Version Codes

Version codes are calculated relative to the most recent tag.

The code is generated by combining each version part with a multiplier. Unspecified version parts are assumed to be zero even if not supplied. So version 1.22 becomes 1022000, while version 2.4.11 becomes 2004011. This allows successive versions to have incrementing version codes.

Intermediate, non-tagged commits do *not* affect the value of the code. The code only advances when a new version tag is specified.

You can configure this behavior with `multipler` and `parts` properties, but be warned that changing the version code scheme for a released Android project can cause problems if your new version code does not [increase monotonically](http://developer.android.com/tools/publishing/versioning.html).

## Configuration Properties

An `androidGitVersion` block can be supplied to configure behavior, e.g.

```groovy
android {
    androidGitVersion {
        prefix 'lib-'
        onlyIn 'my-library'
        multiplier 10000
        parts 2
    }
```

### `prefix` string (default `''`)
Set a tag prefix to indicate that relevant version tags will start with the specified string. For example, with `prefix 'lib'`, a tags like `lib-1.5` will be found while a tag like `1.0` or `app-2.4.2` will be ignored.


### `onlyIn` string (default `''`)
Set the `onlyIn` path to indicate a path within your project. Commits that change files in this path will count, while other commits will not. This is useful when building a versioned library from a git project containing other projects (apps and other libraries).

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
If a commit is tagged with `1.0.1`, and `my-app/lib/build.gradle` is configured with `onlyIn 'lib'`, then commits that change `my-app/build.gradle` or `my-app/app/src` will not affect the version name or code generated from `my-app/lib/build.gradle`.

### `multiplier` int (default 1000)
Changes the multipler used to combine each part of the version number.

For example if you want version 1.2.3 to have a version code of 100020003 (allowing for 9999 patch increments), use `multiplier 10000`.

### `parts` int (default 3)
Changes the assumed number of parts. If you know your product will only ever use two version number parts (1.2) then use `parts 2`.

### `baseCode` int (default 0)
A base version code added to all generated version codes. Use this when you have already released a version with a code, and don't want to go backwards.