# Installing ROS2 on Jetson Orin Nano Super (JetPack 7.2)

This document will walk you through installing ROS2 Jazzy Jazzalisco on an NVIDIA Jetson Orin Nano Super Developer Kit running JetPack 7.2 (Jetson Linux 39.2, Ubuntu 24.04, 64-bit). ROS2 (Robot Operating System 2) is an open-source middleware framework widely used in robotics development. It provides tools, libraries, and conventions for building robot applications, and is the standard platform for both hobbyist and professional robotics work. You will need an active internet connection (wifi or ethernet) throughout this process.

This guide assumes JetPack 7.2 is already flashed and the Jetson has completed its first-boot setup. It does not cover flashing.

> **NOTE:** The Jetson Orin Nano uses ARM64 (aarch64) architecture, NOT amd64 (x86_64). The ROS2 apt package handles this automatically, but if you ever add the repository by hand, always use `arch=arm64`. Using `amd64` is a common mistake that causes a "not signed" or "Not Found" error, as the Jetson is an ARM platform, and AMD64 is used on x86_64 devices (standard desktop/laptop PCs). Additionally, JetPack 7.2 is based on Ubuntu 24.04 (Noble), so the matching ROS2 release is **Jazzy**. Do not follow guides for ROS2 Humble on this device. Humble targets Ubuntu 22.04 (Jammy) and its packages will not install on JetPack 7.2. Older Jetson guides (JetPack 5.x and 6.x, Ubuntu 20.04 and 22.04) do not apply here.

## Contents

- [Verify Your Jetson](#verify-your-jetson)
- [Stop apt Lock Issues](#stop-apt-lock-issues)
- [Set Locales](#set-locales)
- [Enable Universe Repository](#enable-universe-repository)
- [Add the ROS2 Repository](#add-the-ros2-repository)
- [Update Package Lists](#update-package-lists)
- [Install ROS2](#install-ros2)
- [Install Development Tools](#install-development-tools)
- [Configure Environment](#configure-environment)
- [Verify Installation](#verify-installation)
- [Create a ROS2 Workspace (optional)](#create-a-ros2-workspace-optional)
- [Jetson Orin Nano Super Tips](#jetson-orin-nano-super-tips)

<br>

## Verify Your Jetson

Before installing anything, confirm that the device is running what you expect. ROS2 Jazzy needs Ubuntu 24.04 on a 64-bit ARM processor, and it is much easier to find a mismatch now than after a failed install:

```bash
cat /etc/os-release   # should show Ubuntu 24.04 (VERSION_CODENAME=noble)
uname -m              # should print aarch64
dpkg-query --show nvidia-l4t-core   # shows the installed Jetson Linux (L4T) version
```

The first command confirms the Ubuntu release, the second confirms the CPU architecture, and the third prints the version of NVIDIA's board support package, which should correspond to Jetson Linux 39.2 on JetPack 7.2.

If `VERSION_CODENAME` is `jammy` or `focal`, you are on an older JetPack and Jazzy is not the correct ROS2 release for your system.

<br>

## Stop apt Lock Issues

Ubuntu uses a package manager called `apt` to install and manage software. To prevent two processes from modifying packages at the same time and corrupting the system, `apt` uses lock files, small files that signal "*I'm busy, don't touch this*." Ubuntu ships with a background service called `unattended-upgrades` that automatically downloads and installs security updates. This service runs silently in the background and frequently holds the `apt` lock, which will block our installation commands if it happens to be running at the same time. This is especially common right after first boot.

If you run into the error `Waiting for cache lock: Could not get lock /var/lib/dpkg/lock-frontend`, it means `unattended-upgrades` (or another process) currently holds the lock. The simplest fix is to wait a few minutes and try again. If it persists, stop the service gracefully first:

```bash
sudo systemctl stop unattended-upgrades  # this command may freeze if unattended-upgrades is mid-run.
```

If it hangs for more than 30 seconds, the service is likely in the middle of an operation and won't stop cleanly. Cancel with `Ctrl+C` and force-kill it instead:

```bash
sudo pkill -9 unattended-upgrades
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

If `unattended-upgrades` keeps interrupting later steps, you can temporarily disable it rather than removing it. Unlike a purge, this keeps it easy to turn back on once ROS2 is set up:

```bash
sudo systemctl disable --now unattended-upgrades
# To turn it back on later:
# sudo systemctl enable --now unattended-upgrades
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

Ubuntu organizes its software into several repositories. The default installation only enables the `main` repository, which contains software officially supported by Canonical (Ubuntu's publisher). The `universe` repository contains community-maintained open-source software, including several packages that ROS2 depends on. Without enabling it, some dependencies will be unavailable and the ROS2 installation will fail:

```bash
sudo apt install software-properties-common
sudo add-apt-repository universe
```

`software-properties-common` is a utility that provides the `add-apt-repository` command. If it is not already installed, this step installs it first.

<br>

## Add the ROS2 Repository

`apt` needs to know where to download ROS2 packages from, and it needs a way to verify that those packages are authentic and haven't been tampered with. This is handled through two things: a GPG key (a cryptographic signature used to verify package authenticity) and a repository entry (a URL pointing to where the packages live).

For Jazzy, the ROS project provides both in a single small package called `ros2-apt-source`. Installing it sets up the key and the repository entry for you, and keeps them up to date automatically, so you do not need to download the key or write the repository file by hand.

First, install `curl` if it is not already present:

```bash
sudo apt update && sudo apt install curl -y
```

Next, look up the latest version of the `ros2-apt-source` package and download the `.deb` file that matches your Ubuntu release (`noble` on JetPack 7.2):

```bash
export ROS_APT_SOURCE_VERSION=$(curl -s https://api.github.com/repos/ros-infrastructure/ros-apt-source/releases/latest | grep -F "tag_name" | awk -F\" '{print $4}')
curl -L -o /tmp/ros2-apt-source.deb "https://github.com/ros-infrastructure/ros-apt-source/releases/download/${ROS_APT_SOURCE_VERSION}/ros2-apt-source_${ROS_APT_SOURCE_VERSION}.$(. /etc/os-release && echo ${UBUNTU_CODENAME:-${VERSION_CODENAME}})_all.deb"
```

Before proceeding, verify that the download worked. The file should be identified as a `"Debian binary package"`. If it says `"ASCII text"` or `"JSON data"`, the download likely failed and returned an error page instead of the package (this can happen if GitHub's API rate-limits you):

```bash
file /tmp/ros2-apt-source.deb
```

If it failed, wait a minute and re-run the two commands above. Once the file looks correct, install it:

```bash
sudo dpkg -i /tmp/ros2-apt-source.deb
```

This writes a new file to `/etc/apt/sources.list.d/`, which is the standard location for third-party repository definitions on Ubuntu, and installs the ROS2 signing key to `/usr/share/keyrings/`. You can confirm that the repository was added correctly:

```bash
ls /etc/apt/sources.list.d/ | grep -i ros
```

<br>

## Update Package Lists

Now that the ROS2 repository has been added, run `apt update` to download the latest package lists from all configured repositories, including the one just added. This is what makes ROS2 packages discoverable for installation:

```bash
sudo apt update
sudo apt upgrade  # optional, but read the warning below first.
```

The `apt upgrade` step is optional but generally good practice, as it updates any existing packages to their latest versions, which can resolve dependency conflicts. On a Jetson, however, be cautious. The system includes NVIDIA-specific packages (the `nvidia-l4t-*` family, CUDA, and the kernel) that must stay consistent with each other. Read the list of packages that `apt` proposes to upgrade before confirming. If you see a large batch of `nvidia-l4t-*` or kernel packages and you are not sure you want them, cancel and skip the upgrade.

Also be careful not to perform a full distribution upgrade (`do-release-upgrade`). Upgrading Ubuntu to a newer version will break both JetPack and ROS2 Jazzy, which is built specifically for Ubuntu 24.04 (Noble).

<br>

## Install ROS2

This is the main installation step. `ros-jazzy-desktop` is the full ROS2 Jazzy release, which includes the core framework, communication libraries, visualization tools (like RViz), and a suite of demo packages. "Jazzy" is the name of this particular ROS2 release, and it is the version that targets Ubuntu 24.04:

```bash
sudo apt install ros-jazzy-desktop
```

If you are running the Jetson headless (no monitor, controlled over SSH), or you don't need the visualization tools, install `ros-jazzy-ros-base` instead. It includes only the core communication infrastructure without the GUI tools, and it is a much smaller download:

```bash
sudo apt install ros-jazzy-ros-base
```

You only need one of the two. `desktop` already includes everything in `ros-base`.

> **NOTE:** This is a large download. `ros-jazzy-desktop` may take 10–15 minutes depending on your connection speed and storage.

<br>

## Install Development Tools

These tools are needed to build and manage ROS2 packages from source — which you will almost certainly need to do at some point, even if your immediate goal is just to run existing packages.

`ros-dev-tools` is a meta-package that installs a collection of commonly needed development utilities, including `colcon` (ROS2's build tool, used to compile packages in a workspace), `rosdep` (a dependency resolver), `vcstool` (for managing multiple source repositories), and others:

```bash
sudo apt install ros-dev-tools
```

`rosdep` needs a one-time setup before it can look up dependencies. `rosdep init` creates its configuration file, and `rosdep update` downloads the dependency database for your user:

```bash
sudo rosdep init
rosdep update
```

If `rosdep init` reports that the file already exists, this is fine and you can continue.

<br>

## Configure Environment

ROS2 installs its files to `/opt/ros/jazzy/`, but the shell doesn't know to look there for commands by default. The `setup.bash` script sets up all the necessary environment variables — like `PATH`, `AMENT_PREFIX_PATH`, and `PYTHONPATH` — so that ROS2 commands and libraries are accessible from any terminal.

By appending the `source` command to `~/.bashrc`, this configuration is applied automatically every time a new terminal session is opened, so you don't have to run it manually each time:

```bash
echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

The second line applies the changes immediately to your current terminal session, since `~/.bashrc` is normally only read when a new session starts.

<br>

## Verify Installation

To confirm that ROS2 is working correctly, run a quick communication test using the built-in demo nodes. ROS2 uses a publish/subscribe model where nodes communicate by sending messages over named topics. In this test, the `talker` node publishes messages and the `listener` node subscribes to them. If they can talk to each other, the core ROS2 communication layer is functioning correctly.

First, confirm that the correct release is active:

```bash
echo $ROS_DISTRO   # should print: jazzy
ros2 --help        # should print the list of ros2 commands
```

Then open two separate terminal windows (or two SSH sessions) and run one command in each:

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

## Jetson Orin Nano Super Tips

These recommendations address common issues that arise specifically from running ROS2 on the Jetson's hardware, rather than a desktop machine.

- **Use the power supply included with the dev kit.** Builds and GPU workloads can draw significant power, especially in the higher power modes. An underpowered supply can cause throttling or unexpected shutdowns.
- **Check your power mode.** The Orin Nano Super has several power modes, including a higher-performance "Super" mode. Check the current mode and list the available ones (mode names and numbers may differ between JetPack releases, so confirm them on your own device rather than copying numbers from a guide):

```bash
sudo nvpmodel -q                     # show the current power mode
grep -i "POWER_MODEL" /etc/nvpmodel.conf   # list available modes and their IDs
sudo nvpmodel -m <mode_id>           # switch mode
sudo jetson_clocks                   # optional: lock clocks at maximum for the current mode
```

- **Keep the Jetson cool.** Make sure the fan and heatsink that ship with the dev kit are attached and spinning. CPU temperatures spike during builds, and without cooling the Jetson will thermally throttle, automatically reducing clock speed and slowing everything down.
- **Limit parallel build jobs to avoid running out of memory.** The Orin Nano Super has 8GB of RAM that is shared between the CPU and GPU. Large C++ packages built with `colcon` can use all of it and cause the build to freeze or be killed. If this happens, limit the number of parallel jobs:

```bash
colcon build --parallel-workers 2 --executor sequential
# or, for a single package with many compile jobs:
MAKEFLAGS="-j2" colcon build
```

- **Use an NVMe SSD, and consider adding swap.** The full desktop install, plus workspaces, logs, and datasets, adds up quickly, and an NVMe drive is far faster than a microSD card. If you still run out of memory during builds, adding a swap file on the NVMe drive gives the system extra headroom.
- **Monitor the system with `jtop`.** The `jetson-stats` package provides a live view of CPU, GPU, memory, temperature, and power use, which is very useful for spotting throttling or memory pressure while ROS2 nodes are running:

```bash
sudo apt install python3-pip
sudo pip3 install -U jetson-stats --break-system-packages
jtop
```

- **For navigation or high-throughput topics, consider Cyclone DDS over the default Fast DDS.** DDS (Data Distribution Service) is the underlying communication protocol ROS2 uses to pass messages between nodes. Cyclone DDS is a leaner alternative that many users find more stable on constrained hardware and in multi-machine setups:

```bash
sudo apt install ros-jazzy-rmw-cyclonedds-cpp
echo "export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp" >> ~/.bashrc
```

- **Use a `ROS_DOMAIN_ID` when sharing a network.** If other ROS2 machines are on the same network, they will see each other's topics by default. Pick a number between 0 and 101 and set it on every machine that should communicate:

```bash
echo "export ROS_DOMAIN_ID=42" >> ~/.bashrc
```

- **Cameras and other hardware drivers may need updates.** Because JetPack 7.2 uses a new kernel (6.8) and a new Jetson Linux release, out-of-tree kernel modules and vendor camera drivers built for earlier JetPack versions may need to be rebuilt or replaced. Check your hardware vendor's documentation for JetPack 7.2 support.
