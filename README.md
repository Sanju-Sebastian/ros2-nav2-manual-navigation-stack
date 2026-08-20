# ROS2 Nav2: Manual Navigation Stack (No nav2_bringup)

A ROS2 Humble package implementing a full Nav2 navigation stack for a differential-drive robot in Gazebo, with every stage — map loading, AMCL localization, and navigation (planner/controller/behavior tree) — wired manually through individual launch and parameter files instead of the standard `nav2_bringup` package.

## Why manual wiring

`nav2_bringup` is the standard entry point for most Nav2 projects — it bundles map loading, localization, and navigation into one pre-built launch file. Building each piece manually instead required understanding what `nav2_bringup` actually does under the hood: lifecycle node management (configure → activate transitions), parameter loading order, inter-node dependencies, and how the plugin registry resolves custom planner/controller/behavior classes. This repository is that manual implementation.

## What's implemented

- **Map loading** — `map_server` + dedicated lifecycle manager, serving a static occupancy grid map
- **Localization** — AMCL with tuned parameters, verified under both static pose seeding and live motion (teleop-driven divergence and re-convergence, tracked via `/amcl_pose` covariance)
- **Navigation** — `nav2_planner` (NavFn global planner), `nav2_controller` (DWB local planner), `nav2_behaviors` (spin/backup/wait recovery), and `nav2_bt_navigator`, each configured and activated independently, verified with real navigation goals

## Repository Structure

testbed_navigation/
├── launch/
│ ├── map_loader.launch.py # map_server + lifecycle manager
│ ├── localization.launch.py # AMCL + lifecycle manager
│ └── navigation.launch.py # planner, controller, behavior_server, bt_navigator + lifecycle manager
├── config/
│ ├── amcl_params.yaml
│ └── nav2_params.yaml
└── CMakeLists.txt / package.xml


## Environment

- ROS2 Humble
- Ubuntu 22.04
- Gazebo classic 11.10.2
- Cyclone DDS (`RMW_IMPLEMENTATION=rmw_cyclonedds_cpp`) — Fast DDS is known to cause Nav2 timeouts and dropped sensor data in this configuration

## How to Run

Each stage runs in its own terminal, brought up in order:

```bash
# Terminal 1 — simulation
ros2 launch testbed_bringup testbed_full_bringup.launch.py

# Terminal 2 — map server
ros2 launch testbed_navigation map_loader.launch.py

# Terminal 3 — AMCL localization
ros2 launch testbed_navigation localization.launch.py

# Terminal 4 — navigation stack
ros2 launch testbed_navigation navigation.launch.py
```

Send a navigation goal:
```bash
ros2 topic pub /goal_pose geometry_msgs/msg/PoseStamped "{header: {frame_id: 'map'}, pose: {position: {x: <X>, y: <Y>, z: 0.0}, orientation: {w: 1.0}}}" --once
```

## Key Technical Findings

- Robot base frame is `base_footprint`, not `base_link` — confirmed from the URDF and Gazebo diff_drive plugin rather than assumed, since getting this wrong causes AMCL to silently fail.
- The robot's Gazebo spawn position and the map's coordinate origin are offset from each other (two different reference frames) — AMCL's initial pose had to be set accounting for this offset, not left at the default.
- The robot's diff_drive plugin specifies wheel geometry but not velocity/acceleration limits or footprint radius — these were set as conservative estimates for a robot of this size rather than treated as verified spec values, and are flagged as such rather than presented as exact.

## Debugging Highlights

- **Missing install directive**: the starter package's `CMakeLists.txt` installed its `launch` directory but not its `maps` directory, so the map YAML was present at runtime but the referenced image file was not — a build/install pipeline issue, not a config bug.
- **Lifecycle vs. process management**: AMCL got permanently stuck on "Waiting for map…." after `map_server` was stopped in an earlier session — the fix was adopting strict process discipline (each long-running node in its own terminal, never interrupted), not a configuration change.
- **Plugin registry syntax**: initially declared custom planner/behavior plugins using C++ namespace syntax (`nav2_navfn_planner::NavfnPlanner`) instead of the plugin registry's expected declared-type syntax (`nav2_navfn_planner/NavfnPlanner`) — a systematic error repeated across every custom plugin in the first draft, not an isolated typo.
- **Bounds-testing the planner**: deliberately sent a goal outside the map's coverage area to confirm failure behavior — the planner correctly rejected it and `bt_navigator` ran its full recovery sequence (spin → wait → backup) before reporting failure via the action server, rather than hanging indefinitely.

## Verification

Each stage was verified independently with both log output and live topic data, not just "it launched without errors":

- **Map loading**: confirmed via server logs and `ros2 topic echo /map --once` returning real occupancy grid data.
- **AMCL**: verified under static pose seeding (near-zero covariance) and under live motion — a teleop-driven convergence test showed covariance spike to 5–16 during erratic driving, then drop to 0.06–0.6 over ~10 readings after stopping, proving AMCL tracks real LiDAR data rather than echoing a seeded pose.
- **Navigation**: verified by sending real goals and confirming both the controller's own success report and independent `/amcl_pose` data matched the goal within tolerance.

## Demo Video

[Watch the demo video](https://drive.google.com/file/d/1MkvR9lv6bxnWsgc2ZRh_BzFInNQP_Zfs/view?usp=sharing) — map loading, AMCL localization, and two successful navigation goals across the map.

## Technical Stack

ROS2 Humble · Gazebo classic · Nav2 (map_server, AMCL, NavFn, DWB, BT Navigator) · Python

## Contact

Sanju N Sebastian
[linkedin.com/in/sanju-n-sebastian](https://linkedin.com/in/sanju-n-sebastian) · sanju.n.sebastian@gmail.com
