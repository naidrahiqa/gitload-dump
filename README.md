# gitload-dump

A single-purpose GitHub Actions workflow that downloads an OTA package, extracts its `payload.bin` partitions with [payload-dumper-go](https://github.com/ssut/payload-dumper-go), and publishes the resulting `.img` files as a GitHub Release — no local setup, no build machine needed.

The whole repository is one file: [`.github/workflows/dump.yml`](.github/workflows/dump.yml) (`Dump OTA Payload`). It is triggered manually (`workflow_dispatch`) only.

## Features

- **Direct OTA download** with `aria2c` (16 connections), falling back to `curl --retry 3` if aria2c fails.
- **Automatic device codename detection** from the OTA filename:
  - `*-ota-*` → text before `-ota-`
  - `miui_*Global_*` → text between `miui_` and `Global_`
  - `lineage-*.zip` → the last `-` separated token
  - otherwise: the filename without extension (spaces become `_`)
- **Selective dumping**: pass a comma-separated partition list, or leave it empty to dump every partition in the payload.
- **Disk space preparation**: removes preinstalled toolchains (`dotnet`, `ghc`, Android SDK, CodeQL) from the runner before downloading the OTA.
- **Release publishing**: creates (or updates) a GitHub Release with a generated table of files, sizes and download links.
- **Concurrency control**: runs are grouped by OTA URL (`cancel-in-progress: false`), so two runs against the same URL queue instead of racing.

## Inputs

| Input | Required | Default | Description |
| ----- | -------- | ------- | ----------- |
| `ota_url` | yes | — | Direct URL to the OTA `.zip` file (the zip must contain `payload.bin`). Query strings and SourceForge `/download` suffixes are stripped before deriving the filename. |
| `partitions` | no | `""` | Comma-separated list of partitions to dump (e.g. `system,vendor,boot`). Empty = all partitions. Spaces are removed. |

## Outputs

- Dumped `*.img` files uploaded to a **GitHub Release** tagged `dump-<ota-filename>` (e.g. `dump-ota_alioth_global`), titled `<device> — <filename>` and marked `--latest`.
- Release notes generated automatically: device, source OTA URL, UTC dump timestamp, partition selection, and a table of every dumped image with its size and download link.
- Re-running the same URL overwrites the assets on the existing tag (`gh release upload --clobber`).

## Usage

1. Open the repository's **Actions** tab.
2. Select the **Dump OTA Payload** workflow in the left sidebar.
3. Click **Run workflow**.
4. Fill in `ota_url` with a direct link to the OTA `.zip`, and optionally `partitions`.
5. Wait for the run to finish, then grab the images from the **Releases** page.

## Behavior details & limits

- Runner: `ubuntu-latest`, **job timeout is 120 minutes** (includes OTA download + dump).
- The URL **must** point at a `.zip`; the run fails early otherwise.
- **2 GiB per file limit**: images larger than 2147483648 bytes are not uploaded (GitHub Release asset limit); they are still listed under a "not uploaded" section in the release notes. If nothing fits under the limit, the release is created with notes only.
- Uses payload-dumper-go **v2.0.2**, installed from its official release tarball.
- Requires `contents: write` permission (provided via the default `GITHUB_TOKEN`).

## Requirements

- A GitHub account with permission to run workflows on this repository.
- A direct (non-HTML) download URL to an OTA zip containing `payload.bin`.

## Notes

No source code to build — everything happens on the runner. If you fork the repository, make sure the workflow file exists on the default branch before dispatching it.
