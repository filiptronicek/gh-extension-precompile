# Action for publishing GitHub CLI extensions

A GitHub CLI extension is any GitHub repository named `gh-*` that publishes a Release with precompiled binaries. This GitHub Action can be used in your extension repository to automate the creation and publishing of those binaries.

## Go extensions

> [!Note]
> With the use of `actions/setup-go@v5` for Go extensions, **cache is enabled by default** as part of the [action's `v4` release](https://github.com/actions/setup-go/releases/tag/v4.0.0). The action won’t throw an error if the cache can’t be restored or saved. The action will throw a warning message but it won’t stop a build process. 

Create a workflow file at `.github/workflows/release.yml`:

```yaml
name: release

on:
  push:
    tags:
      - "v*"

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: cli/gh-extension-precompile@v2
        with:
          go_version_file: go.mod
```

Then, either push a new git tag like `v1.0.0` to your repository, or create a new Release and have it initialize the associated git tag.

When the `release` workflow finishes running, compiled binaries will be uploaded as assets to the `v1.0.0` Release and your extension will be installable by users of `gh extension install` on supported platforms.

You can safely test out release automation by creating tags that have a `-` in them; for example: `v2.0.0-rc.1`. Such Releases will be published as _prereleases_ and will not count as a stable release of your extension.

To maximize portability of built products, this action builds Go binaries with [cgo](https://pkg.go.dev/cmd/cgo) disabled unless you [enable building for Android targets](#building-for-android). To override cgo for all build targets, set the `CGO_ENABLED` environment variable:

```yaml
- uses: cli/gh-extension-precompile@v2
  env:
    CGO_ENABLED: 1
```

### Building for Android

As of `gh-extension-precompile@v2`, building for Android targets like `android-arm64` and `android-amd64` is disabled by default. To enable building for Android targets, set at least the `release_android` and `android_sdk_version` action inputs:

```yaml
- uses: cli/gh-extension-precompile@v2
  with:
    go_version_file: go.mod
    release_android: true
    android_sdk_version: 34
```

If you are running the workflow on a GitHub hosted runner, you do not need to set the `android_ndk_home` input. The Android SDK Build-tools and environment variables required to build for Android targets are [pre-installed and configured on GitHub hosted runners](https://github.com/actions/runner-images/blob/8cdc506384655ceaaa62d3f800e15b844e06bea4/images/ubuntu/Ubuntu2404-Readme.md?plain=1#L214-L233). 

However, if you are running the workflow on a self-hosted runner, you need to also configure the `android_ndk_home` action input to the installation path of the Android NDK on the runner:

```yaml
- uses: cli/gh-extension-precompile@v2
  with:
    go_version_file: go.mod
    release_android: true
    android_sdk_version: 34
    android_ndk_home: /path/to/android-ndk
```

### Customizing the build process for Go extensions

If you need to customize the build process for your Go extension, you can provide a custom build script. See [Extensions written in other compiled languages](#extensions-written-in-other-compiled-languages) below for instructions.

## Extensions written in other compiled languages

If you aren't using Go for your compiled extension, or your Go extension requires customizations to the build script, you'll need to provide your own script for compiling your extension:

```yaml
- uses: cli/gh-extension-precompile@v2
  with:
    build_script_override: "script/build.sh"
```

The build script will receive the release tag name as the first argument.

This script **must** produce executables in a `dist` directory with file names ending with: `{os}-{arch}{ext}`, where the extension is `.exe` on Windows and blank on other platforms. For example:
- `dist/gh-my-ext_v1.0.0_darwin-amd64`
- `dist/gh-my-ext_v1.0.0_windows-386.exe`

For valid `{os}-{arch}` combinations, see the output of `go tool dist list` with the Go version you intend to use for compiling.

Potentially useful environment variables available in your build script:

- `GITHUB_REPOSITORY`: name of your extension repository in `owner/repo` format
- `GITHUB_TOKEN`: auth token with access to GITHUB_REPOSITORY

## Checksum file and signing

This action can optionally produce a checksum file for all published executables and sign it with GPG.

To enable this, make sure your repository has the secrets `GPG_SECRET_KEY` and `GPG_PASSPHRASE` set. (Tip: you can use `gh secret set` for this; follow the instructions [here](https://github.com/crazy-max/ghaction-import-gpg) to obtain the correct secret values.) Then, configure this action like so:

```yaml
name: release

on:
  push:
    tags:
      - "v*"

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - id: import_gpg
        uses: crazy-max/ghaction-import-gpg@v5
        with:
          gpg_private_key: ${{ secrets.GPG_PRIVATE_KEY }}
          passphrase: ${{ secrets.GPG_PASSPHRASE }}
      - uses: cli/gh-extension-precompile@v2
        with:
          gpg_fingerprint: ${{ steps.import_gpg.outputs.fingerprint }}
```


## Support for Artifact Attestations

This action can optionally generate signed build provenance attestations for all published executables within `${{ github.workspace }}/dist/*`.

For more information, see ["Using artifact attestations to establish provenance for builds"](https://docs.github.com/en/actions/security-guides/using-artifact-attestations-to-establish-provenance-for-builds).

```yaml
name: release

on:
  push:
    tags:
      - "v*"

permissions:
  contents: write
  id-token: write
  attestations: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: cli/gh-extension-precompile@v2
        with:
          generate_attestations: true
```


## Authors

- nate smith <https://github.com/vilmibm>
- the GitHub CLI team <https://github.com/cli>
