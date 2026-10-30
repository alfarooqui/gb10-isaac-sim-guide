# Isaac Sim on NVIDIA DGX Spark (GB10) — Build & Enablement Guide

Getting **NVIDIA Isaac Sim 6.1** built from source and running on a **DGX Spark**
(GB10 Grace-Blackwell, aarch64, sm_121), including every gotcha I hit and the fix.
The GB10 software stack is new and the public docs assume x86 — this is a working,
reproducible path on aarch64 + CUDA 13.
<img width="1855" height="1038" alt="Isaac-sim" src="https://github.com/user-attachments/assets/da11c525-027a-4612-9d29-9ae33ca67d57" />

## Baseline
- NVIDIA DGX Spark (GB10 Grace-Blackwell), aarch64
- DGX OS (Ubuntu 24.04), NVIDIA driver 580.x, CUDA 13.0
- Isaac Sim 6.1.0 (built from source)

## 1. Clone & build Isaac Sim
```bash
git clone -b main https://github.com/isaac-sim/IsaacSim.git ~/IsaacSim
cd ~/IsaacSim && git lfs install && git lfs pull
```
Install the aarch64 build deps the default build misses on GB10, **then** build:
```bash
sudo apt update
sudo apt install -y libegl1-mesa-dev libegl-dev \
  libx11-dev libxrandr-dev libxinerama-dev libxcursor-dev libxi-dev libgl1-mesa-dev
./build.sh
```

## 2. Launch the GUI (at a local display)
```bash
cd ~/IsaacSim/_build/linux-aarch64/release
export LD_PRELOAD="$LD_PRELOAD:/lib/aarch64-linux-gnu/libgomp.so.1"
./isaac-sim.sh
```
First launch is slow — it compiles RTX shaders for sm_121.

## 3. Isaac Lab (version-matched to 6.1)
```bash
git clone https://github.com/isaac-sim/IsaacLab.git ~/IsaacLab && cd ~/IsaacLab
git checkout release/3.0.0        # 6.1-aligned; beta2 targets 6.0.x and breaks
ln -sfn "$HOME/IsaacSim/_build/linux-aarch64/release" ~/IsaacLab/_isaac_sim
./isaaclab.sh -i
export LD_PRELOAD="$LD_PRELOAD:/lib/aarch64-linux-gnu/libgomp.so.1"
./isaaclab.sh -p scripts/tutorials/00_sim/create_empty.py --viz kit
```

## 4. Remote access (headless Spark → WebRTC)
SSH can't show the GUI. Stream it:
```bash
cd ~/IsaacSim/_build/linux-aarch64/release
./isaac-sim.streaming.sh
```
Wait for `Isaac Sim Full Streaming App is loaded.`, connect the Isaac Sim WebRTC
Streaming Client to the Spark's IP, open **TCP 49100 + UDP 47998**.

## Known GB10 limitations
- PhysX **GPU** acceleration unsupported on sm_121 → use the **Newton** engine for GPU physics.
- Livestreaming is aarch64-limited → use the WebRTC streaming app, not a remote X session.
- Stock PyTorch wheels cap at sm_120 → install the CUDA-13 / Blackwell build for GPU work.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `build.sh`: `Could not find ... EGL` | `sudo apt install libegl1-mesa-dev libegl-dev` |
| Build fails on X11 `find_library` | install the X11 `-dev` packages above |
| GUI opens but no window over SSH | you're headless — use `isaac-sim.streaming.sh` |
| Isaac Lab script runs, no viewport | add `--viz kit` (headless is the 3.x default) |
| `No module named 'omni.physxfabric'` / `ml_archive` solver error | version mismatch — IsaacLab `release/3.0.0` with Isaac Sim 6.1 |
| `libgomp.so.1` load error | `export LD_PRELOAD=...libgomp.so.1` before launching |

## License
MIT
