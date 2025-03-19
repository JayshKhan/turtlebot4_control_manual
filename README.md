# TurtleBot 4 Setup and Basic Operation

This document outlines the steps to set up and operate the TurtleBot 4, based on the official TurtleBot documentation, with some specific adjustments for network configuration.

:Note: this is simple first commit

## Initial Setup

1. **Basic Setup:** Follow the initial setup instructions from the official TurtleBot 4 user manual: [https://turtlebot.github.io/turtlebot4-user-manual/setup/basic.html](https://turtlebot.github.io/turtlebot4-user-manual/setup/basic.html)

    *   **Important Network Configuration:**  Instead of relying on automatic IP address publishing, you will manually connect the Raspberry Pi, your laptop, and the TurtleBot 4 to the same network.  This usually involves connecting all devices to the same Wi-Fi network or using a dedicated router.

    *   Once connected, the IP address of the Raspberry Pi will be displayed on the TurtleBot 4's screen.  Use this IP address to connect via SSH:

        ```bash
        ssh ubuntu@IP_ADDRESS
        ```

        The default password is `turtlebot4`.

2. **Wi-Fi Connection:** Access the TurtleBot 4's web interface by navigating to `IP_ADDRESS:8080` in your web browser.  Enter the required Wi-Fi details to connect the robot to your network.  This process may take some time.  Refer to the simple discovery documentation for more details: [https://turtlebot.github.io/turtlebot4-user-manual/setup/simple_discovery.html](https://turtlebot.github.io/turtlebot4-user-manual/setup/simple_discovery.html)

3. **Teleoperation:** After connecting to the network and waiting a short while, check the ROS topics on your laptop to ensure they are being published.  You can use tools like `ros2 topic list` and `ros2 topic echo <topic_name>` to verify. Once topics are being published, you can proceed with the teleoperation tutorial: [https://turtlebot.github.io/turtlebot4-user-manual/tutorials/driving.html](https://turtlebot.github.io/turtlebot4-user-manual/tutorials/driving.html)

    *   **ROS_DOMAIN_ID Check:** Before starting the teleoperation, it is crucial to verify the `ROS_DOMAIN_ID`.  Run the following command on your laptop:

        ```bash
        echo $ROS_DOMAIN_ID
        ```

        If the output is not `0`, set it to `0` using the following command:

        ```bash
        export ROS_DOMAIN_ID=0
        ```

        This ensures that your laptop and the TurtleBot 4 are communicating on the same ROS domain.  You may want to add this export command to your `.bashrc` or `.zshrc` file to make it persistent across terminal sessions.
