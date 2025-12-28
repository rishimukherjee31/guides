# Running Pytorch on Jetson Orin/Nano using Docker and L4T for a Native Ubuntu Installation (22.04)

This README guide is tailored for a Jetson Orin NX running Ubuntu 22.04 with the NVIDIA Container Runtime already installed. The setup runs intensive CUDA/PyTorch workloads inside a container to keep the base Ubuntu 22.04 host system clean. It uses NVIDIA's L4T (Linux for Tegra) optimized containers to bypass host-side CUDA configuration issues.

> Jetson Orin NX: PyTorch in Docker (CUDA Accelerated)

<br>

## 1. Prerequisites
Ensure the NVIDIA Container Toolkit is correctly linked to your Docker daemon on the host:
bash

### Configure Docker to use the NVIDIA runtime by default
```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

<br>

## 2. Recommended Docker Image
- For JetPack 6.x (Ubuntu 22.04), use the official NVIDIA PyTorch container. These images are pre-configured with the correct CUDA, cuDNN, and TensorRT versions for Jetson hardware.
- Registry: ```nvcr.io/nvidia/pytorch```
- Target Tag: ```24.04-py3``` (Commonly used for JetPack 6 environments)

<br>

## 3. Deployment Command
Run the following command to start a shell with GPU access and your local code mounted:

```bash
docker run -it --rm \
    --runtime nvidia \
    --network host \
    --shm-size=2g \
    -v /path/to/your/project:/workspace \
    nvcr.io/nvidia/pytorch:24.04-py3
```

### Flag Explanations:

- ```--runtime nvidia```: Essential for passing the GPU into the container.
- ```--network host```: Allows the container to share the host's network for easy API/web access.
- ```--shm-size=2g```: Increases shared memory, preventing "bus error" crashes during heavy PyTorch data loading.
- ```-v /host/path:/container/path```: Syncs your local project folder with the container.

 <br>
  
## 4. Verification (Inside Container)
Once inside the container shell, verify that PyTorch is using the Jetson GPU correctly:

```python
python3 -c "import torch; print(f'CUDA Available: {torch.cuda.is_available()}'); print(f'Device: {torch.cuda.get_device_name(0)}')"
```

<br>

## 5. Performance Optimization
To ensure the Orin NX provides maximum power to your containerized workload, run these commands on the host before starting your intensive tasks:
```bash
# Set to Maximum Performance Mode
sudo nvpmodel -m 0
# Lock clocks to maximum frequency
sudo jetson_clocks
```

<br>

## 6. Troubleshooting
- Permission Denied: If docker requires ```sudo```, add your user to the docker group: ```sudo usermod -aG docker $USER``` (requires logout/login).
- Missing ```nvcc```: In JetPack 6 containers, CUDA tools are typically located in ```bash /usr/local/cuda/bin```. You may need to add this to your ```PATH``` inside the container if it's missing.



