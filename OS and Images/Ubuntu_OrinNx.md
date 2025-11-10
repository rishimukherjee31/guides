# Installing Ubuntu on the Jetson Orin Nx

This document will serve as a guide on how to install the latest Ubuntu image (22.04 as of writing) on a Jetson Orin NX. 
This process requires some hardware modifications, so make sure to have some female-to-female jumper cables. 

> I will be uploading the image to an NVME drive and then using the drive to load ubuntu on the Jetson. However, there are instructions 
for flashing the software using a USB drive as well. 

## Step 1: Put the Jetson in recovery mode

On the Jetson Orin NX this is done by shorting pin 9 and pin 10 (GND and FC_REC) with a jumper cable. You can then connect a USB-C cable from the Jetson to a host machine (I used an Alienware laptop running Linux).  Now, when you power on the Jetson, you will be able to see the Jetson on your host machine using ```lsusb```. 

```bash
lsusb
# output: Bus 001 Device 008: ID 0955:7323 Nvidia Corp.
```

The device is now in recovery mode.

## Step 2: Prepare the Linux for Tegra Bootloader

You will need this to read the Ubuntu boot image. With this bootloader, you can also run different Linux versions compatible with our Orin. This is also essential to run an OS off a bootable drive.
