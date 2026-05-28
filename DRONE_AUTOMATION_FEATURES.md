# Drone Farming Automation: Features & Capabilities Guide

This document outlines the high-performance features, smart automation mechanics, and command interfaces enabled by the asynchronous Lau drone automation engine.

---

## 📄 Core Automation Features

The codebase elevates the basic drone from a simple sequential grid-walker to a highly intelligent, self-sustaining, and state-aware industrial farming machine.

### 1. Hybrid Adaptive Traversal (Dual-Sweep)
Rather than wasting battery pacing a complete empty grid every iteration, the drone operates a **1-to-4 hybrid sweeping pattern**:
* **Discovery Sweep (1/5 cycles)**: Operates a complete snake scan of the entire field grid to discover freshly cleared spots, map out changes, and plant new seeds.
* **Smart Tree Sweep (4/5 cycles)**: References an internal map memory and completely skips empty coordinates. The drone takes optimized flight paths directly to growing plants, ignoring empty tiles and cutting physical movement times by up to **80%**.

### 2. Physical Growth Hover-Lock
During Tree Sweeps, if the drone encounters an un-matured crop or fruit-bearing tree, it locks its position:
* **Real-Time Polling**: The drone hovers in place over the plant, scanning its growth percentage every **0.5 seconds**.
* **Zero-Delay Harvesting**: The exact split second a crop or fruit hits 100% maturity, the drone harvests or uproots it and immediately flies to the next target.

### 3. Integrated Hydration Management
No crop will ever dry out or halt its growth under this automation:
* **Soil Water Diagnostics**: The drone queries sub-soil moisture thresholds.
* **Hover-Watering**: If a plant’s soil dries up while the drone is hovering and waiting for growth, it will automatically break the hover to water the plant, ensuring continuous growth.

### 4. Self-Managing Inventory & Auto-Market
The drone manages all purchases without locking up the game thread:
* **Seed Refueling**: If empty spots are scanned during a Discovery sweep, it automatically buys seeds from the market up to a safe threshold of **20 per cycle** to avoid over-purchasing or thread locking.
* **Auto-Watering Can Re-stocking**: Monitors the Gear Market stock refresh intervals and automatically purchases up to 20 fresh `Watering_Cans` to ensure the drone never runs dry.

---

## 🕹️ Command Interface & Telemetry

You can interact with the drone in real-time through the chat command console. The drone will instantly adjust its behavior without needing a script restart.

| Command | Action | Description |
|:---|:---|:---|
| `/setseed <seed_name>` | **Seed Switch** | Swaps the preferred crop type. The drone will automatically start buying and planting the new seed type. |
| `/clearreplant` | **Field Wipe** | Instantly forces the drone to clear the entire grid and replant it entirely with the active seed type. |
| `/scanall` | **Forced Map Refresh** | Overrides the hybrid sweep and forces a full Discovery scan on the next iteration to rebuild the internal field model. |
| `/sellall` | **Immediate Cash Out** | Manually forces the drone to sell all harvested inventory instantly. |
| `/status` | **Telemetry Report** | Prints current drone state, active seed types, battery metrics, and coordinate mappings. |

---

## ⚡ Asynchronous Engine Safety & Performance

Running under the `--!ndrone` pragma, the drone gains true multi-threading behavior:
* **Non-Blocking Telemetry**: The drone prints real-time logs, parses user chat commands, and checks stock levels in parallel *while* physically gliding across the sky.
* **Infinite-Loop Capping**: Every market transaction loop and wait routine is bounded (max 20 actions per iteration) and explicitly paced with small `task.wait()` pulses. This prevents the asynchronous threads from starving the game's core processing loop.
