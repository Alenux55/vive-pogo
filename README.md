# VIVE Pogo Board

VIVE Pogo Board is a compact charging adapter for an original HTC VIVE
controller. It plugs into the controller's Micro-USB port and brings USB power
and ground out to two large contact pads, allowing the controller to charge
when it is placed in a compatible pogo-pin dock.

## Hardware

The board contains:

- a Hirose ZX80-B-5S Micro-USB Type-B plug;
- two 2 mm contact pads for the dock's pogo pins (VBUS and ground); and
- a 0 ohm resistor across the USB data lines for charger detection.

The PCB is intended to remain attached to the controller as the electrical
interface to the dock. This repository covers that adapter PCB; the surrounding
dock and mechanical design are separate from this project.

## Repository contents

- `vive-pogo.kicad_sch`, `vive-pogo.kicad_pcb`, and `vive-pogo.kicad_pro` —
  KiCad source files.
- `Outputs.kicad_jobset` — project-specific design and manufacturing export
  jobs.
- `RemoteLibrary/` and the library tables — cached symbols, footprints, and 3D
  models required by the design.
- `Releases/Assembly/` — released schematic, assembly drawing, BOM,
  pick-and-place, 3D PDF, and check report outputs.
- `Releases/Fabrication/` — released fabrication drawing, Gerbers, drill files,
  and check report outputs.

## Releases and revisions

Release outputs are generated through Prism from `Outputs.kicad_jobset` and are
versioned with the design. The reusable formatter, templates, drawing sheets,
runtime setup, and maintenance instructions live in the sibling `kicad-lib`
repository under `release-workflow/`.

Revisions use `Rx.y`:

- `x` is the copper/design revision and changes when the physical PCB design
  changes.
- `y` is the BOM/documentation-only revision and may change without changing
  the copper design.
