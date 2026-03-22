# Frigate iGPU Decode vs. Detection

## The Problem

Enabling VAAPI hardware decode (`preset-vaapi`) globally in Frigate caused crash loops with `Failed to sync surface` errors. The Intel HD 530 iGPU was being overloaded — asked to simultaneously decode 8 video streams (4 cameras × 2 roles) AND run OpenVINO AI detection.

## The Setup

Frigate v0.16.4 running in an LXC on pve1 (i7-6700T with Intel HD 530 iGPU). Four PoE cameras, each with a main stream (5MP) and sub stream (640×480). OpenVINO AI detection was also running on the iGPU at ~12ms inference time.

## What Happened

1. `preset-vaapi` was enabled globally, telling FFmpeg to use the iGPU for decoding all video streams
2. With 4 cameras × 2 streams = 8 simultaneous VAAPI decode sessions on the iGPU
3. OpenVINO detection was also running on the same iGPU
4. The iGPU couldn't handle the combined load — surface allocation failures caused Frigate to crash and restart in a loop

A secondary issue was discovered at the same time: per-camera `hwaccel_args: []` overrides in the config were silently disabling the global VAAPI setting. This meant some cameras were CPU-decoding while others were fighting for iGPU time, creating an inconsistent and confusing load profile.

## The Fix

**Prioritise the iGPU for AI detection, not video decode.** The sub streams (640×480) are trivially light for CPU decode — there's no reason to waste iGPU resources on them. The fix:

1. Disabled VAAPI globally (`hwaccel_args: []`)
2. Removed all per-camera `hwaccel_args` overrides (which were masking the global setting)
3. Let the CPU handle all stream decoding
4. Reserved the iGPU exclusively for OpenVINO inference

Post-change results: CPU usage settled at ~29% total (223% across 8 threads), OpenVINO inference steady at ~12ms. The i7-6700T handles four 5MP streams plus detection without breaking a sweat.

## The Lesson

**On a single iGPU, you have to choose:** hardware decode OR AI inference. Don't try to run both — the Intel HD 530 doesn't have enough execution units to serve 8 decode sessions and an OpenVINO model simultaneously.

The counterintuitive takeaway: **disabling** hardware decode actually improved overall system stability and didn't noticeably increase CPU usage. Modern CPUs are more than capable of software-decoding multiple 640×480 sub streams. Save the iGPU for the work that actually needs it — AI inference.

Also: per-camera `hwaccel_args` in Frigate's config **overrides** the global setting, it doesn't supplement it. If you see `hwaccel_args: []` on individual cameras, that's disabling hardware decode for those cameras regardless of your global `preset-vaapi` setting.
