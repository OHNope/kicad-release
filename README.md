# kicad-release

Shared release pipeline for KiCad 10 board repos. A board repo calls the reusable workflow
[`.github/workflows/pcb-release.yml`](.github/workflows/pcb-release.yml). Changes pushed here reach every
board on its next release.

## Board repo setup

The board repo needs exactly one `.kicad_pro` at its root, next to the `.kicad_sch` and
`.kicad_pcb` of the same name. Add `.github/workflows/pcb-release.yml`:

```yaml
# Release pipeline: https://github.com/OHNope/kicad-release
name: PCB release

on:
  push:
    tags: ['v*']
  workflow_dispatch:
    inputs:
      fab:
        description: Build the fabrication outputs instead of the design release
        type: boolean
        default: false

permissions:
  contents: write

jobs:
  release:
    uses: OHNope/kicad-release/.github/workflows/pcb-release.yml@main
    with:
      fab: ${{ inputs.fab == true }}
    secrets: inherit
```

Secrets and variables are read from the board repo (see [Google Drive upload](#google-drive-upload)).

## Tags

Each tag pushed runs the pipeline once. KiBot runs ERC/DRC first, then builds one output group:

| Tag        | KiBot group | Contents                                                                 |
|------------|-------------|--------------------------------------------------------------------------|
| `v1.0`     | `design`    | Schematic/PCB PDFs, BOM, interactive BOM, STEP, schematic/PCB diffs against the previous `v*` tag. Drive also gets the whole committed project tree |
| `v1.0-fab` | `fab`       | Gerbers, drill files, fab zip, pick-and-place, BOM                       |

ERC/DRC errors only warn on a design release: the run still publishes, shows the counts as
annotations and in the run summary, and ships the full HTML/JSON reports with the outputs. They stop
a `-fab` release, so Gerbers that fail DRC never reach the fab house. A board without a
`.kicad_pcb` yet gets a schematic-only design release (schematic PDF, BOM, schematic diff); a
`-fab` tag fails until the board exists.

Gerbers ship only with a `-fab` tag. That tag must point at the same commit as its design tag,
so tag the design release first:

```bash
git tag v1.0 && git push origin v1.0
```

```bash
git tag v1.0-fab v1.0 && git push origin v1.0-fab
```

Each tag gets its own GitHub Release, and the design release stays "latest". Push at most two
tags at once: GitHub starts no workflow when one push carries more than three tags, and runs
are serialized, so a third queued run cancels the one waiting before it.

To test without tagging, use **Actions → PCB release → Run workflow** in the board repo. Tick
`fab` to build the fabrication group. That run skips the GitHub Release.

## KiBot config

[`kibot/default.kibot.yaml`](kibot/default.kibot.yaml) is used unless the board repo has its own
`.kibot.yaml` at the root. Start an override by copying the default; it must keep the
`design`, `design_sch` and `fab` groups and the `DIFF_REF`/`XRC_DONT_STOP` definitions. A board with an override stops receiving changes to the default.

## Google Drive upload

In the board repo, open Settings → Secrets and variables → Actions. Set the variable
`GDRIVE_FOLDER_ID` to the ID at the end of the folder URL
(`drive.google.com/drive/folders/<ID>`), then add one credential:

- **Your account** works for any folder you can edit, including ones under "Shared with me". Run
  `rclone authorize "drive"` locally once, sign in, and save the printed token JSON as the
  secret `GDRIVE_TOKEN` in each board repo. Reuse that token instead of authorizing again per
  repo: Google keeps at most 100 per account, and a token unused for six months stops working.
  Uploads are owned by you. The token grants full access to that account's Drive, so anyone
  who can edit the board repo's workflows can use it.
- **Service account** works only in a shared drive. It has no storage quota, so writes into a
  My Drive folder fail with `storageQuotaExceeded`. Save the key JSON as the secret
  `GDRIVE_SA_JSON`, set the variable `GDRIVE_TEAM_DRIVE_ID`, and add the service account as a
  shared-drive member. `GDRIVE_FOLDER_ID` may stay empty to use the drive root.

`GDRIVE_TOKEN` wins when both are set. The upload is skipped when neither variable is set and
fails when a variable is set without a credential. Drive layout:

```text
<folder>/<repo>/v1.0/
  <repo>_v1.0.zip, schematic/PCB PDFs, BOM, iBOM, STEP, diffs, ERC/DRC reports
                    from v1.0, flat
  source/           from v1.0: every committed file at the tag (.kicad_pro/.kicad_sch/.kicad_pcb,
                    libraries/, fp-lib-table, sym-lib-table, sources/, README.md, ...)
  Gerbers/          from v1.0-fab, flat: both zips, every Gerber and drill file,
                    pick-and-place, BOM, ERC/DRC reports
```

Both runs fail rather than overwrite when two outputs share a file name.

`source/` holds exactly what Git tracks, so anything the board's `.gitignore` excludes (ERC/DRC
reports, `.kicad-auto/`, backups) stays local. The GitHub Release carries the output zip, and
GitHub attaches the tag's source archives on its own.
