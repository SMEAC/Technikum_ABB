# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

An **ABB RobotStudio** station/project (`ABB_IRB_1300.rsproj`, saved with RobotStudio 26.2) for an IRB 1300-7/1.4 robot on an OmniCore C30 controller (RobotWare / RobotControl 7.19.1). There is no build system, linter, or test suite. Code is written in **RAPID** and is loaded, synchronized, and run from RobotStudio (virtual controller) or on the real controller via FlexPendant. Code comments and many identifiers are in German.

## Layout

- `Controller Data/Steuerung_ABB_IRB_1300/` – controller backup (the readable source of truth):
  - `RAPID/TASK1/PROGMOD/*.modx` – RAPID program modules for task `T_ROB1`. **This is where code is edited.**
  - `SYSPAR/*.cfg` – system parameters (EIO, MOC, SYS, etc.). They end in a `#CTRL_SIGNATURE` line; hand edits can invalidate it.
  - `BACKINFO/backinfo.txt` – lists which modules are loaded into the task. A new module must be added here (or loaded via RobotStudio) to be part of the backup.
- `Virtual Controllers/` – RobotStudio virtual controller runtime state (`Steuerung_ABB_IRB_1300` = IRB 1300-7/1.4, `IRB1300F_7_140` = IRB 1300F variant with SafeMove). Generated/binary; don't edit by hand. RAPID there is stored in binary form under `internal/RAPID/PRG1/`.
- `Station/` – station geometry and graphics (`*.rsstnx`, zipped streams). Binary; edit only in RobotStudio.

## RAPID program architecture (`Module1.modx`)

The robot sits in the center of a cell with up to **6 docking modules** arranged around it (60° apart). Each docked module carries an RFID chip identifying which machine it is. A PLC (TIA Portal) and a Keyence scanner coordinate via group I/O.

- RFID chip IDs: `0xF1`..`0xF6` (241..246) → F1 laser, F2 3D printer, F3 delta, F4 CNC, F5/F6 unassigned. TIA Portal uses 1..6, hence `rfid_chip - 0xF0` when writing `Bank_ModulN`.
- `main()` loops forever: resets all `Bank_ModulN` outputs, then either
  - **automatic** (`PERS bool manual_automatic = TRUE`, toggled from an AppStudio FlexPendant web app): reads `RFID_Modul1..6` group inputs, takes the requested chip from `GInput(Fahre_Position)`, and finds which location holds it; or
  - **manual**: the web app sets `rfid_location` directly (1–6, or 7 = go to general home).
- `programm()` has two `TEST` blocks:
  1. by `rfid_location`: `MoveAbsJ` to that slot's `pos_home_N`, remember it in `last_home_position`, load `coordinate_system_module_N` into `current_coordinatesystem`, and set the scanner bank via `SetGO Bank_ModulN`. Then `WaitDI Bank_OK, 1`.
  2. by `rfid_chip`: run the machine-specific routine (`laser`, `printer_3d`, `delta`, `cnc`, `F5`, `F6`) relative to `current_coordinatesystem`, then return to `last_home_position`.
- Each module's work object (`CalibData.modx`) is oriented so its X axis runs along the edge where the module docks, so per-module routines can be written in module-local coordinates. `CalibData.modx` contains both German (`Koordinatensystem_ModulN`) and English (`coordinate_system_module_N`) duplicates with identical values; the code uses the English ones.
- Module routines are currently placeholders (±50 mm X motion with `tool0`). When a real gripper is added, its tool data must replace `tool0` throughout.

Conventions noted in the source comments:
- `read_signals`, `WaitDI Bank_OK`, and the final return `MoveAbsJ last_home_position` are marked "do not change".
- Supporting more than 6 modules requires extending `programm()` cases, `read_signals`, `bank_modul_reset`, the `pos_home_N` targets, and the work objects.
- I/O signals (`RFID_ModulN`, `Bank_ModulN`, `Fahre_Position`, `Bank_OK`) are created via I/O Engineering and are **not** in the backed-up `EIO.cfg`; RobotStudio may flag them as errors in the editor, which is expected.
- `RsTempModule.modx` is a RobotStudio-generated temp module; ignore it.
