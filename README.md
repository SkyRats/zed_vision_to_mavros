# zed_vision_to_mavros

This repository contains code for using Zed camera's VIO for position estimation on ardupilot. This code was tested with a ZED 2i, ROS 2 humble and mavros.

## Ardupilot parameters
- EK3_SRC1_POSXY = ExternalNav
- EK3_SRC1_POSZ = ExternalNav (could use baro, but not recommended)
- EK3_SRC1_VELXY = ExternalNav
- EK3_SRC1_VELZ = ExternalNav
- EK3_SRC1_YAW = ExternalNav (could use compass)
- GPS1_TYPE = NONE
---

## Installation

Make sure you have ROS2 (humble or later) and MAVROS installed.

Clone this repo into your ROS2 workspace:

```bash
cd ~/ros2_ws/src
git clone https://github.com/hfcaio/zed_vision_to_mavros.git
cd ..
colcon build
source install/setup.bash
```

## How to run

### ROS 2 with mavros
In other to run you need to change the name of apm.launch to apm.launch.xml, after that to run the code is pretty simple. Jus run:

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
```

```bash
ros2 launch zed_vision_to_mavros zed_all_nodes.launch.xml
```
## TODO:
 - argument input to zed_camera position node
 - use zed pose_with_covarience
 - enable zed use with multiple rotations
 - enable transformation from base_link to zed_camera_center
