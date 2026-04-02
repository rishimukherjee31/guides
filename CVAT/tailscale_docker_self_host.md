# Self-Host CVAT on Ubuntu 24.04 with Docker

This repository contains the instructions to host CVAT on a Ubuntu 24.04 server/machine. 
If you have a GPU, there is an option to autolabel your dataset using pretrained models through
```nuclio```. This guide uses tailscale to host the website on a local network. You can also 
host CVAT on an [AWS](https://aws.amazon.com/free/?trk=dc9b9d60-cc82-4cd5-8a61-0b33d6a79fab&sc_channel=ps&ef_id=CjwKCAjwhLPOBhBiEiwA8_wJHMc0rR3EuwL-BUwiOoxxZ1MAYm09RItY5nqFTuOAb1EiaAbVlttQ9hoCSnoQAvD_BwE:G:s&s_kwcid=AL!4422!3!795877020713!e!!g!!aws!23532472510!199502799824&gad_campaignid=23532472510&gbraid=0AAAAADjHtp8h4DERHMzwHrUeuZC0B-Wgj&gclid=CjwKCAjwhLPOBhBiEiwA8_wJHMc0rR3EuwL-BUwiOoxxZ1MAYm09RItY5nqFTuOAb1EiaAbVlttQ9hoCSnoQAvD_BwE)
webserver, or on your own LAN as well. 

## Prequisites
You will need Docker to download the prebuilt CVAT containers that allow you to do this process. 
CVAT is part of OpenCV and thus works on most systems. It can be configured locally from source, 
however, Docker makes the process a lot smoother and we will therefore use Docker for our build.

### 2. Install Docker
First, you will need Docker. In a terminal type the following set of commands:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl git

sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

> Add your user to the docker group so you don't need sudo every time:

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
# Log out and back in, then verify:
groups
```

### 2. (Optional) For GPU Access in the Docker Container
```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

This is critical for Nuclio GPU functions to work. Edit ```/etc/docker/daemon.json``` to set ```NVIDIA``` as the default runtime: 
```bash
sudo nano /etc/docker/daemon.json
```

Set the contents to:
```json
{
  "default-runtime": "nvidia",
  "runtimes": {
    "nvidia": {
      "path": "nvidia-container-runtime",
      "runtimeArgs": []
    }
  }
}
```
```bash
sudo systemctl restart docker
# Verify GPU is accessible in Docker:
docker run --rm --gpus all nvidia/cuda:12.0-base-ubuntu22.04 nvidia-smi
```

### 3. Install CVAT with Serverless Support
Install CVAT from the github page. You will need to set the ```CVAT_HOST``` in this step as well.
I am using tailscale so I will be using the tailscale IP:
```bash
git clone https://github.com/cvat-ai/cvat
cd cvat

# Set CVAT_HOST to the Tailscale IP so the server is exposed to the network
export CVAT_HOST=100.x.x.x   # replace with your IP
```
> If you want to use AWS, local IP, replace ```CVAT_HOST``` to the your IPv4 address.
> If you're using tailscale, you can find it with the following command:

```bash
tailscale ip -4
# 100.x.x.x
```

### 4. Start CVAT
If you don't need GPU access for autolabeling, you can use the following command to start CVAT:
```bash
docker compose up
```

For GPU access, use the following command:
```bash
docker compose \
  -f docker-compose.yml \
  -f components/serverless/docker-compose.serverless.yml \
  up -d
```
> This may take some time as it will install all of the required dependencies.

Now you can create the super user
```bash
docker exec -it cvat_server bash -ic 'python3 ~/manage.py createsuperuser'
```

### 5. Access from the Remote Client
CVAT will expose the website in port 8080. You can access the CVAT webpage in your browser
by typing in the ```IP Address``` of the server followed by the port number: ```http//:100.x.x.x:8080```.
Tailscale is secure so it handles all of the secure tunneling. If you are solely on a LAN, 
you may need to configure port forwarding settings and firewalls if necessary. 





