# Installing Ubuntu on the Jetson Orin Nx

This document will serve as a guide on how to install the latest Ubuntu image (22.04 as of writing) on a Jetson Orin NX. 
This process requires some hardware modifications, so make sure to have some female-to-female jumper cables. 

> I will be uploading the image to an NVME drive and then using the drive to load Ubuntu on the Jetson. However, there are instructions 
for flashing the software using a USB drive as well. 

## Step 1: Put the Jetson in recovery mode

On the Jetson Orin NX this is done by shorting pin 9 and pin 10 (GND and FC_REC) with a jumper cable. You can then connect a USB-C cable from the Jetson to a host machine (I used an Alienware laptop running Linux).  Now, when you power on the Jetson, you will be able to see the Jetson on your host machine using ```lsusb```. 

```bash
lsusb
# output: Bus 001 Device 008: ID 0955:7323 Nvidia Corp.
```

The device is now in recovery mode.

## Step 2: Prepare the Linux for Tegra Bootloader

You will need this to read the Ubuntu boot image. With this bootloader, you can also run different Linux versions compatible with our Orin. This is also essential to run an OS off a bootable drive. The Linux for Tegra tools installed with the boot firmware tarball contain scripts used to flash the software of a Jetson board, both firmware and operating system.

```bash
sudo apt install -y python3 mkbootimg bzip2 cpp device-tree-compiler
```

Extract the boot firmware tarball

```bash
tar xf Jetson_Linux_R36.4.3_aarch64.tbz2 && cd Linux_for_Tegra/
```

Install missing dependencies and fix file permissions

```bash
sudo ./tools/l4t_flash_prerequisites.sh
```

From the Linux_for_Tegra directory, enter the following command to program the latest QSPI boot firmware. It will then reboot the kit automatically upon success. You will need to make sure to select the settings for the Jetson Orin NX. Run the following command:

```bash
# QSPI for Jetson Orin Nano/NX
sudo ./flash.sh p3768-0000-p3767-0000-a0-qspi internal
```


