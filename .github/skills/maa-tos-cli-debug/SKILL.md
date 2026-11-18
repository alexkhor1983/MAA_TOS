---
name: maa-tos-cli-debug
description: Debug and verify MAA_TOS MaaFramework pipelines on the Android emulator, prioritizing maafw-cli for fast targeted inspection and node-level runs, with MaaPiCli for full task verification. Use when modifying game automation, testing pipeline nodes, diagnosing OCR or interrupt failures, or interpreting MaaFramework logs.
---

# MAA_TOS CLI Debugging

Use the project’s MaaFramework runner to verify automation. Do not inject game taps with `adb shell input`, shell scripts, or another controller when the user requested native framework behavior. MaaFramework may use ADB internally for its controller and screenshots.

## Available runners

- **Default to `maafw-cli`** for fast, focused debug: screen OCR/reco and screenshots, pipeline validate/show, and running the smallest relevant pipeline entry with JSON output. Use `MaaPiCli` when the full Project Interface task needs end-to-end verification or as a fallback.
- `maafw-cli --version` prints `0.1.6`. The normal Codex runner could not start its Windows Store Python shim, but invocation with broader process access succeeded; no reinstall was needed. If the regular command fails to spawn, retry with the available broader process access instead of reinstalling Python.
- ADB was not initially on this runner's `PATH`. Derive the SDK `platform-tools` directory from `install/config/maa_pi_config.json`'s `adb.adb_path`, add that directory to `PATH`, then restart the maafw-cli daemon so device discovery inherits it. Verified commands used `maafw-cli -v device adb`, connected `emulator-5554` as session `tos`, and ran `maafw-cli --on tos --json ocr` successfully.
- `install/MaaPiCli.exe` is the proven full-task fallback for this checkout. Its task selection is configured by `install/config/maa_pi_config.json`.
- `uvx --offline maafw-cli` also failed in the verified Codex runner because its shared uv lock/interpreter was inaccessible. Do not repeatedly retry it without changing the runner context.
- MaaPiCli emitted a plugin-directory “Access is denied” warning in the verified environment, but continued to run the MAA task. Check whether controller connection and task execution succeed before treating that warning as fatal.

## Before a run

1. Read `assets/interface.json` and the relevant pipeline to identify the exact task entry and resource label.
2. Back up `install/config/maa_pi_config.json` before changing task selection for MaaPiCli. Preserve controller, resource, and task option fields. MaaPiCli expects each task's `option` to be an array; `{}` is invalid.
3. If the task was changed in `assets/resource/pipeline/`, copy that pipeline into `install/resource/pipeline/` before launching MaaPiCli. Parse the JSON and compare source/installed hashes.
4. Do not assume the emulator is at the task's starting screen. Use the current run logs or MaaFramework's own recognition flow to establish its state. Avoid launching a long task from an unexpected page.

## Run and inspect

1. Use `maafw-cli` first: connect to the emulator, inspect with OCR/reco, validate the pipeline, and run the smallest relevant pipeline entry. Use JSON output for node-level details. If device discovery is empty after adding platform-tools to `PATH`, restart the CLI daemon and retry. Use MaaPiCli from `install` for full Project Interface task verification or when maafw-cli cannot run; select **Run tasks** and inspect `install/debug/maa.log`.
2. Prefer evidence from the native run: `cur_node_`, each `list=[...]`, recognizer `expected`, `all_results_`, `filtered_results_`, `best_result_`, `result.name`, click action points, and timeout/error lines.
3. For OCR problems, compare the full-screen OCR boxes and recognized strings with the node ROI. A recognized box outside the configured ROI explains why a visually present button is not detected. Use full-screen OCR when the UI position varies and the text is stable; allow the OCR result box to drive the click with `target: true`.
4. For a transient popup, attach its recognizer to the interrupt list of the node that is waiting immediately after the action that can cause the popup. An interrupt on a later readiness node will not help if the pipeline is still waiting in the preceding node.
5. For an OCR phrase embedded in a longer detected line, use a regex expected value only after checking this framework version's OCR behavior in the log. In this project, `.*條件不符合.*` with a low OCR confidence threshold matched the full warning line.
6. Verify the follow-up state after the action, not just the recognizer hit. Confirm that the next node was entered and its expected UI was recognized.

## Resource-load failures: efficient diagnosis

1. Start with `install/debug/maa.log`, not the generic MFA exception. Filter native `[ERR]` lines around the failure timestamp and show only relevant lines. The log can contain NUL bytes: decode as UTF-8 and remove NULs before filtering. Avoid dumping the full log or reading old entries.
2. Check the installed MaaFramework version and compare source and installed pipeline hashes before editing or copying; preserve unrelated changes. In the observed v5.0.4 installation, `[JumpBack]` was treated as a missing node; it requires v5.1 or later. Existing `interrupt` lists are compatible with v5.0.
3. JSON parsing and reference checks do not prove native pipeline validity. Use explicit UTF-8 when reading Chinese JSON in PowerShell (`Get-Content -Encoding UTF8`). If `maafw-cli pipeline validate` repeatedly returns a generic failure, stop retrying it and inspect the native error instead.
4. Verify resource loading without game actions by using the installed MaaFramework native API to create a resource, post each configured bundle, wait for its load job, and destroy the resource. In the observed installation, status `3000` was success and `4000` failure. Check both bundles; a plugin-directory access warning alone did not prevent successful loading. Resource loading does not verify live recognition or actions.

## Turn failures into reusable knowledge

- Keep a short attempt record: starting screen, command/entry node, expected recognition, actual OCR/reco/log result, and resulting screen.
- If the same symptom fails twice, stop repeating the same run. Compare logs against the node lifecycle and screen state; check ROI bounds, OCR text/regex, interrupt attachment timing, pipeline resource sync, and task entry selection.
- Once a cause is proven by logs or a successful replay, add a concise reusable lesson to this skill (or a linked reference). Mark unverified ideas as hypotheses; do not encode guesses as project facts.
- Preserve high-value negative results too, such as an incorrect ROI or launcher/daemon environment mismatch, so future runs skip the same dead end.

## Proven Arena equipment-warning flow

The Arena flow was verified on the connected emulator through MaaPiCli:

1. `Arena_ConfirmSubLeader` waits after tapping the initial friend’s `選擇`; its interrupt list must include `Arena_SubLeaderEquipmentNotice`.
2. `Arena_SubLeaderEquipmentNotice` matches `.*條件不符合.*` over the full screen.
3. `Arena_SubLeaderEquipmentConfirm` OCRs `確定` over the full screen and clicks its detected box.
4. `Arena_SelectAlternateSubLeader` taps the next row at `[360,350]`.
5. `Arena_ConfirmAlternateSubLeader` OCRs `選擇` over the full screen and clicks its detected box. In the verified run, OCR found `[422,918,55,31]` and MaaFramework clicked its center `[452,936]`.
6. `Arena_SubLeaderReady` recognizes the main-team row after the alternate is accepted. If the alternate also triggers the equipment warning, the separate after-alternate handler confirms the dialog and stops rather than looping.

The earlier ROI `[400,770,180,90]` excluded the `選擇` button at y≈918. A tighter crop still produced a stray punctuation recognition; full-screen OCR resolved it.

## Cleanup

- Let the task finish or stop it through MaaPiCli. Exit the CLI cleanly.
- Restore `install/config/maa_pi_config.json` from the backup and remove the temporary backup. Verify the configured task/resource are back to their pre-run values.
- Keep source and installed pipeline copies synchronized after edits.
- Summarize what log evidence passed and what could not be reproduced. Do not claim a live verification if the recognizer/action sequence is absent from the log.
