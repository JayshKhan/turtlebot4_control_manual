# TurtleBot4 SLAM & Navigation Guide (ROS 2 Jazzy)

**Goal:** To map an unknown environment using TurtleBot4 SLAM and then use that map for autonomous point-to-point navigation.

**Assumptions:**

1.  You have a TurtleBot4 (Standard or Lite).
2.  You have a separate User PC (running Ubuntu 24.04 recommended for Jazzy compatibility).
3.  Both the TurtleBot4's Raspberry Pi and your User PC have **ROS 2 Jazzy Jalisco** installed. (Refer to official ROS 2 installation guides if needed).
4.  Both devices are connected to the **same Wi-Fi network**.
5.  You can `ssh` into the TurtleBot4's Raspberry Pi.
6.  You are using a standard TurtleBot4 OS image where core nodes might auto-start via systemd services.

**Terminology:**

* **Pi:** The Raspberry Pi 4 computer onboard the TurtleBot4.
* **User PC:** Your separate laptop or desktop computer.

---

## Phase 1: Prerequisites and Initial Setup

*(Steps 1-4 can often be skipped if previously configured, but are essential checks)*

### 1. Network Configuration:

* Ensure both the Pi and User PC are on the same Wi-Fi network.
* Find the IP address of the Pi (e.g., `ip a` on Pi, or check router). Let's assume it's `PI_IP`.
* Find the IP address of your User PC (`ip a`). Let's assume it's `USER_PC_IP`.
* **On User PC:** Verify connectivity:
    ```bash
    ping PI_IP
    ```
* **On Pi (via SSH):** Verify connectivity:
    ```bash
    ping USER_PC_IP
    ```
* *Troubleshooting:* If pings fail, check Wi-Fi connection, IP addresses, and firewall settings on both machines.

### 2. SSH Access:

* **On User PC:** Access the Pi's terminal:
    ```bash
    ssh ubuntu@PI_IP
    ```
    (Default password is often `turtlebot`. Change it for security!) You will run Pi-specific commands in this SSH session.

### 3. ROS 2 Environment Setup (Domain ID & Sourcing):

* ROS 2 nodes on different machines need the same `ROS_DOMAIN_ID` to communicate. Choose an ID (e.g., 30).
* **On Pi (via SSH):**
    ```bash
    # Check current ID (optional)
    echo $ROS_DOMAIN_ID
    # Set and save ID for future sessions (replace 30 if needed)
    echo "export ROS_DOMAIN_ID=30" >> ~/.bashrc
    source ~/.bashrc
    # Verify ROS 2 sourcing (should happen automatically, but check)
    printenv ROS_DISTRO # Should output 'jazzy'
    ```
* **On User PC:**
    ```bash
    # Check current ID (optional)
    echo $ROS_DOMAIN_ID
    # Set and save ID for future sessions (use the SAME ID as the Pi)
    echo "export ROS_DOMAIN_ID=30" >> ~/.bashrc
    source ~/.bashrc
    # Source ROS 2 Jazzy environment
    source /opt/ros/jazzy/setup.bash
    # Add sourcing to .bashrc for convenience (if not already there)
    echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc
    ```
* *Troubleshooting:* If nodes can't see each other later (e.g., topics missing), incorrect or mismatched `ROS_DOMAIN_ID` is a common cause. Restart terminals after changing `.bashrc`.

### 4. Time Synchronization (Crucial):

* ROS 2 relies heavily on synchronized system clocks, especially for TF transforms. `chrony` is typically used.
* **On Pi (via SSH):** Check status (often pre-configured):
    ```bash
    chronyc sources
    ```
    (Look for an entry starting with `^*` indicating sync).
* **On User PC:** Install and check:
    ```bash
    sudo apt update
    sudo apt install chrony -y
    chronyc sources
    ```
* *Troubleshooting:* If clocks are not synchronized (`LookupTransform Error`, `ExtrapolationException` in TF), ensure `chrony` is installed, running, and configured correctly on both machines to use a reliable time source.

---

## Phase 2: Package Verification/Installation

### 5. Install/Verify Necessary Packages:

* You need core TB4 packages, SLAM tools, Navigation tools, and visualization tools.
* **On Pi (via SSH):** Verify or install:
    ```bash
    sudo apt update
    sudo apt install -y \
      ros-jazzy-slam-toolbox \
      ros-jazzy-turtlebot4-navigation \
      ros-jazzy-turtlebot4-bringup \
      ros-jazzy-turtlebot4-description \
      ros-jazzy-turtlebot4-msgs \
      ros-jazzy-teleop-twist-keyboard # Good for testing base movement
    ```
* **On User PC:** Install visualization and control tools:
    ```bash
    sudo apt update
    sudo apt install -y \
      ros-jazzy-turtlebot4-desktop \
      ros-jazzy-teleop-twist-keyboard \
      ros-jazzy-slam-toolbox # Install slam-toolbox here too for map saving client if needed
    ```
    *Note:* `turtlebot4-desktop` usually pulls in `rviz2`, `nav2-rviz-plugins`, `turtlebot4-viz` etc.

---

## Phase 3: Running SLAM (Mapping the Environment)

### 6. Check Core TurtleBot4 Nodes (Auto-Start Service):

* Standard TurtleBot4 images often start the core nodes automatically on boot using a `systemd` service (`turtlebot4-lite.service` or `turtlebot4-standard.service`).
* **On Pi (SSH Terminal 1):** Check if the service is active:
    ```bash
    # For TB4 Lite:
    systemctl is-active turtlebot4-lite.service
    # OR For TB4 Standard:
    # systemctl is-active turtlebot4-standard.service
    ```
* If the output is `active`, the core nodes are running. Proceed to Step 7.
* If the output is `inactive` or `failed`, try starting it:
    ```bash
    # For TB4 Lite:
    sudo systemctl start turtlebot4-lite.service
    # OR For TB4 Standard:
    # sudo systemctl start turtlebot4-standard.service
    ```
    Then check `is-active` again. View logs with `sudo journalctl -u <service_name> -f` if issues persist.
* **Only if NOT using the service / Service fails:** You would manually launch the bringup in this terminal (as described in the previous guide version): `ros2 launch turtlebot4_bringup lite.launch.py` (or `standard.launch.py`).

### 7. Start SLAM Node:

* **On Pi (SSH Terminal 2):** Launch the SLAM node. Using `sync:=false` (asynchronous mode) is often recommended for performance on the Pi.
    ```bash
    ros2 launch turtlebot4_navigation slam.launch.py sync:=false
    ```
* This node listens for topics like `/scan` and `/odom` (or `/tf`) published by the core nodes (from Step 6) and starts publishing the `/map`.

### 8. Start Visualization (RViz):

* **On User PC (Terminal 1):** Launch RViz with a configuration suitable for SLAM.
    ```bash
    ros2 launch turtlebot4_navigation rviz.launch.py
    ```
* In RViz:
    * Ensure the **"Fixed Frame"** (under Global Options) is set to `map`.
    * Add the `Map` display (Topic: `/map`). You should see the map start to appear.
    * Add the `RobotModel` display.
    * Add the `LaserScan` display (Topic: `/scan`).
    * (Optional) Add the `TF` display.
* *Troubleshooting:* If RViz shows errors like "Fixed Frame [map] does not exist", wait a bit for SLAM (Step 7) to initialize. If topics are missing, double-check `ROS_DOMAIN_ID`, network connectivity, and ensure the core bringup service (Step 6) and SLAM node (Step 7) are running without errors.

### 9. Teleoperate Robot to Build Map:

* **On User PC (Terminal 2):** Launch the keyboard teleoperation node.
    ```bash
    ros2 run teleop_twist_keyboard teleop_twist_keyboard
    ```
* Click on this terminal window and use the keys (usually `i`, `,`, `j`, `l`, `k`) to **slowly and carefully** drive the robot around the entire area you want to map.
* **Tips for Good Mapping:**
    * Move slowly and turn gently.
    * Cover the entire area from different angles.
    * Make "loop closures" (return to previously mapped areas).
    * Avoid bumps and shakes.
    * Watch the map build in RViz.

### 10. Save the Map:

* Once satisfied with the map in RViz:
* **On Pi (SSH Terminal 3):** Use the `slam_toolbox` service to save the map. Choose a name and path.
    ```bash
    # Example: Save map named 'my_lab_map' in ~/maps directory
    mkdir -p ~/maps # Create directory if it doesn't exist
    ros2 service call /slam_toolbox/save_map slam_toolbox/srv/SaveMap "{name: {data: '${HOME}/maps/my_lab_map'}}"
    ```
* This creates `~/maps/my_lab_map.yaml` and `~/maps/my_lab_map.pgm`.
* *Confirmation:* Service call should return `success: true`.
* **Save the map *before* stopping the SLAM node.**

### 11. Shutdown SLAM Process:

* Press `Ctrl+C` in the terminals running:
    * SLAM launch (`slam.launch.py` - Pi Terminal 2)
    * Teleop (`teleop_twist_keyboard` - User PC Terminal 2)
    * RViz (`rviz.launch.py` - User PC Terminal 1)
* If you manually started the core bringup (rare case from Step 6), stop it too (`Ctrl+C`). If using the service, it can remain running or be stopped (`sudo systemctl stop <service_name>`).

---

## Phase 4: Autonomous Navigation using the Saved Map

### 12. Ensure Core TurtleBot4 Nodes are Running:

* **On Pi (SSH Terminal 1):** Verify the core service is active, same as in Step 6.
    ```bash
    # For TB4 Lite:
    systemctl is-active turtlebot4-lite.service
    # OR For TB4 Standard:
    # systemctl is-active turtlebot4-standard.service

    # If inactive, start it:
    # sudo systemctl start <service_name>
    ```
* Ensure it's `active`.

### 13. Start Navigation Stack:

* **On Pi (SSH Terminal 2):** Launch the Nav2 stack, providing the **full, absolute path** to your saved map file (`.yaml`).
    ```bash
    # Use the EXACT path where you saved your map .yaml file
    ros2 launch turtlebot4_navigation nav_bringup.launch.py map:=${HOME}/maps/my_lab_map.yaml
    ```
* This launches Nav2 components (AMCL, planners, controllers) using your map.

### 14. Start Visualization (RViz):

* **On User PC (Terminal 1):** Launch RViz using the same navigation configuration.
    ```bash
    ros2 launch turtlebot4_navigation rviz.launch.py
    ```
* In RViz:
    * You should see your saved map (`Map` display).
    * You should see the laser scan (`LaserScan` display).
    * You should see the robot model (`RobotModel` display).
    * You should see Nav2 elements like `Costmaps`, `Planned Path`, `Particle Cloud` (AMCL localization).

### 15. Initialize Robot Pose (Localization):

* **In RViz (On User PC):** Tell Nav2 where the robot is on the map.
    * Click the **"2D Pose Estimate"** button in the RViz toolbar.
    * Click on the map approximately where the robot is located, and drag the arrow to indicate its orientation. Release.
    * Observe the `Particle Cloud` in RViz. It should converge around the robot's location. A tight cluster indicates good localization.
    * *Crucial:* Navigation will fail without a good initial pose estimate.

### 16. Send Navigation Goal:

* **In RViz (On User PC):**
    * Click the **"Nav2 Goal"** button in the RViz toolbar.
    * Click on the map where you want the robot to go, and drag the arrow for the desired final orientation. Release.
    * Observe: A path should appear, and the robot should start moving autonomously.

### 17. Shutdown Navigation:

* Press `Ctrl+C` in the terminals running:
    * Navigation launch (`nav_bringup.launch.py` - Pi Terminal 2)
    * RViz (`rviz.launch.py` - User PC Terminal 1)
* The core service (`turtlebot4-*.service`) can be left running or stopped using `sudo systemctl stop <service_name>` on the Pi.

---

## Common Troubleshooting and Tips

* **"Fixed Frame [map] does not exist":** SLAM/Nav2 node hasn't started properly or isn't publishing transforms. Check terminals for errors. Check `ROS_DOMAIN_ID`. Ensure core nodes (Step 6/12) are active.
* **"LookupTransform Error" / TF Errors:** Often time sync issues (`chrony`), network latency, or a node crash. Check `chronyc sources`. Check all nodes are running. Use `ros2 run tf2_tools view_frames.py` to visualize TF tree.
* **Poor Map Quality:** Drive slower during mapping, ensure good features for LiDAR, try loop closures. Avoid large, empty areas or glass walls. `slam_toolbox` parameters can be tuned (advanced).
* **Poor Localization (AMCL):** Robot gets lost. Ensure map accurately reflects the environment. Provide a good initial pose estimate (Step 15). Remap if environment changed significantly.
* **Robot Doesn't Move/Stops:** Check Nav2 costmaps in RViz (is goal/path blocked?). Check terminal output for Nav2 errors. Is initial pose correct? Ensure core nodes are publishing `/odom` and `/scan`.
* **Robot Hits Obstacles:** Check LiDAR scan in RViz - does it see the obstacle? Tune Nav2 costmap parameters (inflation radius, etc.). Clean sensors.
* **Performance Issues (Pi):** Close unnecessary programs. Use `htop` on Pi. `sync:=false` for SLAM helps.
* **Check Topics:** Use `ros2 topic list`, `ros2 topic echo <topic_name>` (e.g., `/scan`, `/odom`, `/map`, `/tf`, `/cmd_vel`, `/goal_pose`, `/particle_cloud`) to debug data flow.
* **Service Management:** Use `sudo systemctl status/stop/start/restart turtlebot4-lite.service` (or `standard`) to manage the core node service on the Pi. Use `sudo journalctl -u <service_name> -f` to view live logs for the service.

---

This guide provides a detailed path for TurtleBot4 SLAM and Navigation with ROS 2 Jazzy in Markdown format. Remember that robotics often involves troubleshooting, so observe carefully and check outputs when issues arise. Good luck!
