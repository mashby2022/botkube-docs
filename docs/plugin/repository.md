---
id: repo
title: "Repository"
sidebar_position: 5
---

A plugin repository is a place where you store your plugin binaries. This repository must be publicly available and supports downloading assets via HTTP(s). This document explains how to create Botkube plugin repositories by providing examples based on GitHub functionality. However, any static file server can be used, for instance: `s3`, `gcs`, etc.

# Index file

Your plugin repository must contain at least one index file and one plugin binary. Depending on your needs and preferences, you can create one or more index files to categorize your plugins. You can host both the executor and source plugins in a single repository. You can also include them in the same index file.

In the index file, provide an entry for every plugin from your plugin repository. The index file must have the following syntax:

```yaml
entries:
  - name: { plugin_name }
    type: { plugin_type } # executor or source
    description: { plugin_description }
    version: { plugin_version }
    urls:
      - url: { url_to_plugin_binary }
        platform:
          os: { plugin_operating_system } # darwin or linux
          architecture: { plugin_architecture } # amd64 or arm64
```

It is not required to host a plugin binary on the same server as the index file.

## Generate index file

You can create the index file by yourself our use our tool to generate it automatically based on the directory with plugins binaries. The binaries must be named according to the following pattern:

- For executors, `executor_{plugin_name}_{os}_{arch}`; for example, `executor_helm_darwin_amd64`.
- For sources, `source_{plugin_name}_{os}_{arch}`; for example, `source_prometheus_darwin_amd64`.

**Steps**

1. In your plugin repository, add `tools.go`:

   ```go
   cat << EOF > tools.go
   //go:build tools

   package tools

   import (
   	 _ "github.com/kubeshop/botkube/hack"
   )
   EOF
   ```

   :::note
   We use the `tools.go` file, which is the [recommended way of tracking tool dependencies](https://github.com/golang/go/wiki/Modules#how-can-i-track-tool-dependencies-for-a-module).
   :::

2. Refresh dependencies:

   ```bash
   go mod tidy
   ```

3. Build all your plugins. See [**Build plugin binaries**](custom-executor.md).

4. Generate an index file:

   ```bash
   go run github.com/kubeshop/botkube/hack -binaries-path "./dist" -url-base-path "https://example.com"
   ```

   :::info
   Replace the `-url-base-path` flag with the base path of your HTTP server. See [**Hosting plugins**](#host-plugins) for some examples.
   :::

# Host plugins

This section describes example ways for serving Botkube plugins.

## GitHub releases

A GitHub release allows you to upload additional assets that are later accessible with a predictable URL. When you generate the index file, specify the `-url-base-path` flag as `https://github.com/{owner}/{repo}/releases/download/{release_tag}`, for example, `https://github.com/kubeshop/botkube/releases/download/v1.0.0`.

Once the plugin binaries are built and the index file is generated, you can create a GitHub release using [GitHub CLI](https://cli.github.com/). For example:

```bash
gh release create v1.0.0 \
 ./dist/source_* \
 ./dist/executor_* \
 ./plugins-index.yaml
```

### Automation

You can use [GitHub Actions](https://docs.github.com/en/actions) to publish Botkube plugins automatically each time a new tag is pushed. To do that, add the following workflow under `.github/workflows/release.yml` in your repository:

```yaml
name: Official Release

on:
  push:
    tags:
      - "*"

jobs:
  release:
    name: Release
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Set up Go
        uses: actions/setup-go@v3
        with:
          go-version-file: "go.mod"
          cache: true

      - name: Build plugins binaries
        uses: goreleaser/goreleaser-action@v3
        with:
          version: latest
          args: build --rm-dist

      - name: Generate plugins index.yaml
        env:
          PLUGIN_DOWNLOAD_URL_BASE_PATH: "https://github.com/${{ github.repository }}/releases/download/${{ github.ref_name }}"
        run: |
          go run github.com/kubeshop/botkube/hack -binaries-path "./dist"
      - name: Publish GitHub release
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          gh release create "${GITHUB_REF#refs/tags/}" \
          --generate-notes \
          ./dist/source_* \
          ./dist/executor_* \
          ./plugins-index.yaml
```

This workflow uses GoReleaser that was described in the [**Build plugin binaries**](custom-executor.md#build-plugin-binaries) section.

## GitHub pages

GitHub allows you to serve static pages via GitHub Pages. When you generate the index file, specify the `-url-base-path` flag as `https://{user}.github.io/{repository}`, for example, `https://kubeshop.github.io/botkube-plugins`.

**Initial setup**

1. Navigate to the Git repository with your plugins.

2. Create the `gh-pages` branch:

   ```bash
   git switch --orphan gh-pages
   git commit --allow-empty -m "Initialization commit"
   git push -u origin gh-pages
   ```

3. Follow [this](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-from-a-branch) guide to make sure your `gh-pages` branch is set as the source for GitHub Pages.

**Publishing steps**

1. Clone `gh-pages` into `/tmp/botkube-plugins`:

   ```bash
   git clone -b gh-pages "https://github.com/{owner}/{repo}.git" /tmp/botkube-plugins
   ```

2. Move built binaries and generated index file:

   ```bash
   mv dist/executor_* /tmp/botkube-plugins/
   mv dist/source_* /tmp/botkube-plugins/
   mv plugins-index.yaml /tmp/botkube-plugins
   ```

3. Commit and push copied files:

   ```bash
   cd /tmp/botkube-plugins
   git add -A
   git commit -m "Release Botkube plugins"
   git push
   ```

4. Remove cloned `gh-pages`:

   ```bash
   cd -
   rm -rf /tmp/botkube-charts
   ```

In such setup, you can use your default branch to store your plugins code, and the `gh-pages` branch as a plugin repository.

### Automation

You can use [GitHub Actions](https://docs.github.com/en/actions) to publish Botkube plugins automatically each time a new tag is pushed. To do that, add the following workflow under `.github/workflows/release.yml` in your repository:

```yaml
name: Deploy Botkube plugins GitHub Pages

on:
  push:
    tags:
      - "*"

# Sets permissions of the GITHUB_TOKEN to allow deployment to GitHub Pages
permissions:
  contents: read
  pages: write
  id-token: write

# Allow one concurrent deployment
concurrency:
  group: "pages"
  cancel-in-progress: true

jobs:
  # Single deploy job since we're just deploying
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Setup Pages
        uses: actions/configure-pages@v2

      - name: Set up Go
        uses: actions/setup-go@v3
        with:
          go-version-file: "go.mod"
          cache: true

      - name: Build plugins binaries
        uses: goreleaser/goreleaser-action@v3
        with:
          version: latest
          args: build --rm-dist

      - name: Generate plugins index.yaml
        env:
          PLUGIN_DOWNLOAD_URL_BASE_PATH: "https://${{github.repository_owner}}.github.io/${{ github.event.repository.name }}"
        run: |
          go run github.com/kubeshop/botkube/hack -binaries-path "./dist"
      - name: Publish GitHub release
        run: |
          mkdir public
          mv dist/executor_* public/
          mv dist/source_* public/
          mv plugins-index.yaml public/

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v1
        with:
          path: public
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v1
```

## Use hosted plugins

To use the plugins that you published, add your repository under `plugins` in the [values.yaml](https://github.com/kubeshop/botkube/blob/main/helm/botkube/values.yaml) file for a given Botkube deployment. For example:

```yaml
plugins:
  repositories:
    repo-name:
      url: https://example.com/plugins-index.yaml
```

Once the plugin repository is added, you can refer to it in the `executor` or `sources` section.

```yaml
executors:
  "plugin-based":
    repo-name/executor-name@v1.0.0: # Plugin name syntax: {repo}/{plugin}[@{version}]. If version is not provided, the latest version from repository is used.
      enabled: true
      config: {} # Plugin's specific configuration.
sources:
  "plugin-based":
    repo-name/source-name@v1.0.0: # Plugin name syntax: {repo}/{plugin}[@{version}]. If version is not provided, the latest version from repository is used.
      enabled: true
      config: {} # Plugin's specific configuration.
```