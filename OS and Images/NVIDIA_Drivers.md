# NVIDIA Runtime Installation Guide for the Jetson Orin NX

This guide uses the [canonical-link](https://canonical-ubuntu-for-jetson.readthedocs-hosted.com/classic/installation-jammy/) on the Ubuntu website.
The process is specific to Ubuntu 22.04 running on a Jetson Orin NX. This is the latest supported version at the time of writing. 

<br>

## Install CUDA and TensorRT
SDKs like CUDA Toolkit and TensorRT that allow building AI applications on Jetson devices are available directly from NVIDIA:

```bash
# CUDA
sudo apt-key adv --fetch-keys "https://repo.download.nvidia.com/jetson/jetson-ota-public.asc"
sudo add-apt-repository -y "deb https://repo.download.nvidia.com/jetson/t234 r36.4 main"
sudo add-apt-repository -y "deb https://repo.download.nvidia.com/jetson/common r36.4 main"
sudo apt install -y cuda

# Tensor RT
sudo apt install -y libnvinfer-bin libnvinfer-samples

# cuda-samples dependencies
sudo apt install -y cmake

echo "export PATH=/usr/local/cuda-12.6/bin\${PATH:+:\${PATH}}" >> ~/.profile
echo "export LD_LIBRARY_PATH=/usr/local/cuda-12.6/lib64\${LD_LIBRARY_PATH:+:\${LD_LIBRARY_PATH}}" >> ~/.profile

# Logout or reboot to apply the profile change
sudo reboot
```

