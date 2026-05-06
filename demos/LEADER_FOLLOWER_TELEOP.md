# Leader-Follower Franka Teleop Demo

This document explains how to start the current `franka_feedback_teleop` demo
for leader-follower Franka teleoperation.

The recommended path is to use the Docker/devcontainer environment from the
`franka_feedback_teleop` workspace.

## What This Demo Starts

The launch file:

```bash
ros2 launch franka_feedback_teleop basic_teleop.launch.py
```

starts three parts of the teleop stack:

- `franka_ros2_teleop`: leader-follower arm teleoperation.
- `franka_gripper`: Franka gripper control.
- `franka_gripper_teleop`: the custom gripper command path used by this demo.

This document covers running the teleop demo on hardware. It does not cover
data recording. Some stale recorder references may still exist in package
metadata or tests, but they are not part of this demo workflow.

## Hardware Preflight

Before launching anything that can move robot hardware, confirm:

- The leader and follower Franka robots are powered on and in a safe state.
- Both robots are unlocked with FCI activated
- The robot workspaces are clear of people, tools, loose cables, and objects
  that should not be contacted.
- The robot network is connected and the host/container can reach the robots.
- The gripper serial device is connected on the host as `/dev/ttyUSB0`.

Basic host-side checks:

```bash
ls /dev/tty* | grep USB0
ping 192.168.1.11
ping 192.168.1.15
```

## Start The Docker Environment

From the `franka_feedback_teleop` workspace root on the Beelink NUC:

```bash
cd /home/beelink/ros2_ws/franka_feedback_teleop
git submodule update --init --recursive
docker compose up -d
```

Attach to the running container with VS Code

```bash
Ctrl + Shift + P -> Dev Containers: Attach to Running Container
```

The container uses host networking and passes through `/dev/ttyUSB0`. If the
container was started through the devcontainer flow, `.env` should be generated
with the host user and serial group IDs. If you start Compose manually and
serial permissions fail, see the troubleshooting section below.

## Build The Workspace

Source the workspace in every new terminal before using `ros2 launch`:

```bash
cd /ros2_ws
source install/setup.bash
```

## Run The Demo

With the hardware preflight complete and the workspace sourced, launch:

```bash
ros2 launch franka_feedback_teleop basic_teleop.launch.py
```


## Useful Launch Arguments

The wrapper launch file exposes these arguments:

- `robot_config_file`: path to the leader-follower teleop robot config file.
  By default, this uses `franka_ros2_teleop/config/fr3_teleop_config.yaml`.
- `gripper_robot_ip`: hostname or IP address for the gripper robot. The default
  is `192.168.1.11`.
- `gripper_use_fake_hardware`: whether to use fake gripper hardware. The
  hardware demo should leave this as `false`.
- `gripper_robot_type`: robot type used to derive Franka gripper joint names.
  The default is `fr3`.
- `gripper_namespace`: namespace passed to the Franka gripper launch.
- `gripper_teleop_parameters_file`: parameter YAML for the gripper teleop node.
  By default, this uses `franka_gripper_teleop/config/gripper_teleop.yaml`.
- `gripper_teleop_namespace`: namespace for the custom gripper teleop node.

Example with multiple overrides:

```bash
ros2 launch franka_feedback_teleop basic_teleop.launch.py \
  gripper_robot_ip:=192.168.1.11 \
  gripper_robot_type:=fr3
```

## During The Demo

Use a deliberate startup sequence:

1. Confirm the hardware preflight checklist.
2. Start the Docker environment.
3. Build and source the workspace.
4. Launch `basic_teleop.launch.py`.
5. Watch the terminal output for package, network, controller, or gripper
   connection errors before moving the robots.
6. Begin teleoperation only after the stack is fully started and the workspace
   is clear.

To stop the demo, press `Ctrl+C` in the launch terminal and wait for ROS 2 to
shut down the included launch files. Do not close the terminal abruptly unless
the normal shutdown path is not responding.

## Troubleshooting

### `/dev/ttyUSB0` Is Missing

Check that the gripper serial device is plugged into the host and appears as
the expected device:

```bash
ls -l /dev/ttyUSB0
```

If it appears as a different device, update the Docker Compose device mapping
or reconnect the device according to lab practice.

### Serial Permissions Or `SERIAL_GID` Problems

The devcontainer initialization command writes `.env` with `USER_UID`,
`USER_GID`, and `SERIAL_GID`. If you start Docker Compose manually and
`SERIAL_GID` is empty or wrong, regenerate it from the host:

```bash
cd /home/dmalexa5/ros2_ws/franka_feedback_teleop
printf 'USER_UID=%s\nUSER_GID=%s\nSERIAL_GID=%s\n' \
  "$(id -u "$USER")" \
  "$(id -g "$USER")" \
  "$(stat -c '%g' /dev/ttyUSB0)" > .env
docker compose up --build
```

Inside the container, check that the device exists and the current process has
the expected supplemental group:

```bash
ls -l /dev/ttyUSB0
id
```

### ROS Packages Cannot Be Found

If `ros2 launch` cannot find `franka_feedback_teleop`,
`franka_ros2_teleop`, `franka_gripper`, or `franka_gripper_teleop`, the
workspace is usually missing submodules, has not been built, or has not been
sourced in the current shell.

Run inside the workspace/container:

```bash
cd /ros2_ws
git submodule update --init --recursive
colcon build
source install/setup.bash
```

### Recorder References Appear In The Package

The `franka_feedback_teleop` package previously included recorder
functionality. If you see stale references to recorder nodes, recorder tests,
or Parquet output in package metadata, ignore them for this demo. The run path
documented here is the launch-only teleop workflow.

## Manual Verification

After updating this document or the demo workspace, recommended manual checks
are:

```bash
cd /home/dmalexa5/ros2_ws/franka_feedback_teleop
docker compose up --build
docker compose exec franka_feedback_teleop bash
```

Inside the container:

```bash
cd /ros2_ws
colcon build
source install/setup.bash
ros2 launch franka_feedback_teleop basic_teleop.launch.py
```

Only run the final launch command on hardware after the preflight checklist is
complete.
