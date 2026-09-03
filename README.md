# Mobile Robot ROS 2 Workspace

ROS 2 workspace for a differential-drive mobile robot. The repository combines a robot model and Gazebo Sim world with three packages:

* `robot_description` — URDF model, meshes, RViz configurations, Gazebo Sim world, ROS–Gazebo bridges, and simulation sensor nodes.
* `robot_estimation` — IMU filtering and bias correction, complementary orientation estimation, differential-drive command conversion, motor-command splitting, and wheel-odometry processing.
* `robot_nav` — HTTP ingestion for phone IMU/magnetometer data, filtering and bias correction, and complementary pose/path estimation for RViz.

This project supports development and teaching of mobile-robot modeling, simulation, sensor processing, and state estimation. It is not a production localization stack: IMU-only position integration drifts and should be fused with wheel odometry, visual odometry, LiDAR, or GNSS for long-duration navigation.

## Requirements

* Ubuntu with ROS 2 (Humble or a compatible distribution)
* `colcon`, `rosdep`, and a C++17-capable compiler
* Gazebo Sim / `ros_gz_sim` and `ros_gz_bridge`
* RViz2

Use the ROS distribution name installed on your machine wherever `<ros_distro>` appears below.

## Installation and build

```bash
source /opt/ros/<ros_distro>/setup.bash
mkdir -p ~/robotic_ws
cd ~/robotic_ws
git clone https://github.com/Amir-sut82/robotic_ws.git
sudo apt update
sudo apt install -y python3-rosdep
# Run these two commands once per machine if rosdep is not initialized:
sudo rosdep init
rosdep update
rosdep install --from-paths src --ignore-src --rosdistro <ros_distro> -r -y
colcon build --symlink-install
source install/setup.bash
```

For every new terminal:

```bash
source /opt/ros/<ros_distro>/setup.bash
source ~/robotic_ws/install/setup.bash
```

## Run the simulation

Launch Gazebo Sim, spawn the robot, start the ROS–Gazebo bridge, publish TF, and open the prepared RViz layout:

```bash
ros2 launch robot_description gazebo.launch.py
```

Drive the robot with velocity commands:

```bash
ros2 topic pub --rate 10 /cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.2}, angular: {z: 0.0}}"
```

Useful simulation topics include `/wheel_encoder/odom`, `/joint_states`, `/zed/zed_node/imu/data_raw`, `/gz_lidar/scan`, `/gz_lidar/points`, and the bridged camera topics.

## Inspect the model without Gazebo

```bash
ros2 launch robot_description display.launch.py
```

The launch file opens RViz and the joint-state publisher GUI. A different URDF or RViz configuration can be supplied explicitly:

```bash
ros2 launch robot_description display.launch.py \
  model:=/absolute/path/to/model.urdf \
  rvizconfig:=/absolute/path/to/config.rviz \
  gui:=False
```

## Run the estimation pipeline

```bash
ros2 launch robot_estimation estimator.launch.py
```

The default data flow is:

```text
/zed/zed_node/imu/data_raw -> low_pass_filter_node -> /imu/filtered
/imu/filtered -> bias_correct_imu_node -> /imu/bias_corrected
/imu/bias_corrected -> complementary_yaw_node -> /estimation/orientation
/cmd_vel -> diff_drive_controller_node -> /motor_commands
/motor_commands -> motor_command_node -> /left_motor_rpm, /right_motor_rpm
/wheel_encoder/odom -> motion_model_odom_node -> /motion_model/odom (+ TF)
```

Parameters can be overridden, for example:

```bash
ros2 run robot_estimation low_pass_filter_node --ros-args \
  -p imu_topic_in:=/my_imu -p imu_topic_out:=/my_filtered_imu
```

## Phone IMU navigation pipeline

`robot_nav` exposes an HTTP server that accepts `POST /data` JSON payloads and publishes `sensor_msgs/msg/Imu` and `sensor_msgs/msg/MagneticField` messages. Start it with:

```bash
ros2 launch robot_nav nav.launch.py
```

The default server listens on `0.0.0.0:8000`; send data to `http://<robot-ip>:8000/data`. Sensor objects named `gyroscope`, `accelerometer` (or `totalacceleration`), and `magnetometer` are recognized. Magnetometer values are interpreted as microtesla and converted to tesla for ROS.

The navigation nodes publish filtered and bias-corrected IMU data, `/estimation/orientation`, and `/estimation/path`.

## Package layout

```text
robotic_ws/
├── README.md
└── src/
    ├── robot_description/     # model, meshes, worlds, bridges, launch files
    ├── robot_estimation/      # filtering, estimation, control, odometry
    └── robot_nav/             # phone ingestion and pose estimation
```

## Troubleshooting

* Source both ROS and the workspace overlay in every new terminal.
* If Gazebo cannot find a mesh or world, run the Gazebo launch from a sourced ROS environment and inspect `GZ_SIM_RESOURCE_PATH`.
* If the phone cannot reach the server, verify the robot IP, port 8000, and that both devices share a network.
* IMU-only `/estimation/path` will drift by design; use sensor fusion for bounded navigation.

## License

Apache License 2.0. See [`src/LICENSE`](src/LICENSE).

