# NaVILA VLN Benchmark for IsaacLab Arena

**Vision-Language Navigation (VLN)** benchmark using the [NaVILA](https://github.com/AnjieCheng/NaVILA) VLM on the H1 humanoid robot in Matterport 3D indoor scenes.

## Architecture

```
 ISAAC SIM CONTAINER                       NAVILA SERVER CONTAINER
 ══════════════════                        ═══════════════════════

 ┌──────────────┐  ┌───────────────────┐
 │ Matterport   │  │ VLN-CE R2R Data   │
 │ Scene (.usd) │  │ (.json.gz)        │
 └──────┬───────┘  └───────┬───────────┘
        │                  │
        ▼                  ▼
 ┌──────────────┐  ┌───────────────────┐
 │ H1 Robot     │  │VlnR2rMatterport   │
 │ + Head Cam   │  │Task               │
 │ + Follow Cam │  │ scene_filter      │
 └──────┬───────┘  │ termination       │
        │          │ metrics (OS, SPL...) │
        │          └───────┬───────────┘
        │                  │
        ▼                  ▼
 ┌─────────────────────────────────┐
 │  VlnVlmLocomotionPolicy        │
 │                                 │
 │  ┌─ High-level ──────────────┐  │
 │  │  Send 1 RGB frame ────────┼──┼──► ZeroMQ ──►┐
 │  │  Receive vel_cmd ◄────────┼──┼──◄ ZeroMQ ◄──┤
 │  └────────────┬──────────────┘  │              │
 │               │                 │              │
 │  ┌────────────▼──────────────┐  │              │
 │  │  Low-level: RSL-RL        │  │              │
 │  │  Inject vel_cmd → obs     │  │              │
 │  │  NN forward → joint act   │  │              │
 │  └────────────┬──────────────┘  │              │
 └───────────────┼─────────────────┘              │
                 │                                │
                 ▼                                │
 ┌─────────────────────────────┐    ┌─────────────▼────────────┐
 │  PhysX GPU Simulation       │    │  NaVilaServerPolicy      │
 │  joint act → robot moves    │    │                          │
 │  → new camera RGB           │    │  1. Receive 1 RGB frame  │
 └─────────────────────────────┘    │  2. Append to history    │
                                    │  3. Uniform sample 8     │
                                    │     from full history    │
                                    │  4. VLM inference        │
                                    │  5. Parse → vel_cmd      │
                                    │  6. Return vel_cmd + dur │
                                    └──────────────────────────┘
```

## Quick Start

### 1. Start the NaVILA VLM Server

```bash
# Option A: Use the launch script (rebuilds Docker if needed)
bash docker/run_vln_server.sh -m ~/models/navila/navila-llama3-8b-8f --port 5555

# Option B: Manual start with code mounted (no rebuild)
docker run --rm -d --gpus all --net host \
    --name vln_policy_server_container \
    -v ~/models:/models \
    -v ~/IsaacLab-Arena/isaaclab_arena_navila:/workspace/isaaclab_arena_navila \
    -v ~/IsaacLab-Arena/isaaclab_arena/remote_policy:/workspace/isaaclab_arena/remote_policy \
    vln_policy_server:latest \
    --host 0.0.0.0 --port 5555 --timeout_ms 15000 \
    --policy_type isaaclab_arena_navila.navila_server_policy.NaVilaServerPolicy \
    --model_path /models/navila/navila-llama3-8b-8f
```

Wait for: `listening on tcp://0.0.0.0:5555`

### 2. Run VLN Evaluation (Client)

```bash
# Enter Isaac Sim container
bash docker/run_docker.sh

# Inside the container:
/isaac-sim/python.sh -u -m isaaclab_arena.evaluation.policy_runner \
    --enable_cameras --num_envs 1 \
    --policy_type isaaclab_arena.policy.vln.vln_vlm_locomotion_policy.VlnVlmLocomotionPolicy \
    --remote_host localhost --remote_port 5555 \
    --ll_checkpoint_path isaaclab_arena/policy/vln/pretrained/h1_navila_locomotion.pt \
    --ll_agent_cfg isaaclab_arena/policy/vln/pretrained/h1_navila_agent.yaml \
    --num_episodes 10 \
    h1_vln_matterport \
    --usd_path /path/to/matterport_usd/zsNo4HB9uLZ/zsNo4HB9uLZ.usd \
    --r2r_dataset_path /path/to/vln_ce_isaac_v1.json.gz
```

Add `--headless` for batch evaluation without GUI.

## Matterport Scene Collision

The default VLN-CE-Isaac Matterport USD scenes contain **visual meshes only**
and do not include reliable PhysX collision geometry. By default an invisible
ground plane at z=0 provides basic floor support.

To enable experimental mesh collision, generate a collision-proxy USD from the
visual scene and pass it as an overlay:

```bash
# 1. Export visual USD to OBJ (inside Isaac Sim container)
/isaac-sim/python.sh isaaclab_arena/scripts/assets/export_matterport_collision_proxy.py \
    --input_path /path/to/matterport_usd/zsNo4HB9uLZ/zsNo4HB9uLZ.usd \
    --output_path /tmp/zsNo4HB9uLZ.obj

# 2. Convert OBJ to collision-only USDA (runs on host Python, no Isaac Sim needed)
python3 isaaclab_arena/scripts/assets/obj_to_usd_mesh.py \
    --input_obj /tmp/zsNo4HB9uLZ.obj \
    --output_usd /path/to/zsNo4HB9uLZ_collision_proxy.usda

# 3. Run evaluation with collision overlay (disables default ground plane)
/isaac-sim/python.sh -u -m isaaclab_arena.evaluation.policy_runner \
    ... \
    h1_vln_matterport \
    --usd_path /path/to/zsNo4HB9uLZ/zsNo4HB9uLZ.usd \
    --r2r_dataset_path /path/to/vln_ce_isaac_v1.json.gz \
    --use_global_matterport_prim \
    --disable_matterport_ground_plane \
    --matterport_collision_usd_path /path/to/zsNo4HB9uLZ_collision_proxy.usda \
    --robot_root_height_offset 1.0
```

Collision-related CLI flags:

| Flag | Default | Description |
|------|---------|-------------|
| `--use_global_matterport_prim` | off | Spawn Matterport at `/World/matterport` with `collision_group=-1`. Required for collision overlay and height scanner. |
| `--disable_matterport_ground_plane` | off | Remove the invisible z=0 ground plane. Use when a collision overlay provides floor support. |
| `--matterport_collision_usd_path` | None | Path to a hidden collision-only USD layered under the visual scene. |
| `--enable_matterport_child_mesh_colliders` | off | Explicitly apply collision schemas to descendant Mesh prims of the visual USD. |
| `--matterport_mesh_collider_type` | `triangle` | Mesh collider approximation: `triangle`, `sdf`, or `convex_decomposition`. |
| `--robot_root_height_offset` | 1.0 | Added to dataset `start_position.z` when resetting the robot (H1 pelvis is ~1m above the floor). |

## Training Your Own Low-Level Locomotion Policy

The VLN pipeline uses a pre-trained RSL-RL locomotion policy that converts
velocity commands `[vx, vy, yaw_rate]` into joint-position actions. A default
checkpoint is included at `isaaclab_arena/policy/vln/pretrained/`.

To train a replacement checkpoint with the built-in rough-terrain environment:

```bash
docker run --rm --gpus all --net host --entrypoint /bin/bash \
  -v "$(pwd):/workspaces/isaaclab_arena" \
  -v "/home/$USER:/home/$USER" \
  isaaclab_arena:latest -lc "
    cd /workspaces/isaaclab_arena && \
    /isaac-sim/python.sh -u isaaclab_arena/scripts/reinforcement_learning/train.py \
      --headless \
      --num_envs 4096 \
      --max_iterations 1000 \
      --save_interval 50 \
      --experiment_name h1_base_rough \
      --agent_cfg_path isaaclab_arena/policy/rl_policy/h1_base_rough_policy.json \
      h1_base_rough
  "
```

Checkpoints are saved under `logs/rsl_rl/h1_base_rough/<timestamp>/`.
To use a newly trained checkpoint for VLN evaluation, point to it with:

```bash
--ll_checkpoint_path logs/rsl_rl/h1_base_rough/<timestamp>/model_<iter>.pt
--ll_agent_cfg logs/rsl_rl/h1_base_rough/<timestamp>/params/agent.yaml
```

The training environment preserves the same 69-dim proprioceptive layout as the
VLN evaluation, so the velocity-command injection at `obs[:, 9:12]` remains
compatible without code changes.

## Sensor Configuration

### Camera Positions (CLI-configurable)

| Camera | Default Position (pelvis frame) | FOV | CLI Override |
|--------|--------------------------------|-----|-------------|
| Head camera | `(0.1, 0.0, 0.5)` | 54 deg | `--head_camera_offset_xyz X Y Z` |
| Follow camera | `(-1.0, 0.0, 0.57)` | 100 deg | `--follow_camera_offset_xyz X Y Z` |

### Optional Sensors

| Flag | Description |
|------|-------------|
| `--enable_head_camera_depth` | Add depth output to the head camera (not sent to VLM by default). |
| `--enable_height_scanner` | Enable a raycast-based height scanner on the pelvis. Requires `--use_global_matterport_prim`. |
| `--height_scanner_debug_vis` | Visualize the height-scanner rays in the viewport. |
| `--camera_resolution N` | Camera resolution in pixels (default: 512). |
| `--no_follow_camera` | Disable the third-person follow camera. |
| `--use_tiled_camera` | Use TiledCamera for parallel multi-env evaluation. |

### STOP Diagnostic Flags

| Flag | Description |
|------|-------------|
| `--ignore_vlm_stop` | Never let the VLM STOP command terminate an episode. Useful for diagnosing early-stop behaviour. |
| `--min_vlm_stop_distance D` | Ignore VLM STOP when distance-to-goal exceeds `D` meters. Negative value (default) disables. |
| `--debug_vln` | Enable verbose VLM query logging and debug frame saving. |
| `--save_interval N` | Save camera frames every N steps to `--save_dir`. |
| `--save_dir DIR` | Directory for saved camera frames. |

## CLI Parameters

### Client Parameters (before `h1_vln_matterport`)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--num_envs` | 1 | Must be 1 (multi-env not yet supported). |
| `--policy_type` | (required) | `isaaclab_arena.policy.vln.vln_vlm_locomotion_policy.VlnVlmLocomotionPolicy` |
| `--remote_host` | localhost | VLM server hostname. |
| `--remote_port` | 5555 | VLM server port. |
| `--ll_checkpoint_path` | (required) | RSL-RL locomotion checkpoint (.pt). |
| `--ll_agent_cfg` | (required) | RSL-RL agent config (.yaml). |
| `--warmup_steps` | 200 | Low-level policy warmup steps. |
| `--num_episodes` | (required) | Number of episodes to evaluate. |

### Environment Parameters (after `h1_vln_matterport`)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--usd_path` | (required) | Path to Matterport USD scene file. |
| `--r2r_dataset_path` | (required) | Path to VLN-CE R2R dataset (.json.gz). |
| `--episode_start` | None | Start index within filtered episodes. |
| `--episode_end` | None | End index (exclusive) within filtered episodes. |
| `--episode_length_s` | 60 | Max episode duration in seconds. |
| `--success_radius` | 3.0 | Success distance threshold (meters). |

### Server Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--model_path` | (required) | Path to NaVILA / LLaVA model checkpoint. |
| `--port` | 5555 | Server listening port. |
| `--num_video_frames` | 8 | Number of frames sent to VLM per query. |
| `--conv_mode` | llama_3 | LLaVA conversation template. |
| `--history_padding_mode` | repeat_first | Early-frame padding: `repeat_first` or `black` (NaVILA-Bench style). |
| `--max_history_frames` | 200 | Maximum frames kept in server-side history. |
| `--max_new_tokens` | 80 | Maximum generated tokens per VLM query. |

## Metrics

| Metric | Description |
|--------|-------------|
| **Oracle Success** | Habitat/VLN-CE-style oracle success: `1[min_t d_t < r]`, i.e. the trajectory is successful if it ever enters the success radius. This is not STOP-gated. |
| **Success** | Habitat-style success: the agent emits `STOP` and the final distance-to-goal is below the success radius. |
| **SPL** | Habitat-style SPL: `S_i * l_i / max(p_i, l_i)`, where `S_i` is STOP-gated success, `l_i` is the dataset shortest-path geodesic distance when available, and `p_i` is actual path length. |
| **Distance-to-Goal** | Approximate geodesic distance from the final position to the goal in the XY plane, estimated from the reference path. |
| **Path Length** | Total distance traversed (XY only). |

The benchmark follows a `DONE/STOP`-style evaluation protocol: without an explicit `STOP`, an episode is counted as unsuccessful even if the agent passes near the goal.

`Oracle Success` follows the Habitat/VLN-CE structure exactly, but in the current Isaac Sim benchmark the underlying per-step distance `d_t` still comes from the same approximate distance surrogate used by `Distance-to-Goal`, because a true simulator geodesic query is not yet available.

### Habitat / VLN-CE Alignment Notes

The intent of this benchmark is to follow the standard Habitat / VLN-CE metric semantics as closely as possible:

- `Distance-to-Goal`: Habitat defines this as the simulator geodesic distance from the current agent position to the goal.
- `Success`: Habitat defines this as `STOP` called and final `Distance-to-Goal < success_radius`.
- `Oracle Success`: Habitat / VLN-CE defines this as `1[min_t d_t < r]`, where `d_t` is the per-step distance-to-goal measure and `r` is the success radius.
- `SPL`: Habitat defines this as `S_i * l_i / max(p_i, l_i)`, where `l_i` is the shortest-path geodesic distance from episode start to goal.

Current Arena implementation status:

- `goal radius` is read from the dataset episode's `goals[0].radius` when available.
- `SPL` uses the dataset episode's `info.geodesic_distance` as the shortest-path baseline when available.
- `Distance-to-Goal` is still an approximate surrogate, not a true simulator/navmesh geodesic query.
- Therefore, `Success` and `Oracle Success` already match the official *structure* of Habitat / VLN-CE, but the underlying distance value is still approximate until a true Isaac Sim navmesh shortest-path query is wired in.

What "approximate distance" means here:

- We do **not** currently ask Isaac Sim for the true shortest traversable path from the robot's current position to the goal.
- Instead, we project the robot onto the ground-truth `reference_path`, find the closest reference waypoint in XY, and add the remaining reference-path length to the goal.
- This is a useful surrogate for debugging and benchmarking, but it is not identical to Habitat's true geodesic distance.
- As a result, values such as `Distance-to-Goal`, `Success`, and `Oracle Success` should currently be interpreted as *Habitat-style metrics computed with an approximate distance backend*.

## Known Limitations

- **`num_envs` must be 1.** Multi-env requires per-env VLM instruction tracking.
- **Scene switching requires restart.** Matterport USD is loaded at initialization.
- **True navmesh geodesic distance is not wired in yet.** The benchmark currently uses a reference-path-based approximate distance surrogate for `Distance-to-Goal`, `Success`, and `Oracle Success`.
- **Some episode start positions are outside the building envelope.** The
  Matterport scan may have incomplete or visually degraded geometry at these
  locations, which can confuse the VLM and cause the robot to get stuck.
  Consider filtering edge-case episodes or adjusting `--episode_start` / `--episode_end`.
- **Collision proxy is experimental.** The exported proxy mesh may not perfectly
  match the original building geometry. Use the default ground plane for stable
  baseline evaluation.
