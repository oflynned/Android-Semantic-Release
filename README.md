<h1 align="center">
Android Version Bump
</h1>

<p align="center">
  <a href="https://github.com/oflynned/Android-Semantic-Release/actions/workflows/master.yml">
    <img alt="GitHub Actions status" src="https://github.com/oflynned/Android-Semantic-Release/actions/workflows/master.yml/badge.svg">
  </a>
  <a href="https://codecov.io/gh/oflynned/Android-Semantic-Release">
    <img src="https://codecov.io/gh/oflynned/Android-Semantic-Release/branch/master/graph/badge.svg?token=VTW7E1X43G" alt="Codecov">
  </a>
</p>

Sick of depending on fastlane or bumping versions manually?

This action uses semantic commits to automate version bumping in native Android repos and creates a release on every successful merge to master.
By using this action, your CI runner can examine the commits in the event and bump the semantic version accordingly as long as you are using conventional commits.
If you are not using conventional commits, it will just bump the patch version consistently.

Keep in mind that the max patch version is 99 on Android if you use the default version code generator. Inspired by [npm version bump](https://github.com/phips28/gh-action-bump-version). :tada:

## Installation

### GitHub Actions

#### Dependencies

It's recommended to use `actions/checkout@v2` or greater when checking out the repo.

#### .github/workflows/master.yml

Add the following to your yaml workflow declaration. 
Make sure to bump **before** building any artifacts so that the correct versions are applied.

```yaml
- name: Bump version
  id: bump_version
  uses: oflynned/android-version-bump@master
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

#### Private repos

To use this action with `${{ secrets.GITHUB_TOKEN }}` in a private repo, you must set the contents: write permission for the token to write to the package.json file specified in the workflow.

```
jobs:
  publish:
    ...
    permissions:
      contents: write
```

### Android Studio

#### version.properties
The CI will create a `version.properties` by default, but you can also create this yourself to set the first version **to bump from**.
Remember, creating a file with 0.0.1 will bump it to 0.0.2 on first run.
Not creating any `version.properties` will make the CI action handle this edge case itself and create the first version 0.0.1 without bumping.

```properties
majorVersion=1
minorVersion=0
patchVersion=0
buildNumber=
```  

The build number can remain unset if you are using the default version name generator below.

#### build.gradle

`build.gradle` will not use these values unless you add some logic to it.

The version properties file must first be loaded into the Gradle context. 
Add this to the top of `build.gradle`
```groovy
Properties props = new Properties()
props.load(new FileInputStream("$project.rootDir/version.properties"))
props.each { prop ->
    project.ext.set(prop.key, prop.value)
}
```

You can define how version name & codes are generated in that file:

```groovy
private Integer getVersionCode() {
    int major = ext.majorVersion as Integer
    int minor = ext.minorVersion as Integer
    int patch = ext.patchVersion as Integer

    return major * 10000 + minor * 100 + patch
}

private String getVersionName() {
    if (ext.buildNumber) {
        return "${ext.majorVersion}.${ext.minorVersion}.${ext.patchVersion}.${ext.buildNumber}"
    }

    return "${ext.majorVersion}.${ext.minorVersion}.${ext.patchVersion}"
}
```

You then use those two functions in your config in the same file:

```groovy
android {
    defaultConfig {
        versionCode getVersionCode()
        versionName getVersionName()
    }
}
```

## Workflow

### Major

Bumps on the following intents:
```text
major: drop support for api v21
```

Also triggered if the commit body contains `BREAKING CHANGE` or if the intent contains a `!`.
```text
refactor!: drop support for api v21
refactor: BREAKING CHANGE drop support for api v21  
```

If any commit like this is in the list of commits within the event, then the **major** version will get bumped (`1.0.0 -> 2.0.0`)

### Minor

Bumps on the following intents:
```text
feat: add oauth login with google
minor: allow user to delete account
```

If any commit like this is in the list of commits within the event, then the **minor** version will get bumped (`1.0.0 -> 1.1.0`)

### Patch

Any other changes, even if not following conventional commits will bump the patch version (`1.0.0 -> 1.0.1`)

### Build number

This field is optional.

Providing the build number will automatically affix this to the version name

Enable this field by passing a build number/string/SHA as an env var to the action:

```yaml
- name: Bump version
  id: bump_version
  uses: oflynned/android-version-bump@master
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  with:
    build_number: ${{ github.run_number }}
```

## Optional arguments

Pass these in the `with:` block

| Tag            | Effect                                                                                                                                                                         | Example                                                                     | Default value            |
|----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------|--------------------------|
| tag_prefix     | Allows you to set a prefix for a tag                                                                                                                                           | `tag_prefix: 'v'` sets the tag to `v1.0.0`                                  | ''                       |
| skip_ci        | Affixes `[skip-ci]` to the end of the commit message, even if you provide a custom message                                                                                     | `skip_ci: true`                                                             | false                    |
| build_number   | Sets the build run number in the version                                                                                                                                       | `build_number: ${{ github.run_number }}` generates `1.0.0.5`                | ''                       |
| commit_message | Sets the commit message when a release bump is performed. Can optionally use `{{ version }}` to insert the generated version bump with the tag prefix into the commit message. | `ci: {{ version }} was just released into the wild! :tada: :partying_face:` | `release: {{ version }}` |
    
## Q&A

### I need to also create a release, not just a tag 

The action also outputs a tag that you can use in later stages of the workflow like so. 

```yaml
- name: Create release
  id: create_release
  uses: actions/create-release@v1
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  with:
    tag_name: ${{ steps.bump_version.outputs.new_tag }}
    release_name: ${{ steps.bump_version.outputs.new_tag }}
    draft: false
    prerelease: false
```

Make sure you also assign the bump version step its own id (in this case it was already set to `id: bump_version`)
