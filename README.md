# Generate Pre-Packaged Binaries for PIE Extensions

> **Fork of [`php/pie-ext-binary-builder`](https://github.com/php/pie-ext-binary-builder)** by James Titcumb.
>
> This fork adds:
> - **`build-path` input** — build from a subdirectory (useful for mirror repos where extension source lives in a subfolder)
> - **Asset replacement** — re-uploads the release asset if one with the same name already exists
> - **CI-built dist** — `dist/` is built on release via GitHub Actions, not committed to the repo

This is a GitHub action designed to help generate pre-packaged binaries for
[PIE (PHP Installer for Extensions)](https://github.com/php/pie) extensions and attach
them to your GitHub releases. It will:

 - Build a `.so` from your extension
 - Package the `.so` in an archive, and name it according to PIE's expectations
 - Upload the archive to your GitHub release

> [!TIP]
> Looking for Windows support? You probably want [`php/php-windows-builder`](https://github.com/php/php-windows-builder?tab=readme-ov-file#examples)

## The Action

You must specify a `release-tag`. This is a tag, for which you have created
a release ready for the action to upload the builds to.

> [!IMPORTANT]
> If you have enabled immutable releases on your repository, the release must be
> a draft release, otherwise uploading the assets will fail.

```yaml
uses: pie-extensions/pie-ext-binary-builder@v1
with:
  configure-flags: '--enable-something --enable-other-things'
  build-path: 'ext'  # optional, defaults to repo root
  release-tag: ${{ github.ref_name }}
  github-token: ${{ secrets.GITHUB_TOKEN }}
```

### Inputs

| Name              | Description                                                                                     | Required | Default |
|-------------------|-------------------------------------------------------------------------------------------------|----------|---------|
| `release-tag`     | The tag to use when building the extension; there must be an existing draft release for the tag | `true`   | -       |
| `github-token`    | The GitHub token to use. Usually `${{ secrets.GITHUB_TOKEN }}` would be fine for most cases.    | `true`   | -       |
| `configure-flags` | If you need to pass additional flags to the `./configure` command, specify them here            | `false`  | `''`    |
| `build-path`      | Path to the extension source directory containing `config.m4`, relative to repo root            | `false`  | `'.'`   |

### Outputs

| Name           | Description                       |
|----------------|-----------------------------------|
| `package-path` | Path to the generated `.zip` file |

## Complete example

This use case is for a scenario where:

 - The action triggers when you push any tag
 - It will create a draft release
 - It will then check out your extension, set up the required PHP version, build it, and upload to the draft release

You would then have to navigate to the draft release and publish it.

```yaml
name: Build and release binaries for PIE

on:
  push:
    tags:
      - '*'

permissions:
  contents: read

jobs:
  # This first step will create a *draft* release based on the tag name. In the
  # case where immutable releases are enabled, the release MUST be in draft
  # mode, otherwise we would not be able to attach the release assets (since
  # the release is immutable once published.
  create-draft-release:
    runs-on: ubuntu-latest
    permissions:
      # contents:write is required to create the draft release
      contents: write
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-tags: 'true'
          ref: ${{ github.ref }}
      - name: Create draft release from tag
        env:
          GH_TOKEN: ${{ github.token }}
        run: gh release create "${{ github.ref_name }}" --title "${{ github.ref_name }}" --draft --notes-from-tag

  add-pie-binaries:
    needs: [ create-draft-release ]
    runs-on: ${{ matrix.operating-system }}
    # The matrix defines which combination of binaries you want to build
    strategy:
      matrix:
        operating-system:
          - ubuntu-latest
          - macos-latest
        php-versions:
          - 8.2
          - 8.3
          - 8.4
        zts-mode:
          - ts
          - nts
    permissions:
      # contents:write is required to upload to the release assets
      contents: write
    steps:
      - name: Checkout
        uses: actions/checkout@v6

      # Install the desired version of PHP in order to build the extension for it
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ matrix.php-versions }}
        env:
          phpts: ${{ matrix.zts-mode }}

      # Finally, this invokes the action, which builds the extension, creates
      # the archive with the correct naming, and uploads it to the release for
      # the given tag name
      - name: Build and release
        id: php-ext-binary-builder
        uses: pie-extensions/pie-ext-binary-builder@v1
        with:
          release-tag: ${{ github.ref_name }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
```
