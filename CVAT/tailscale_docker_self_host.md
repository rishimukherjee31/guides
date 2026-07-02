# Self-Host CVAT on Ubuntu 24.04 with Docker

This repository contains the instructions to host CVAT on a Ubuntu 24.04 server/machine. 
If you have a GPU, there is an option to autolabel your dataset using pretrained models through
```nuclio```. This guide uses tailscale to host the website on a local network. You can also 
host CVAT on an [AWS](https://aws.amazon.com/free/?trk=dc9b9d60-cc82-4cd5-8a61-0b33d6a79fab&sc_channel=ps&ef_id=CjwKCAjwhLPOBhBiEiwA8_wJHMc0rR3EuwL-BUwiOoxxZ1MAYm09RItY5nqFTuOAb1EiaAbVlttQ9hoCSnoQAvD_BwE:G:s&s_kwcid=AL!4422!3!795877020713!e!!g!!aws!23532472510!199502799824&gad_campaignid=23532472510&gbraid=0AAAAADjHtp8h4DERHMzwHrUeuZC0B-Wgj&gclid=CjwKCAjwhLPOBhBiEiwA8_wJHMc0rR3EuwL-BUwiOoxxZ1MAYm09RItY5nqFTuOAb1EiaAbVlttQ9hoCSnoQAvD_BwE)
webserver, or on your own LAN as well. 

The setup is meant for about 5 users (including 1 admin). 
The dataset lives in a plain directory on the server and is shared into CVAT, so users pick files from the 
server instead of uploading their own.

> The user can add images as well however since it is through ssh, it will be slow. I usually add the data on my own on the admin side as it is faster and more reliable. The client will simply complete the jobs assigned to them. 

<br>

## Prequisites
You will need Docker to download the prebuilt CVAT containers that allow you to do this process. 
CVAT is part of OpenCV and thus works on most systems. It can be configured locally from source, 
however, Docker makes the process a lot smoother and we will therefore use Docker for our build.

<br>

### 1. Install Docker
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

<br>

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

<br>

## Set Up the Dataset Directory
Before installing CVAT, create a plain directory on the server to hold your data. This is the 
"share" that CVAT will read from, so users can pick files straight from the server instead of 
uploading anything from their own machines.

```bash
sudo mkdir -p /mnt/cvat_share
sudo chmod -R 775 /mnt/cvat_share
```

Now sort your data into this folder. Organize it however makes sense for your projects — CVAT 
shows this whole directory tree in the task-creation dialog, so a clean layout here makes tasks 
easy to build later. For example:

```bash
/mnt/cvat_share/
├── project_a/
│   ├── images/          # .jpg / .png frames
│   └── videos/          # .mp4 etc.
└── project_b/
    └── images/
```

Copy your datasets in with a normal ```cp``` (or ```rsync``` / ```scp``` from another machine):

```bash
cp -r ~/datasets/my_project /mnt/cvat_share/
```

> The original files stay right here in ```/mnt/cvat_share``` — CVAT reads them in place and never 
> copies them out. This is the folder you manage, back up, and add new data to.

<br>

## Install CVAT with Serverless Support
You will need to set the ```CVAT_HOST``` in this step as well. I am using tailscale so I will be using the tailscale IP.

### Install CVAT on your Machine
CVAT from source:
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
> If you're SSH-ing in with a port forward instead of tailscale (see the Access section below), 
> set ```export CVAT_HOST=localhost``` since that's the address you'll type in the browser.

<br>

### 3. Point CVAT at the Dataset Directory
CVAT needs to know about the ```/mnt/cvat_share``` folder you made. Create a file called 
```docker-compose.override.yml``` inside the ```cvat/``` directory:

```bash
nano docker-compose.override.yml
```

Set the contents to:
```yaml
services:
  cvat_server:
    volumes:
      - cvat_share:/home/django/share:ro
  cvat_worker_import:
    volumes:
      - cvat_share:/home/django/share:ro
  cvat_worker_chunks:
    volumes:
      - cvat_share:/home/django/share:ro

volumes:
  cvat_share:
    driver_opts:
      type: none
      device: /mnt/cvat_share
      o: bind
```

> This mounts your data folder into CVAT read-only (```:ro```). That's all the config the file 
> sharing needs. Your folders show up under the "Connected file share" tab.

<br>

### 4. Start CVAT
If you don't need GPU access for autolabeling, you can use the following command to start CVAT:
```bash
docker compose up -d
```

For GPU access, use the following command:
```bash
docker compose \
  -f docker-compose.yml \
  -f docker-compose.override.yml \
  -f components/serverless/docker-compose.serverless.yml \
  up -d
```
> This may take some time as it will install all of the required dependencies.
> Note: when you pass explicit ```-f``` flags, you have to include ```docker-compose.override.yml``` 
> yourself as it only loads automatically when no ```-f``` flags are given.

Now you can create the super user:
```bash
docker exec -it cvat_server bash -ic 'python3 ~/manage.py createsuperuser'
```
> Choose a username and a password for your admin account.

<br>

### 5. Add Your Users
The admin account you just made manages everyone else. Open the admin panel in a browser at 
```http://<CVAT_HOST>:8080/admin```, then go to **Auth -> Users -> Add user**. Set a username and 
temporary password for each of your 3–4 annotators and save. Leave *Staff* and *Superuser* 
unchecked for them. Make sure to add the credientials as the admin and have the users change passwords later.

<br>

### 6. Make a Task from the Shared Data
In the web UI, go to **Tasks -> Create a new task**, define your labels, then under **Select files** 
open the **"Connected file share"** tab. You'll see the contents of ```/mnt/cvat_share```. Pick the images/videos for the task straight from the server; 
nothing gets uploaded from anyone's machine.

<br>

## Access from the Remote Client
CVAT will expose the website in port 8080. You can access the CVAT webpage in your browser
by typing in the ```IP Address``` of the server followed by the port number: ```http://100.x.x.x:8080```.
Tailscale is secure so it handles all of the secure tunneling. If you are solely on a LAN, 
you may need to configure port forwarding settings and firewalls if necessary. 

Note: if you're directly SSH-ing and not using tailscale, you will need forward a local port from your device to 
the server to get it to run on the browser. In other words SSH with the following command:

```bash
ssh -L 8080:localhost:8080 user@<server-ip>
```

Where ```user@<server-ip>``` is what you would use to normally SSH into the machine. While that 
SSH session stays open, go to your browser and visit ```http://localhost:8080```. Make sure to set ```CVAT_HOST=localhost``` (see the step 
above) so the address matches. Now you can create a user and upload the data.

> If port 8080 is already taken on your laptop, just change the left number, e.g. 
> ```ssh -L 9090:localhost:8080 user@<server-ip>``` and browse to ```http://localhost:9090```.

<br>

### 7. Stop the Docker Container
Use the same ```-f``` flags you started with:
```bash
docker compose -f docker-compose.yml \
  -f docker-compose.override.yml \
  -f components/serverless/docker-compose.serverless.yml down
```
> Your annotations, users, and tasks are kept in Docker volumes and survive ```down``` and reboots. 
> Don't use ```down -v``` unless you want to wipe everything.

---
*For more detailed instructions visit [this link](https://docs.cvat.ai/docs/administration/community/basics/installation/), specifically to organize the data directory etc.*
