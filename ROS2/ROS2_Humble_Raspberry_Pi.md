# Installing ROS2 on Raspberry Pi 4B

This document will walk you through installing ROS2 on a Raspberry Pi model 4B running Ubuntu 22.04.5 (64-bit) desktop version. ROS2 (Robot Operating System 2) is an open-source middleware framework widely used in robotics development. It provides tools, libraries, and conventions for building robot applications, and is the standard platform for both hobbyist and professional robotics work. You will need an active internet connection (wifi or ethernet) throughout this process.

> **NOTE:** The Raspberry Pi 4B uses ARM64 (aarch64) architecture, NOT amd64 (x86_64). Always use `arch=arm64` when adding the ROS2 repository. Using `amd64` is a common mistake that causes a "not signed" or "Not Found" error, as the Pi is an ARM platform, and AMD64 is used on x86_64 architecture devices (standard desktop/laptop PCs). If you have used Linux on a regular PC before, be aware that some familiar steps differ here because of this architectural difference.

## Contents

- [Stop apt Lock Issues](#stop-apt-lock-issues)
- [Set Locales](#set-locales)
- [Enable Universe Repository](#enable-universe-repository)
- [Add the ROS2 GPG Key and Repository](#add-the-ros2-gpg-key-and-repository)
- [Update Package Lists](#update-package-lists)
- [Install ROS2](#install-ros2)
- [Install Development Tools](#install-development-tools)
- [Configure Environment](#configure-environment)
- [Verify Installation](#verify-installation)
- [Create a ROS2 Workspace (optional)](#create-a-ros2-workspace-optional)
- [Raspberry Pi 4B Tips](#raspberry-pi-4b-tips)

<br>

## Stop apt Lock Issues

Ubuntu uses a package manager called `apt` to install and manage software. To prevent two processes from modifying packages at the same time and corrupting the system, `apt` uses lock files, small files that signal "I'm busy, don't touch this." Ubuntu ships with a background service called `unattended-upgr` that automatically downloads and installs security updates. This service runs silently in the background and frequently holds the `apt` lock, which will block our installation commands if it happens to be running at the same time.

If you run into the error `Waiting for cache lock: Could not get lock /var/lib/dpkg/lock-frontend`, it means `unattended-upgr` (or another process) currently holds the lock. Stop it gracefully first:

```bash
sudo systemctl stop unattended-upgr  # this command may freeze if unattended-upgr is mid-run.
```

If it hangs for more than 30 seconds, the service is likely in the middle of an operation and won't stop cleanly. Cancel with `Ctrl+C` and force-kill it instead:

```bash
sudo pkill -9 unattended-upgr
sudo lsof /var/lib/dpkg/lock-frontend
```

The `lsof` command lists which process (if any) still has the lock file open. If a process is listed, wait for it to exit or kill it. If the lock file is stale, meaning the process that created it has already died but the file was never cleaned up, remove it manually. This is safe to do when no `apt` processes are running:

```bash
sudo rm /var/lib/dpkg/lock-frontend
sudo rm /var/lib/dpkg/lock
sudo rm /var/lib/apt/lists/lock
sudo dpkg --configure -a
```

The final command, `dpkg --configure -a`, tells the package manager to finish configuring any packages that were left in a half-installed state, which can happen if a previous `apt` run was interrupted.

Since `unattended-upgr` will keep coming back and potentially interrupt later steps, it is safest to remove it entirely for now. You can reinstall it after ROS2 is set up if you want automatic updates to resume:

```bash
sudo apt-get purge unattended-upgr -y
```

<br>

## Set Locales

A locale defines the language, character encoding, and regional formatting conventions that the operating system uses. ROS2 requires a UTF-8 locale to be set correctly — without it, some tools may fail to parse text or display errors about unsupported characters. This step ensures the system is configured to use standard US English with UTF-8 encoding, which is the expected environment for ROS2:

```bash
sudo apt update && sudo apt install locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8
locale  # verify
```

The final `locale` command prints your current locale settings so you can confirm everything looks correct before moving on.

<br>

## Enable Universe Repository

Ubuntu organizes its software into several repositories. The default installation only enables the `main` repository, which contains software officially supported by Canonical (Ubuntu's publisher). The `universe` repository contains community-maintained open-source software — including several packages that ROS2 depends on. Without enabling it, some dependencies will be unavailable and the ROS2 installation will fail:

```bash
sudo apt install software-properties-common
sudo add-apt-repository universe
```

`software-properties-common` is a utility that provides the `add-apt-repository` command. If it is not already installed, this step installs it first.

<br>

## Add the ROS2 GPG Key and Repository

`apt` needs to know where to download ROS2 packages from, and it needs a way to verify that those packages are authentic and haven't been tampered with. This is handled through two things: a GPG key (a cryptographic signature used to verify package authenticity) and a repository entry (a URL pointing to where the packages live).

First, install `curl` if it is not already present, then download the ROS2 GPG key:

```bash
sudo apt update && sudo apt install curl -y
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg
```

Before proceeding, verify the key downloaded correctly. It should be identified as `"OpenPGP Public Key"` or `"data"` — if it says `"ASCII text"`, the download likely failed and returned an error page instead of the actual key file:

```bash
file /usr/share/keyrings/ros-archive-keyring.gpg
```

If it says `"ASCII text"` or `"Not Found"`, the download failed. Delete it and re-run the `curl` command above. Now add the repository entry, which tells `apt` where to find ROS2 packages. Note `arch=arm64` — this is required for Raspberry Pi and must not be changed to `amd64`:

```bash
echo "deb [arch=arm64 signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] \
  http://packages.ros.org/ros2/ubuntu jammy main" | \
  sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
```

This writes a new file to `/etc/apt/sources.list.d/`, which is the standard location for third-party repository definitions on Ubuntu.

<br>

## Update Package Lists

Now that the ROS2 repository has been added, run `apt update` to download the latest package lists from all configured repositories, including the one just added. This is what makes ROS2 packages discoverable for installation:

```bash
sudo apt update
sudo apt upgrade  # optional. Do not upgrade the Ubuntu distro as this may break things.
```

The `apt upgrade` step is optional but generally good practice as it updates any existing packages to their latest versions, which can resolve dependency conflicts. However, be careful not to perform a full distribution upgrade (`do-release-upgrade`), as upgrading Ubuntu to a newer version (e.g., from 22.04 to 24.04) can break ROS2 Humble, which is built specifically for Ubuntu 22.04 (Jammy).

<br>

## Install ROS2

This is the main installation step. `ros-humble-desktop` is the full ROS2 Humble release, which includes the core framework, communication libraries, visualization tools (like RViz), and a suite of demo packages. "Humble" is the name of this particular ROS2 release, and it is the version that targets Ubuntu 22.04:

```bash
sudo apt install ros-humble-desktop
sudo apt install ros-humble-ros-base  # this is a lightweight alternative.
```

If you are working in a resource-constrained environment or don't need the visualization tools, `ros-humble-ros-base` is a lighter alternative that includes only the core communication infrastructure without the GUI tools.

> **NOTE:** This is a large download. May take 10–20 minutes on the Pi.

<br>

## Install Development Tools

These tools are needed to build and manage ROS2 packages from source — which you will almost certainly need to do at some point, even if your immediate goal is just to run existing packages.

`colcon` is ROS2's build tool, used to compile packages in a workspace. `ros-dev-tools` is a meta-package that installs a collection of commonly needed development utilities, including `rosdep` (a dependency resolver), `rosidl` tools (for generating message interfaces), and others:

```bash
sudo apt install python3-colcon-common-extensions ros-dev-tools
```

<br>

## Configure Environment

ROS2 installs its files to `/opt/ros/humble/`, but the shell doesn't know to look there for commands by default. The `setup.bash` script sets up all the necessary environment variables — like `PATH`, `AMENT_PREFIX_PATH`, and `PYTHONPATH` — so that ROS2 commands and libraries are accessible from any terminal.

By appending the `source` command to `~/.bashrc`, this configuration is applied automatically every time a new terminal session is opened, so you don't have to run it manually each time:

```bash
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

The second line applies the changes immediately to your current terminal session, since `~/.bashrc` is normally only read when a new session starts.

<br>

## Verify Installation

To confirm that ROS2 is working correctly, run a quick communication test using the built-in demo nodes. ROS2 uses a publish/subscribe model where nodes communicate by sending messages over named topics. In this test, the `talker` node publishes messages and the `listener` node subscribes to them. If they can talk to each other, the core ROS2 communication layer is functioning correctly.

Open two separate terminal windows and run one command in each:

```bash
# Terminal 1
ros2 run demo_nodes_cpp talker
```

```bash
# Terminal 2
ros2 run demo_nodes_py listener
```

If the listener prints messages from the talker, ROS2 is installed correctly.

<br>

## Create a ROS2 Workspace (optional)

A ROS2 workspace is a directory structure where you develop, build, and install your own ROS2 packages. This is where your custom robot code will live. It is separate from the system-wide ROS2 installation so that your code doesn't interfere with the base install, and multiple workspaces can coexist on the same machine.

`colcon build` compiles everything in the `src/` directory. The workspace is currently empty, so this will produce no output, but it creates the `install/`, `build/`, and `log/` directories and confirms that `colcon` is working:

```bash
mkdir -p ~/ros_ws/src
cd ~/ros_ws
colcon build
echo "source ~/ros_ws/install/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

Sourcing `~/ros_ws/install/setup.bash` makes your workspace's packages visible to ROS2, layered on top of the base installation.

<br>

## Raspberry Pi 4B Tips

These recommendations address common issues that arise specifically from running ROS2 on the Pi's hardware, rather than a desktop machine.

- **Use the official Raspberry Pi 5V/3A power supply.** `colcon` builds are CPU-intensive and can cause undervoltage on weaker supplies. Undervoltage causes the Pi to throttle its CPU speed, which dramatically slows down builds and can cause unpredictable crashes.
- **Add a heatsink or active cooling fan.** CPU temperatures spike during builds. Without cooling, the Pi will thermally throttle, automatically reducing clock speed to protect the hardware — which again slows everything down significantly.
- **For navigation or high-throughput topics, consider Cyclone DDS over the default Fast DDS.** DDS (Data Distribution Service) is the underlying communication protocol ROS2 uses to pass messages between nodes. The default implementation, Fast DDS, can have performance and reliability issues on constrained hardware. Cyclone DDS is a leaner alternative that many users find more stable on the Pi:

```bash
sudo apt install ros-humble-rmw-cyclonedds-cpp
echo "export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp" >> ~/.bashrc
```

- **Use a 32GB or larger microSD card (or a USB SSD).** The full desktop install uses significant storage, and robotics development — with datasets, logs, and built workspaces — adds up quickly. A USB SSD will also provide considerably faster read/write speeds than a microSD card, which noticeably improves build times.
