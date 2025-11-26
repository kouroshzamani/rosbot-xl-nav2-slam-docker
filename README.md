# rosbot-xl-nav2-slam-docker
End-to-end SLAM + Navigation2 pipeline for ROSbot XL simulation running fully inside Docker (tested on Ubuntu 24.04 host), based on Husarion’s tutorials and extended with fixes for reproducible navigation.

What you get

✅ Gazebo simulation (ROSbot XL)

✅ RViz2 visualization (TF, LaserScan, Map, Costmaps)

✅ slam_toolbox online async SLAM (live mapping)

✅ Nav2 bringup (planner/controller/bt_navigator + costmaps)

✅ Navigation goals from RViz or CLI (/navigate_to_pose)

✅ Known issues fixed: frame does not exist, duplicate SLAM, cmd_vel mismatch, RViz Nav panel missing

Credits / References

Husarion ROS 2 Tutorials (Navigation & Exploration)

Tutorial 9: Navigation

Tutorial 10: Exploration

Please consider ⭐️ starring Husarion’s resources and this repo if it helps.

Repository layout (suggested)

├── launch/

 navigation.launch.py              # Nav2 bringup wrapper (slam:=false by default)
 slam.launch.sh                    # convenience launcher
 bringup_all.md                    # runbook / cheat-sheet

├── config/

 nav2_params.yaml                  # Nav2 parameters (working baseline)
 rviz_nav2.rviz            # RViz config with Navigation2 panel enabled

├── scripts/

 host_xhost.sh                      # xhost +local:docker
 enter_container.sh                 # docker exec -it ...
static_tf_basefootprint.sh         # base_link -> base_footprint
 cmd_vel_relay.sh                   # /cmd_vel_nav -> /cmd_vel
 diagnostics.sh                     # ros2 node/topic/tf checks

├── README/


Your actual folders may differ. README focuses on commands and concepts you can match to your structure.

Requirements
Host machine

Ubuntu 24.04 (tested), Docker Engine + docker compose

X11 display (for Gazebo GUI / RViz GUI)

Container image

husarion/rosbot-xl-gazebo:humble (or your derived image)

ROS 2 Humble already included

Quick start (TL;DR)
0) Allow Docker to use your display (host)

Gazebo GUI requires X11 access:

xhost +local:docker


If Gazebo fails with qt.qpa.xcb: could not connect to display then you forgot this step.

1) Start the container (host)
docker start demo-rosbot_xl-1
docker exec -it demo-rosbot_xl-1 bash


Inside the container, always source:

source /opt/ros/humble/setup.bash
source /ros2_ws/install/setup.bash

Full runbook (recommended: 5 terminals)

Keeping each subsystem in its own terminal makes debugging easy and prevents accidental double launches.

Terminal A — Gazebo simulation (container)
docker exec -it demo-rosbot_xl-1 bash
source /opt/ros/humble/setup.bash
source /ros2_ws/install/setup.bash

ros2 launch rosbot_xl_gazebo simulation.launch.py


Note: If you restart the container and Gazebo GUI doesn’t appear, check logs:

docker logs --tail 60 demo-rosbot_xl-1

Terminal B — Static TF fix (container)

This fixes the common frame does not exist errors when components expect base_footprint.

docker exec -it demo-rosbot_xl-1 bash
source /opt/ros/humble/setup.bash
source /ros2_ws/install/setup.bash

ros2 run tf2_ros static_transform_publisher 0 0 0 0 0 0 base_link base_footprint


✅ Keep this running.

Terminal C — SLAM (slam_toolbox online async) (container)
docker exec -it demo-rosbot_xl-1 bash
source /opt/ros/humble/setup.bash
source /ros2_ws/install/setup.bash

ros2 launch slam_toolbox online_async_launch.py


Expected to see:

solver initialization

sensor registration

no TF “invalid frame” spam

Terminal D — Nav2 bringup (container)

Important: we run Nav2 with SLAM disabled here because we already launch slam_toolbox separately.

docker exec -it demo-rosbot_xl-1 bash
source /opt/ros/humble/setup.bash
source /ros2_ws/install/setup.bash

ros2 launch rosbot_xl_navigation navigation.launch.py slam:=false


✅ Verify lifecycle:

ros2 lifecycle get /controller_server
ros2 lifecycle get /planner_server
ros2 lifecycle get /bt_navigator


Expected:

active [3] for all

Terminal E — cmd_vel pipeline fix (container, if needed)

In our setup, Nav2 outputs /cmd_vel_nav while the base controller listens on /cmd_vel.
Bridge them with a relay:

docker exec -it demo-rosbot_xl-1 bash
source /opt/ros/humble/setup.bash
source /ros2_ws/install/setup.bash

ros2 run topic_tools relay /cmd_vel_nav /cmd_vel


✅ Keep running.

To confirm publishers/subscribers:

ros2 topic info /cmd_vel_nav -v
ros2 topic info /cmd_vel -v
ros2 node info /rosbot_xl_base_controller

RViz2 setup (container)
docker exec -it demo-rosbot_xl-1 bash
source /opt/ros/humble/setup.bash
source /ros2_ws/install/setup.bash

rviz2

RViz checklist

Fixed Frame: map

Add displays:

TF

LaserScan topic: /scan

Map topic: /map

Costmap (Local/Global) as needed

RobotModel if available

Nav2 Goal UI tip (the “why nothing happens” fix)

RViz has multiple “goal tools”, but the one that drives Nav2 properly depends on the panel:

Open: Panels → Add New Panel → Navigation2

In the top toolbar select Nav2 Goal (not “2D Goal Pose”)

✅ After enabling the Navigation2 panel, “Nav2 Goal” works as expected.

Sending goals (CLI)

This is the most reliable test because it bypasses RViz UI confusion:

ros2 action send_goal /navigate_to_pose nav2_msgs/action/NavigateToPose \
"{pose: {header: {frame_id: map}, pose: {position: {x: 1.0, y: 0.0, z: 0.0}, orientation: {w: 1.0}}}}"


Expected:

“Goal accepted”

status: SUCCEEDED

Common issues & fixes
1) frame does not exist in RViz / SLAM / Nav2

Cause: something expects base_footprint but TF only provides base_link.
Fix: run the static TF publisher:

ros2 run tf2_ros static_transform_publisher 0 0 0 0 0 0 base_link base_footprint

2) “Goal was rejected by server”

Most common cause in our case: duplicate SLAM publishing map -> odom twice.
Fix: ensure only one SLAM source exists:

Either run slam_toolbox yourself and slam:=false for Nav2

Or let Nav2 bring it up and do not launch slam_toolbox separately

3) Robot does not move although goals are accepted

Cause: Nav2 output is on /cmd_vel_nav but the base controller listens to /cmd_vel.
Fix: relay it:

ros2 run topic_tools relay /cmd_vel_nav /cmd_vel

4) Gazebo GUI crashes / “could not connect to display”

Cause: X11 not allowed for Docker.
Fix on host:

xhost +local:docker

Diagnostics (quick commands)
TF graph
ros2 run tf2_tools view_frames

Check cmd topics
ros2 topic list | grep cmd_vel
ros2 topic echo /cmd_vel_nav
ros2 topic echo /cmd_vel

Check Nav2 nodes
ros2 node list | grep -E "controller|planner|bt_navigator|costmap|lifecycle"

Check actions
ros2 action list | grep navigate
ros2 action info /navigate_to_pose

Notes about goal arrow direction (RViz)

When setting a goal in RViz, the arrow indicates the desired final orientation.
So it’s normal that the robot may rotate near the end to match the arrow direction.

Next steps / roadmap

Frontier-based exploration (autonomous mapping)

Saving/loading maps (map_server)

Tuning costmaps and controllers for smoother trajectories

(Optional) running everything via a single combined launch file
