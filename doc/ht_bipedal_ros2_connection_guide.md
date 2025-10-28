# HighTorque bipedal robot ROS2 connect guide

## Requirements

* Robot IP
* ROS2 environment

## Intro

HighTorque Bipedal robots supports ROS1 development. To simplify control, we need to bridge the communication to ROS2. To bridge the connection, we run a ros1_bridge container on the robot. This allows us to discover the robot as a ros_bridge node and directly send /cmd_vel commands to the robot to control it's movements.

## Steps

If the robot has been configured, skip to Robot Start & Control step.

### Docker Installation

This guide walks you through installing and configuring Docker on any Raspberry Pi-like ARM board running Debian-based Linux (Raspberry Pi OS, Ubuntu, Armbian, etc.).

---

#### 1. Check prerequisites

Make sure your system is 64-bit:

```bash
uname -m
```

If it prints `aarch64`, great — that’s 64-bit.  
If it says `armv7l`, you’re on 32-bit — Docker still works, but use lightweight images.

Update and upgrade packages:

```bash
sudo apt update && sudo apt upgrade -y
```

---

#### 2. Install Docker (official script)

Run the official Docker installation script:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

This installs `docker-ce`, `docker-cli`, and `containerd` and enables the Docker service.

---

#### 3. Add user to Docker group

Allow your user to run Docker without `sudo`:

```bash
sudo usermod -aG docker $USER
```

Then **log out and back in** (or reboot).  
Test Docker:

```bash
docker run hello-world
```

#### 4. Enable Docker at boot

```bash
sudo systemctl enable docker
sudo systemctl start docker
sudo systemctl status docker
```

You should see “active (running)” status.

#### 5. Verify installation

```bash
docker version
docker info
```

You should see something like:

```
Server: Docker Engine - Community
 Architecture: aarch64
 OS/Arch: linux/arm64
```

**Next:** You can now deploy containers like `ros1_bridge` for ROS integration.

### Docker Launch Command

When docker is installed properly, you can launch a ros1_bridge by running this docker command. This pulls a pre-built ros2 foxy build for running ros1_bridge.

```docker
docker run -d \
  --name ros1_bridge \
  --restart unless-stopped \
  --network host \
  -e ROS_DOMAIN_ID=0 \
  -e ROS_NAMESPACE=robot1 \
  -e ROS_MASTER_URI=http://127.0.0.1:11311 \
  ros:foxy-ros1-bridge \
  bash -c "sleep 15 && ros2 run ros1_bridge dynamic_bridge"
```

Explanation: 
"docker run -d": Launch the container in detached mode.

"--name ros1_bridge": Create a name called "ros1_bridge". You can use this name to start/stop the container

"--restart unless-stopped": Restart the docker on boot or restart. In theory you don't need to launch this docker everytime. It starts automatically after startup.

"--network host": Use the host network.

"-e ROS_DOMAIN_ID=0": ROS2 environment variable. By default the domain ID is 0. Only ROS2 nodes under the same ROS_DOMAIN_ID can discover each other.

"-e ROS_NAMESPACE=robot1": ROS2 namespace. In a multi-agent setting, you should set different namespace for each robot, so that you can control which robot receives the message.

"-e ROS_MASTER_URI=http://127.0.0.1:11311": ROS1 master URI. Since the robot runs ROS1, you need to set this variable to the host IP to talk to the ROS1 master.

"ros:foxy-ros1-bridge": Name of the pre-built image.

"bash -c "sleep 15 && ros2 run ros1_bridge dynamic_bridge"": the bash command launched inside the docker. The command first sleep for 15s, waiting for the host to start up ROS1 dependencies and launch the corresponding topics. Then, the docker runs a dynamic bridge between ROS1 and ROS2. 


After these steps, you should be able to see a /ros_bridge node if you run "ros2 topic list" on a different machine under the same domain ID.

### Robot Start \& Control

The robot can be controlled by sending /cmd_vel messages to the robot when it is in walking mode (press RT for standing up and Y for stepping).