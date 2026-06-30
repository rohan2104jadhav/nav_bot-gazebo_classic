# nav_bot — Gazebo Classic Differential-Drive Navigation Robot

A ROS 2 Humble differential-drive ground robot simulated in **Gazebo Classic**, with LiDAR and camera sensors, `ros2_control`-based drive, joystick teleop, `twist_mux` command arbitration, and SLAM Toolbox mapping support.

## Overview

`nav_bot` spawns a custom differential-drive robot (chassis + two driven wheels + caster) into a Gazebo Classic world, publishes its full TF tree via `robot_state_publisher`, and drives it through a `ros2_control` `diff_drive` controller. Sensor data (2D LiDAR, RGB camera, optional depth camera) is published natively by Gazebo Classic's ROS 2 plugins — no `ros_gz_bridge` is needed, unlike Gazebo Sim (Ignition) setups.

```
Joystick ──► joy_node ──► teleop_twist_joy ──► /cmd_vel_joy ─┐
                                                              ├─► twist_mux ──► /diff_cont/cmd_vel_unstamped
Keyboard / Nav2 ─────────────────────────────► /cmd_vel ─────┘
                                                              │
                                                              ▼
                                                   ros2_control diff_drive
                                                              │
                                              ┌───────────────┴───────────────┐
                                              ▼                               ▼
                                        Gazebo Classic                  /odom, TF
                                   (LiDAR /scan, camera /image)
```

## Features

- Custom URDF/Xacro robot description (chassis, wheels, caster, inertials)
- 2D LiDAR (360°, 12 m range, `/scan`) via `libgazebo_ros_ray_sensor.so`
- RGB camera (`camera.xacro`) and optional RGB-D depth camera (`depth_camera.xacro`)
- `ros2_control` differential-drive controller (`diff_cont`) + `joint_broad` joint state broadcaster
- Joystick teleop (`joy` + `teleop_twist_joy`) merged with other command sources via `twist_mux`
- SLAM Toolbox config (`mapper_params_online_async.yaml`) for online async mapping
- Two Gazebo Classic worlds: `empty.world` and `obstacles.world`
- Saved example map (`my_map_save.pgm` / `.yaml`) and SLAM Toolbox serialized pose graph

## Prerequisites

- Ubuntu 22.04 + ROS 2 Humble
- Gazebo Classic (`gazebo11`)

```bash
sudo apt install ros-humble-gazebo-ros-pkgs ros-humble-gazebo-ros2-control \
  ros-humble-xacro ros-humble-robot-state-publisher ros-humble-joint-state-publisher \
  ros-humble-ros2-control ros-humble-ros2-controllers ros-humble-controller-manager \
  ros-humble-joy ros-humble-teleop-twist-joy ros-humble-twist-mux \
  ros-humble-slam-toolbox ros-humble-rviz2
```

## Build

```bash
mkdir -p ~/nav_bot_ws/src
cd ~/nav_bot_ws
git clone https://github.com/rohan2104jadhav/nav_bot-gazebo_classic.git src/nav_bot
colcon build --packages-select nav_bot
source install/setup.bash
```

## Usage

### Launch the full simulation

```bash
ros2 launch nav_bot launch_sim.launch.py
```

This starts:
- Gazebo Classic (server + client) with `worlds/obstacles.world` by default
- `robot_state_publisher` (via `rsp.launch.py`, with `ros2_control` enabled)
- The robot spawned into Gazebo via `spawn_entity.py`
- RViz with the `drive_bot.rviz` display configuration
- `joint_broad` and `diff_cont` controller spawners
- Joystick teleop (`joy_node` + `teleop_twist_joy`)
- `twist_mux`, arbitrating between `/cmd_vel` and `/cmd_vel_joy` and publishing to `/diff_cont/cmd_vel_unstamped`

Useful launch arguments:

```bash
ros2 launch nav_bot launch_sim.launch.py world:=<path/to/world.world> use_sim_time:=true rvizconfig:=<path/to/config.rviz>
```

### Robot description only (no Gazebo)

```bash
ros2 launch nav_bot rsp.launch.py use_sim_time:=false use_ros2_control:=true
```

### Manual teleop

```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard --ros-args -r /cmd_vel:=/cmd_vel_keyboard
```

(Add any additional source to `config/twist_mux.yaml` to have it arbitrated alongside joystick input.)

### SLAM mapping

```bash
ros2 launch slam_toolbox online_async_launch.py params_file:=src/nav_bot/config/mapper_params_online_async.yaml use_sim_time:=true
```

Drive the robot around (joystick or keyboard teleop) while SLAM Toolbox builds the map, then save it:

```bash
ros2 run nav2_map_server map_saver_cli -f my_map_save
```

An example saved map (`my_map_save.pgm`, `my_map_save.yaml`) and SLAM Toolbox serialized pose graph (`my_map_serial.data`, `my_map_serial.posegraph`) are included at the repository root for reference.

## Key Topics

| Topic | Type | Description |
|---|---|---|
| `/robot_description` | `std_msgs/String` | Latched URDF XML |
| `/cmd_vel`, `/cmd_vel_joy` | `geometry_msgs/Twist` | Velocity command sources merged by `twist_mux` |
| `/diff_cont/cmd_vel_unstamped` | `geometry_msgs/Twist` | Arbitrated output driving the diff-drive controller |
| `/odom` | `nav_msgs/Odometry` | Wheel odometry from `ros2_control` diff-drive |
| `/scan` | `sensor_msgs/LaserScan` | 2D LiDAR scan |
| `/tf`, `/tf_static` | `tf2_msgs/TFMessage` | `odom → base_link → wheels/sensors` tree |

## Repository Structure

```
src/nav_bot/
├── description/             # URDF/Xacro: chassis, sensors, ros2_control, Gazebo plugins
│   ├── robot.urdf.xacro       # entry point
│   ├── robot_core.xacro       # chassis, wheels, joints
│   ├── lidar.xacro            # 2D LiDAR sensor + Gazebo ray plugin
│   ├── camera.xacro           # RGB camera + Gazebo camera plugin
│   ├── depth_camera.xacro     # RGB-D camera variant
│   ├── ros2_control.xacro     # ros2_control hardware interface
│   ├── gazebo_control.xacro   # Gazebo ros2_control plugin binding
│   └── inertial_macros.xacro  # reusable inertia macros
├── launch/
│   ├── launch_sim.launch.py   # full simulation bringup
│   ├── rsp.launch.py          # robot_state_publisher only
│   └── joystick.launch.py     # joy + teleop_twist_joy
├── config/
│   ├── my_controllers.yaml         # ros2_control controller manager config
│   ├── joystick.yaml               # joy/teleop_twist_joy parameters
│   ├── twist_mux.yaml              # twist_mux input priorities
│   ├── mapper_params_online_async.yaml  # SLAM Toolbox parameters
│   ├── drive_bot.rviz / config.rviz     # RViz display configs
├── worlds/
│   ├── empty.world
│   └── obstacles.world
└── package.xml / CMakeLists.txt
```

## Notes

- This project was generated from the popular `articubot`/ROS 2 robot-package template — `package.xml` still has placeholder maintainer/license fields worth filling in.
- Gazebo Classic handles ROS 2 topic publishing natively through its plugins (`libgazebo_ros_ray_sensor.so`, `libgazebo_ros_camera.so`, `gazebo_ros2_control`), so no `ros_gz_bridge` is required — this differs from a Gazebo Sim (Ignition) setup.
- `build/`, `install/`, and `log/` are `colcon` build artifacts and should be excluded from version control if not already.

## License

See `LICENSE.md`.