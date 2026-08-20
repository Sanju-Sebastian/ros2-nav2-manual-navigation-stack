# testbed_navigation

ROS2 Nav2 package for the Testbed-T1.0.0 robot, built for the ERIC Robotics technical assignment.

## Assignment Constraint

The standard approach to Nav2 uses `nav2_bringup`, which handles map loading, localization, and navigation through a single pre-built launch file. This assignment explicitly disallows that approach. Instead, `map_server`, `amcl`, and the navigation plugins (`nav2_planner`, `nav2_controller`, `nav2_bt_navigator`, `nav2_behaviors`) are wired manually through custom launch files and parameter files, each paired with its own `nav2_lifecycle_manager` instance. This required understanding what `nav2_bringup` actually does under the hood — lifecycle node management, parameter loading order, and inter-node dependencies — rather than treating it as a black box.

## Environment

- ROS2 Humble
- Ubuntu 22.04
- Gazebo classic 11.10.2
- Cyclone DDS (`RMW_IMPLEMENTATION=rmw_cyclonedds_cpp`) set as default middleware, based on a warning in the assignment's own `help.md` FAQ that Fast DDS can cause Nav2 timeouts and dropped sensor data.

## Package Structure

testbed_navigation/
- launch/
  - map_loader.launch.py — map_server + lifecycle manager
  - localization.launch.py — AMCL + lifecycle manager
  - navigation.launch.py — planner, controller, behavior_server, bt_navigator + lifecycle manager
- config/
  - amcl_params.yaml
  - nav2_params.yaml
- CMakeLists.txt / package.xml

## How to Run

Requires the full stack to be brought up in this order, each in its own terminal, none closed once started (closing a node's terminal stops that node and silently breaks anything downstream of it):

Terminal 1 — simulation:
`ros2 launch testbed_bringup testbed_full_bringup.launch.py`

Terminal 2 — map server:
`ros2 launch testbed_navigation map_loader.launch.py`

Terminal 3 — AMCL localization:
`ros2 launch testbed_navigation localization.launch.py`

Terminal 4 — navigation stack:
`ros2 launch testbed_navigation navigation.launch.py`

Send a navigation goal:
`ros2 topic pub /goal_pose geometry_msgs/msg/PoseStamped "{header: {frame_id: 'map'}, pose: {position: {x: <X>, y: <Y>, z: 0.0}, orientation: {w: 1.0}}}" --once`

Note: the map covers roughly x: -10.2 to 10.05, y: -9.94 to 10.06 (derived from the map origin and resolution) — goals outside this range will correctly fail rather than silently doing nothing (see Challenges below).

## Key Technical Findings

- The robot's base frame is `base_footprint`, not `base_link` — confirmed from the URDF and Gazebo diff_drive plugin, not assumed.
- The robot spawns in Gazebo at (0, 5, 0) while the map's origin is (-10.2, -9.94, 0) — two different coordinate references. AMCL's initial pose was set to (10.2, 14.94) to correctly account for this offset.
- The robot's diff_drive Gazebo plugin only specifies wheel separation (0.35m) and diameter (0.1m) — it does not specify velocity limits, acceleration limits, or a footprint radius anywhere in the URDF or plugin config. The values used in nav2_params.yaml for these (max_vel_x: 0.4 m/s, acc_lim_x: 1.0 m/s², robot_radius: 0.25m, etc.) are conservative estimates for a robot of this size, not values derived from the provided spec. These would need real-world or more detailed simulation tuning before being trusted for anything beyond this test environment.

## Challenges & Debugging Log

1. map_server "bad file" error. The starter testbed_bringup package's CMakeLists.txt only installed its launch directory, not its maps directory, so the map YAML was installed but the referenced .pgm image was not. Fixed by adding maps to the install directive and rebuilding.

2. AMCL stuck on "Waiting for map....". Root cause was a process-lifecycle mistake, not a config bug: map_server had been stopped in a prior session, so AMCL had no map to subscribe to. Fixed by adopting a strict rule — every long-running node gets its own terminal and is never interrupted once started.

3. Nav2 plugin naming syntax errors. Both the planner (nav2_navfn_planner::NavfnPlanner) and all three behavior plugins (nav2_behaviors::Spin/BackUp/Wait) were initially declared using C++ namespace syntax (::) instead of the plugin registry's expected declared-type syntax (/). This caused the planner and behavior servers to fail at configuration with a clear error listing the correct declared types, which made the fix straightforward — nav2_navfn_planner/NavfnPlanner and nav2_behaviors/Spin, etc. Worth noting this was a systematic error (same syntax mistake repeated across every custom plugin declaration in the first draft), not an isolated typo.

4. Goal sent outside map bounds. During navigation testing, a goal was deliberately sent outside the map's actual coverage area to observe failure behavior. The planner correctly rejected it (worldToMap failed, "goal is off the global costmap"), and bt_navigator correctly ran its recovery behavior sequence (spin → wait → backup → spin) before reporting Goal failed via the action server rather than hanging indefinitely. This was a useful negative test — it confirmed the recovery-behavior logic in the behavior tree is functioning correctly, not just the happy path.

## Verification

Each stage was verified independently before moving to the next, using both log output and live topic data — not just "it launched without errors":

- Map loading: confirmed via launch logs (405 X 400 map @ 0.05 m/cell) and `ros2 topic echo /map --once` returning real occupancy grid data.
- AMCL: confirmed three ways — (1) static pose seed test with near-zero covariance, (2) live node check confirming map_server was actually running, (3) a teleop-driven convergence test: covariance spiked to 5–16 during erratic manual driving, then dropped steadily to 0.06–0.6 over ~10 readings after stopping, proving AMCL tracks real LiDAR data rather than just echoing a seeded pose.
- Navigation: confirmed by sending a real goal within map bounds and observing both controller_server's own success report (Reached the goal!, Goal succeeded) and independently checking /amcl_pose — final position matched the goal within the configured 0.25m tolerance.

## Not Yet Explored / Possible Extensions

- Real velocity/acceleration tuning based on the robot's actual physical performance rather than conservative estimates.
- Dynamic obstacle avoidance testing (all testing so far used a static map with no moving obstacles introduced).
- RViz-based interactive goal setting (all goals in this testing were sent via CLI for reproducibility).
