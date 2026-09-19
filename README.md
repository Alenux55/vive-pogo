# VIVE Pogo Board release outputs

`Outputs.kicad_jobset` is the canonical configuration consumed by Prism. Native
KiCad exports are followed by a formatting job inside each destination. Prism
remains the release entry point and versions the results through Git/Assets.
The formatter does not run exports, invoke Git, or publish a separate release.

## Runtime setup

Validated with **KiCad 10.0.6**, and local Prism source commit
`50a4812c7c05aebcc8dda28c64c9382c9fce719d` (`v3.0.2-alpha-3-g50a4812`). The local
Prism Dockerfile defaults to KiCad 10.0.4; the deployed worker version has not
been verified. Use the validated KiCad version for matching output behavior.

The formatter requires Python 3.10+ available as `python`, with:

```text
python -m pip install -r Workflow/requirements.txt
```

Install into the worker interpreter once, not on each release. The pinned
packages are openpyxl 3.1.5 and pypdf 6.10.0. Prism's Dockerfile places
`/app/venv/bin` on PATH; install there, for example with `uv pip install --python
/app/venv/bin/python -r Workflow/requirements.txt` in the deployed image.
Local validation used an existing environment containing those exact packages.
No deployment or automatic package installation is performed by this project.

Prism passes the selected destination to `kicad-cli jobset run` without a
job-type whitelist. KiCad explicitly runs `special_execute` jobs and propagates
failures; its folder handler collects the generated files. This verified path
supports the PDF merge, XLSX formatting and filename conversion. Arbitrary
`.prism.json` workflow definitions remain unsupported and are not used.

## Destinations

| Prism card | Fixed destination ID | Output/Assets root |
| --- | --- | --- |
| Manufacturing | `9e5c254b-cb26-4a49-beea-fa7af8a62903` | `Releases/Fabrication/` |
| Design | `28dab1d3-7bf2-4d8a-9723-bcdd14e1d814` | `Releases/Assembly/` |

Keep these IDs. `.prism.json` selects this Jobset and both roots. The Renders
card is not configured. Releases must not be ignored: Prism commits outputs,
and revision-specific Assets read from Git. Confirm Git synchronization succeeds;
commit/push problems may appear as warnings after successful generation. This
Prism version has no immutable approved-release record. Run from the intended
synchronized source revision without unrelated changes: it stages the checkout.

There are **15 unique jobs**. Fabrication runs DRC, four native exports and
a formatter (6 jobs). Assembly runs ERC, six native exports and a formatter
(9 jobs, including worksheet preparation). One run produces **22 fabrication files and 8 assembly files**.

```text
Releases/
  Fabrication/
    Checks/                         DRC, formatter log
    <title> R<PCB rev> FAB - No Variant (<date> <time>).pdf
    <title> R<PCB rev> STEP - No Variant (<date> <time>).step
    <title> R<PCB rev> CAM/
      physical/documentation Gerbers and native .gbrjob
      Drill/                        Excellon, maps and drill report
  Assembly/
    Checks/                         ERC, worksheet preparation and formatter logs
    No Variant/
      <title> R<PCBA rev> ASY - No Variant (<date> <time>).pdf
      <title> R<PCBA rev> BOM - No Variant (<date> <time>).xlsx
      <title> R<PCBA rev> SCH - No Variant (<date> <time>).pdf
      <title> R<PCBA rev> 3DPCB - No Variant (<date> <time>).pdf
      <title> R<PCBA rev> Pick Place - No Variant (<date> <time>).csv
```

## Names and document identity

Names use `ProjectTitle` and the appropriate existing PCB/PCBA revision.
Timestamps are exactly **(YYYY-MM-DD HH-MM)**, captured once per destination
by the formatter and shared with its BOM metadata. The worker's local clock
and timezone apply. The two destinations can run at different times. Repeated
runs within a minute reuse names; later timestamps accumulate. CAM is updated
in place and its history is retained by Git.

KiCad's native clock cannot format HH-MM. The final native Execute Command job
applies the requested filenames and removes the placement export's `-all-pos`
suffix. Native intermediates remain in `_work/` inside KiCad's temporary
folder and are removed after successful formatting. The helper rejects
execution without `JOBSET_OUTPUT_WORK_PATH`, and rejects a source-directory
target. Only final documents, CAM, check reports and logs are collected.

The project has no named variants. Files, folders and BOM metadata use **No
Variant**, labelling KiCad's empty/base selection rather than creating an
electrical variant. The canonical assembly worksheet retains `${VARIANT}`;
KiCad 10.0.6 renders that field blank for the base selection. Fabrication remains
variant-neutral.

Drawing sheets derive from the existing custom style. Fabrication uses
`ProjectPCBRevision`; assembly uses `ProjectPCBARevision`. Printed sheet dates
use `${CURRENT_DATE}`; existing release-date variables are retained. PCB title
metadata references the PCB revision so the native Gerber job file also receives it.

## Assembly PDF and BOM

Worksheet lookup uses `KICAD_LIB_ROOT` when set, otherwise the `kicad-lib`
directory beside the project checkout. Locally this resolves to
`E:/Documents/GitHub/kicad-lib/drawing-sheets/alex-generic-pcba.kicad_wks`.
For Prism, make the shared library available to the worker (preferably read-only)
and set `KICAD_LIB_ROOT` to its container path, or use the same sibling layout.
The inspected Prism Compose configuration has no dedicated shared-library mount;
that deployment setup is required and was not applied here. There is no download,
embedded-sheet fallback or independently maintained copy.

One native Execute Command job runs `Workflow/finalize_outputs.py --kind
prepare-assembly` immediately before the Top/Bottom export jobs. It reads the
canonical file once as bytes and creates two exclusive temporary files under
`${JOBSET_OUTPUT_WORK_PATH}/_work/`: `assembly-top.kicad_wks` and
`assembly-bottom.kicad_wks`. It replaces `${##}` with 2 first, then `${#}` with
1 or 2. Every other byte, including variables and line endings, is preserved.
Both files are ready before Top is exported; Bottom then uses its own copy.
The native export mirror/layer settings and pypdf merge/bookmarks are unchanged.
Successful finalization removes both generated worksheets before KiCad collects
the destination. On failure any remnants are confined to KiCad's temporary area.
The canonical file is only ever opened for reading, and pre-existing generated
targets are rejected rather than overwritten. No source worksheet is rewritten.

The canonical worksheet controls its own labels: it currently uses
`${ProjectPCBAReleaseDate}` rather than the generation clock and contains no
Top/Bottom label. Page counters and PDF bookmarks identify the two views.

The **single ASY PDF has Top on page 1 and Bottom on page 2**, with bookmarks.
Native jobs plot F.Fab + Edge.Cuts and B.Fab + Edge.Cuts at monochrome 1:1.
Bottom geometry is mirrored before merging; sheet text remains readable.
Both pages derive automatically from the single shared library worksheet
`kicad-lib/drawing-sheets/alex-generic-pcba.kicad_wks`. No page-specific worksheet
is maintained in this repository. Joining preserves vector content, scale and
orientation. DNP parts are sketched/crossed out. Small references require zooming;
footprint geometry and reference positions are unchanged.

`Workflow/BOM-template.xlsx` is an unchanged copy of the supplied sample.
KiCad creates grouped CSV; the final job fills the template, preserving the
title, merged metadata cells, fonts, grey fills and six-column layout.
Column E uses the template width multiplied by 1.20; F uses 1.75. Other widths
are unchanged. Current template baselines are E=22.42578125 and F=41.42578125;
outputs are E=26.9109375 and F=72.4951171875. If a width is absent, the template's
default column width is used, falling back to openpyxl's 13-character default.
Body rows wrap text with height unset, leaving AutoFit to the spreadsheet reader.
No character-count estimator or `math` import remains. Excel's actual AutoFit
rendering is reader-dependent and was not tested with Excel automation. Row 8
retains its 30-point minimum. Print area, repeating rows 1:8 and fit-to-width
behavior are unchanged. Quantities/line numbers are numeric; identifiers/revisions
are text; publication date is an Excel datetime formatted `yyyy-mm-dd hh-mm`.

| Template item | Source |
| --- | --- |
| Project Title | `ProjectTitle` |
| Part Number | `ProjectPartNumber` if defined, otherwise `N/A` |
| BOM Revision | `ProjectPCBARevision` |
| Publication Date | Destination generation timestamp |
| Assembly Variant | `No Variant` for base design |
| Line # | Native `${ITEM_NUMBER}` |
| Quantity | Native `${QUANTITY}` |
| Designator | Native grouped `Reference` |
| Description | `Description` |
| Manufacturer | `Manufacturer` |
| Manufacturer Part Number | `Manufacturer Part Number` |

Grouping uses manufacturer and part number, with Reference sorting and DNP
excluded. The template and actual field both omit the earlier trailing `1` from
the part-number heading. No symbol fields are added. P101/R101 appear; existing
BOM-excluded test points remain excluded. Strings starting `=` remain literal
text, not Excel formulas. Missing exports or unexpected CSV headers fail formatting.

## Fabrication, CAM, drills and 3D

The fabrication PDF combines F.Cu, Edge.Cuts and User.1 with the fabrication
sheet. User.1 `Fab_Notes` holds board-level notes; User.2 `Router_VCut_Panel` is
reserved and empty; User.3 `Fab_Title_Block` has independent PCB text/rectangle
title/revision/date graphics. User.4 is unchanged. No tolerances or panel
instructions are invented. Notes stay off F.Fab. Documentation is not overlaid
onto manufacturing Gerbers.

Native Protel extensions are enabled for copper/mask/paste/silkscreen; outline
uses `.gm1`. User layers and maps remain `.gbr`, without custom
`.gm2/.gm3/.gm33/.gd1` conversion. Copper selection covers F.Cu, In1.Cu through
In30.Cu and B.Cu, intersected by KiCad with enabled layers. This board generates
two copper files and both sides of mask/paste/silkscreen. The `.gbrjob` contains
actual file functions, stackup, thickness, defined finish and PCB revision;
it is not a combined CAM README/drill manifest. Its outline size includes
stroke width and must not be interpreted as a mechanical tolerance.

Drills are metric Excellon, decimal, separate PTH/NPTH, with routed slots,
Gerber maps and a drill report under the named CAM folder's `Drill/` directory.
Gerbers, drills, STEP and placements use the existing auxiliary/drill origin
(134.25, 102.25) mm. KiCad determines real drill layer pairs automatically.
Current board: 6 plated holes including 4 slots, and 2 nonplated slots.
Native `.drl`/`.rpt` names remain; there is no separate `.LDP` output.

Fabrication STEP includes board body and all defined component models, with
DNP/unspecified items included, substitution enabled, empty component filter
and no selected variant. Connector/resistor models were verified, including
retaining a DNP resistor in a disposable test. Test points define no 3D models.

Assembly 3D PDF is native U3D, with board, fitted component models, mask and
silkscreen enabled, and DNP excluded. It requires a 3D-capable PDF reader;
ordinary previews can be blank. Native output produces pypdf cross-reference
warnings; desktop interactive viewing remains untested. The formatter copies
it unchanged. The filename supplies PCBA revision/variant/date; custom internal
3D PDF title metadata remains deferred.

Pick-and-place is CSV, mm, both sides, existing origin, with DNP/BOM-excluded
items omitted. SMD-only and through-hole-pad exclusion are disabled, retaining
mixed-technology P101. Bottom X is not artificially negated.

## Checks and remaining limitations

ERC/DRC include errors and warnings (`severity: 48`), with `fail_on_error: true`.
Existing project severities/exclusions are unchanged. DRC neither refills zones
nor saves the PCB. Reports list ignored rules. KiCad 10.0.6 Jobset DRC requests
parity but skips it with a netlist warning; direct `pcb drc --schematic-parity`
works. Check parity in KiCad before release until that upstream issue is fixed.

Prism omits `--stop-on-error`: rule-check failures can leave generated local
files, but nonzero exit prevents Git synchronization. Failed runs are not
approved releases. Formatter jobs use `ignore_exit_code: false`; missing
packages, incomplete exports or invalid BOMs fail the destination. KiCad does
not collect that destination after a formatter error. Older Assets are not
automatically invalidated by failure.

Automatic whole-package iteration over future variants remains unavailable.
Named variants need explicit native job selectors, matching drawing-sheet labels,
separate intermediate inputs and corresponding formatter configuration inside
this canonical Jobset. The helper currently expects one base assembly set per
destination. Fabrication stays variant-neutral.

Source ZIP and custom dynamic CAM README remain outside this formatter's scope.
Native archive destinations, `.gbrjob` and drill reports remain available. PDF
merging, exact date names and the custom BOM header are now implemented inside
Prism's existing Jobset rather than through a separate release mechanism.

## Validation and operation

With the configured Python on PATH, run from the project root:

```text
kicad-cli jobset run -f Outputs.kicad_jobset vive-pogo.kicad_pro
kicad-cli jobset run -f Outputs.kicad_jobset --output 9e5c254b-cb26-4a49-beea-fa7af8a62903 vive-pogo.kicad_pro
kicad-cli jobset run -f Outputs.kicad_jobset --output 28dab1d3-7bf2-4d8a-9723-bcdd14e1d814 vive-pogo.kicad_pro
python -m unittest discover -s Workflow -p test_finalize_outputs.py -v
```

Validated on 2026-09-18: 7/7 fabrication and 10/10 assembly jobs passed. Both merged
PDF pages were rendered and view order, numbering and readable title text checked.
XLSX metadata, rows and numeric/text/date types were verified after reopening,
and rendered against the template. Eight formatter tests cover merge order,
metadata/types, literal text, missing exports, schema mismatch and context/path
guards, exact worksheet substitutions and missing canonical sheets. Saved XLSX
XML confirms item rows have neither `ht` nor `customHeight`; source SHA-256 hashes
confirm unchanged PCB, schematic, project, template and canonical worksheet.
Actual local Prism handler/resolver/Assets code is exercised with real
KiCad subprocesses but stubbed server context/Git synchronization. No remote
API, deployment, commit or push was performed.

Earlier disposable tests verified DNP exclusion from assembly BOM/PnP/3D,
retention in fabrication STEP, crossed-out graphics and separate PCB/PCBA
revisions. A routing fault correctly returned exit 6. Direct DRC/parity found
zero violations, unconnected items or parity issues. Original electrical design
objects were preserved. Output refinements do not alter connectivity, routing,
placement, footprints, values, copper or dimensions.

Review/commit the configuration, install worker dependencies, synchronize the
intended revision, check parity and run both Prism cards. Reload open documents
before saving stale in-memory documentation settings. Earlier validation layouts
are preserved in temporary storage outside the repository.

References: official [KiCad 10.0.6 jobs](https://github.com/KiCad/kicad-source-mirror/tree/10.0.6/common/jobs),
[job runner](https://github.com/KiCad/kicad-source-mirror/blob/10.0.6/kicad/jobs_runner.cpp),
[folder handler](https://github.com/KiCad/kicad-source-mirror/blob/10.0.6/common/jobs/jobs_output_folder.cpp),
and local Prism `project_service.py`, `path_config_service.py`, `file_service.py`
and `api/projects.py` for dispatch, Git synchronization and Assets behavior.
