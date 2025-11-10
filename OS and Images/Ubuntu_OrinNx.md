# Installing Ubuntu on the Jetson Orin NX

This document will serve as a guide on how to install the latest Ubuntu image (22.04 as of writing) on a Jetson Orin NX. This process requires some hardware modifications, so make sure to have some female-to-female jumper cables handy.

> I will be uploading the image to an NVMe drive and then using the drive to load Ubuntu on the Jetson. However, there are instructions for flashing the software using a USB drive as well.

<br>

- [Recovery Mode](#step-1-put-the-jetson-in-recovery-mode)
- [Bootloader Setup](#step-2-prepare-the-linux-for-tegra-bootloader)
- [Ubuntu Image](#step-3-download-latest-ubuntu-image)
- [Desktop Install](#step-4-run-ubuntu-on-the-orin-nx)

<br>

## Step 1: Put the Jetson in Recovery Mode

On the Jetson Orin NX this is done by shorting pin 9 and pin 10 (GND and FC_REC) with a jumper cable. You can then connect a USB-C cable from the Jetson to a host machine (I used an Alienware laptop running Linux). Now, when you power on the Jetson, you will be able to see the Jetson on your host machine using `lsusb`.

```bash
lsusb
# output: Bus 001 Device 008: ID 0955:7323 Nvidia Corp.
```

The device is now in recovery mode. 

<br>

## Step 2: Prepare the Linux for Tegra Bootloader

You will need a bootloader to load the image onto the Orin. With this bootloader, you can also run different Linux versions compatible with our Orin. This is also essential to run an OS off a bootable drive. The Linux for Tegra tools installed with the boot firmware tarball contain scripts used to flash the software of a Jetson board, both firmware and operating system.

First, grab the dependencies:

```bash
sudo apt install -y python3 mkbootimg bzip2 cpp device-tree-compiler
```

Extract the boot firmware tarball:

```bash
tar xf Jetson_Linux_R36.4.3_aarch64.tbz2 && cd Linux_for_Tegra/
```

Install missing dependencies and fix file permissions:

```bash
sudo ./tools/l4t_flash_prerequisites.sh
```

From the Linux_for_Tegra directory, enter the following command to program the latest QSPI boot firmware. It will then reboot the kit automatically upon success.

> Make sure to select the settings for the Jetson Orin NX. Run the following command:

```bash
# QSPI for Jetson Orin Nano/NX
sudo ./flash.sh p3768-0000-p3767-0000-a0-qspi internal
```

Reboot the Jetson once the command runs successfully. The difficult part is done.

<br>

## Step 3: Download Latest Ubuntu Image

With the bootloader flashed, prepare your bootable medium. I used a 1TB NVMe SSD (2280 form factor) for reliability and optimal performance. You will need to download the correct image: [Ubuntu 22.04 for Tegra](https://cdimage.ubuntu.com/releases/jammy/release/nvidia-tegra/ubuntu-22.04-preinstalled-server-arm64+tegra-jetson.img.xz).

Once you have this image in the host machine, you can upload it to the drive. Navigate to the image directory and run the following commands:

```bash
# Copy the image over the boot media (assuming here it is detected as /dev/sda)
xzcat ubuntu-22.04-preinstalled-server-arm64+tegra-jetson.img.xz | \
    sudo dd of=/dev/sda bs=16M status=progress
    
sudo sync
```

Once you have the image installed on the drive, you may see an option to 'run' the drive on a discrete GPU. This is because the OS thinks it can run on an existing NVIDIA GPU on the host machine. If you don't have a NVIDIA GPU, you may not see this. 

> If you are using a USB drive as a bootloader, simply copy the image into that drive instead. Same process.

<br>

## Step 4: Run Ubuntu on the Orin NX

The image installed is a server image. If you don't need a GUI, this is the end of the process. Once you follow the prompted steps for the Ubuntu installation, the Jetson will run Ubuntu on boot. To install Ubuntu desktop, run the following command in the shell:

```bash
sudo apt install ubuntu-desktop
```

> Once you install the desktop, reboot the system, and you will now have Ubuntu installed on your machine.
- The username and password for the initial Ubuntu account are `ubuntu`/`ubuntu`
- You will need to create a new account and change admin privileges to ensure security

You've now got Ubuntu running on your Jetson Orin NX. 
---

*If something goes wrong, double-check your jumper cable connections in step 1 – that's a common failure point. Also, make sure your USB-C cable isn't a charging-only cables. That may not work*
