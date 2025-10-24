# BNET Addon Upload Github Action

![BNET Upload](https://img.shields.io/badge/BNET-Upload-blue?logo=github-actions&style=flat-square)
[![Used by](https://img.shields.io/badge/dynamic/json?color=success&label=Used%20by&query=repositoryCount&url=https://api.github.com/repos/m00nyONE/bnet-upload)](https://github.com/m00nyONE/bnet-upload/network/dependents)
[![GitHub release](https://img.shields.io/github/v/tag/m00nyONE/bnet-upload?label=release)](https://github.com/m00nyONE/bnet-upload/releases)


This Github Action Automates the upload of your ESO (Elder Scrolls Online) addon to [mods.bethesda.net](https://mods.bethesda.net/en/elderscrollsonline/) by using the ESOAddOnUploader-Cli for bethesda.net made by [@sirinsidiator](https://github.com/sirinsidiator).

This only works for Console Addons.

It wraps the upload process in a safe and reusable action, helping you keep your secrets secure and your workflows clean.

## Features

- ✅ Secure: credentials are masked from logs if they ever leak.
- ✅ Validation: all inputs are validated.
- ✅ Simple: Clean and easy integration in your workflows.
- ✅ Customizable: Reusable across multiple addons or repositories.

## Usage

### 0. Prerequisite

- The addon needs to be uploaded manually to BNET once to get it's ID. You can find the ID of your addon in the URL. Example [LibGroupCombatStats](https://mods.bethesda.net/en/elderscrollsonline/details/25cfa10b-66f5-4e8c-9d1a-1c452491665f/LibGroupCombatStats): the ID is `25cfa10b-66f5-4e8c-9d1a-1c452491665f`
- MFA on Bethesda.net must be turned OFF for the cli-uploader to work. You can turn it off here: https://accounts.bethesda.net/en/enhanced-security

### 1. Reference the Action in Your Workflow

Example workflow step:

```yaml
- name: Upload to BNET
  uses: m00nyONE/bnet-upload@v1
  with:
    BNET_USERNAME: ${{ secrets.BNET_USERNAME }}
    BNET_PASSWORD: ${{ secrets.BNET_PASSWORD }}
    addon_id: '25cfa10b-66f5-4e8c-9d1a-1c452491665f'
    version: ${{ env.ADDON_VERSION }}
    zip_file: ${{ env.ZIP_FULL_NAME }}
    release_notes_file: 'CHANGELOG-stripped.md'
    publish: false
```

### 2. Inputs
| Name                 | Required | Default | Description                                                                                                              |
|----------------------|----------|---------|--------------------------------------------------------------------------------------------------------------------------|
| BNET_USERNAME        | true     | -       | Your Bethesda.net username (stored as a GitHub secret)                                                                   |
| BNET_PASSWORD        | true     | -       | Your Bethesda.net password (stored as a GitHub secret)                                                                   |
| addon_id             | true     | -       | The BNET Addon ID                                                                                                        |
| version              | true     | -       | The version number of the addon                                                                                          |
| zip_file             | true     | -       | Path to your zipped addon file (relative to your repository root directory or absolute from the runners root directory)  |
| release_notes_file   | true     | -       | Path to your release notes file (relative to your repository root directory or absolute from the runners root directory) |
| publish              | false    | true    | Whether to publish the addon after upload                                                                                |
| concurrency          | false    | 1       | The number of concurrent uploads                                                                                         |
| cli_uploader_version | false    | 1.0.0   | The version of the cli-uploader to use (e.g. '1.0.0')                                                                    |

### 3. Secrets

Make sure to add your Bethesda.net username and password to your repository’s secrets:
- go into your repository settings -> Secrets and create two new secrets with the name `BNET_USERNAME` and `BNET_PASSWORD` and put in your credentials there as a value.

### 4. Full Workflow Example

This is an example based on LibGroupCombatStats. Adapt it to your needs.

```yaml
name: Create ESOUI Release

on:
  push:
    branches:
      - release

jobs:
  docs:
    if: github.repository_owner == 'm00nyONE'
    uses: ./.github/workflows/_generate-docs.yml
    permissions:
      contents: write
  release:
    if: github.repository_owner == 'm00nyONE'
    name: "release"
    runs-on: ubuntu-latest
    permissions: write-all
    steps:
      - name: Check out repository
        uses: actions/checkout@v4

      - name: get repo name
        run: echo "REPO_NAME=${{ github.event.repository.name }}" >> $GITHUB_ENV

      - name: Create env variables
        run: |
          addon_name="${{ env.REPO_NAME }}"
          version="${{ env.BUILD_DATE_WITH_HYPHEN }}"

          echo "ADDON_NAME=$addon_name" >> $GITHUB_ENV
          echo "ADDON_VERSION=$version" >> $GITHUB_ENV

          echo "ZIP_FULL_NAME=${addon_name}-${version}.zip" >> $GITHUB_ENV

      - name: Get current year, month and day
        run: |
          echo "BUILD_DATE_YEAR=$(date -u +'%Y')" >> $GITHUB_ENV
          echo "BUILD_DATE_MONTH=$(date -u +'%m')" >> $GITHUB_ENV
          echo "BUILD_DATE_DAY=$(date -u +'%d')" >> $GITHUB_ENV
          echo "BUILD_DATE_NUMBER=$(date +'%Y%m%d')" >> $GITHUB_ENV
          echo "BUILD_DATE_WITH_DOT=$(date +'%Y.%m.%d')" >> $GITHUB_ENV
          echo "BUILD_DATE_WITH_HYPHEN=$(date +'%Y-%m-%d')" >> $GITHUB_ENV

      - name: Replace placeholders with current date
        run: |
          sed -i "s/version = \"dev\"/version = \"${{ env.ADDON_VERSION }}\"/g" ${{ env.ADDON_NAME }}.lua
          sed -i "s/## Version: dev/## Version: ${{ env.ADDON_VERSION }}/g" ${{ env.ADDON_NAME }}.addon
          sed -i "s/## AddOnVersion: 99999999/## AddOnVersion: ${{ env.BUILD_DATE_NUMBER }}/g" ${{ env.ADDON_NAME }}.addon

      - name: Create ZIP archive
        run: |
          REPO_FOLDER=$(pwd)
          TMP_FOLDER="/tmp/${{ env.ADDON_NAME }}"

          # Define the path to the ignore pattern file
          ignore_file=".build-ignore"

          # Read and process ignore patterns into a single line
          exclude_patterns=$(cat "$ignore_file" | awk '{print "--exclude " $0}' | tr '\n' ' ')

          # Make folder and copy content
          mkdir -p $TMP_FOLDER
          rsync -a --quiet $exclude_patterns "$REPO_FOLDER/" "$TMP_FOLDER/"

          # create zip
          (cd /tmp && zip -r --quiet "$REPO_FOLDER/${{ env.ZIP_FULL_NAME }}" "${{ env.ADDON_NAME }}")

      - name: Extract latest changelog entry
        run: |
          awk '/^## / { if (!found) { found=1; print; next } else { exit } } found' CHANGELOG.md > latest_changes.md
          cat latest_changes.md

      - name: Create GitHub Release
        #id: create_release
        uses: ncipollo/release-action@v1
        with:
          name: "${{ env.ADDON_VERSION }}"
          commit: ${{ github.ref }}
          tag: "${{ env.ADDON_VERSION }}"
          artifacts: "${{ env.ZIP_FULL_NAME }}"
          artifactContentType: application/zip
          bodyFile: latest_changes.md
          allowUpdates: true
          makeLatest: true

      - name: Upload to ESOUI
        uses: m00nyONE/esoui-upload@v2
        with:
          api_key: ${{ secrets.ESOUI_API_KEY }}
          addon_id: '4024'
          version: ${{ env.ADDON_VERSION }}
          zip_file: "${{ env.ZIP_FULL_NAME }}"
          changelog_file: latest_changes.md

      - name: Upload to BNET
        uses: m00nyONE/bnet-upload@main
        with:
          BNET_USERNAME: ${{ secrets.BNET_USERNAME }}
          BNET_PASSWORD: ${{ secrets.BNET_PASSWORD }}
          addon_id: '25cfa10b-66f5-4e8c-9d1a-1c452491665f'
          version: ${{ env.ADDON_VERSION }}
          zip_file: "${{ env.ZIP_FULL_NAME }}"
          release_notes_file: latest_changes.md
```


### Notes:
- Ensure your zip_file path is correct and points to a valid zipped addon file.
- Make sure your release notes file is up-to-date before the run. ( tip: you can extract the latest changelog entry as shown in the example above )

Enjoy automated BNET uploads! 🚀