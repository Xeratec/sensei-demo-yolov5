<!--
SPDX-FileCopyrightText: 2026 ETH Zurich and University of Bologna
SPDX-License-Identifier: CC-BY-4.0

Authors:
- Philip Wiese <wiesep@iis.ee.ethz.ch>
-->

# SENSEI YOLOv5 Demo Firmware

Standalone GAP9 vision demo (head detection on HM0360 frames) for the
SENSEI platform; the nRF5340 acts only as orchestrator / host bridge.
Companion repo to `sensei-demo-multimodal` (adds environmental sensing)
and `sensei-demo-sensorhub` (sensors only). This is the canonical
reference for the **GAP9 deployment pipeline** and YOLOv5 evaluation
numbers.

## What this repo does

- `src_GAP/` (GAP SDK + PMSIS) runs the full vision pipeline: HM0360
  capture → ISP (demosaic + white balance + resize, AutoTiler kernels) →
  YOLOv5 INT8 inference on the NE16 → dequantization across 8 cluster
  cores → NMS on the fabric controller → optional JPEG → frame transmit.
- `src_NRF/` (Zephyr/NCS v2.6.1) orchestrates the GAP9, manages power
  (MAX77654), and forwards GAP9 frames over BLE NUS / USB-CDC / RTT to
  host visualization scripts.

## Inter-SoC link

UART at **921 600 baud** (`src_NRF/main.c`, `src_GAP/uart_comm.c`),
framed with the same 16-bit start/end markers as the multimodal repo:
`0xAB/AC` bbox, `0xBC/BD` JPEG, `0xCD/CE` raw image. Host-side parsing in
`scripts/show_bboxes_{uart,ble}.py` and `scripts/show_bboxes_util.py`
(both share the framing layer).

## GAP9 deployment pipeline

Per-resolution Colab notebook (`src_GAP/networks/YOLOv5/*.ipynb`) trains
on Roboflow human-head data, exports to TFLite with per-layer INT8 PTQ
(scales/zero-points in `modelInfos.h`), then AutoTiler v3 emits cluster
kernels in `src_GAP/src/networks/YOLOv5/generated_<NxN>/model.c`. 8
input variants (64² → 512², ~410–450 kB each) are pre-generated; pick by
setting `YOLO_INPUT_SIZE` in `src_GAP/CMakeLists.txt`. Frequencies are
hard-coded in `CMakeLists.txt` lines 34–36 (cluster + FC 370 MHz,
periph 225 MHz, 0.8 V).

## Performance numbers (verified from `src_NRF/serial_log.log`)

- Pure CNN inference: **22.5–24.6 ms @ 512² INT8** on NE16 (~41–44 Hz if
  isolated).
- Full pipeline @ 512² with JPEG: **474–541 ms (1.85–2.11 FPS)**.
  Preprocessing (demosaic + WB + resize on cluster) dominates at
  316–381 ms; HM0360 acquisition is a fixed 131 ms; dequantization 2–4
  ms; NMS 0–2 ms.
- Quoting only "22 ms inference" understates the loop; always disclose
  whether the ISP path is included.

The serial log itself contains **227 inference iterations** captured
between 2025-03-21 and 2025-05-23 — useful as a primary data source for
journal-paper timing tables.

## Build

- nRF: `west build -b nrf5340_senseiv1_cpuapp` in `src_NRF/`.
- GAP9: `make run` / `make flash` in `src_GAP/` (wrappers around `gapy`).
  `requirements-streaming.txt` lists host-side deps (opencv, pyserial,
  bleak, coloredlogs).

## Gotchas

- The `serial_log.log` mixes two phases: early runs (JPEG disabled, ~10
  FPS without preprocessing breakdown) and late runs (JPEG enabled, full
  per-stage timing). Read the iteration counter, not the line number.
- Detection thresholds live in `src_GAP/src/networks/YOLOv5/yolo_utils.h`
  (`THRESHOLD_BOX 0.4f`, `THRESHOLD_NMS 0.65f`, `MAX_BOXES 20`).
- No environmental sensors are sampled here — for IAQ-related numbers,
  use `sensei-demo-sensorhub` or `sensei-demo-multimodal`.
