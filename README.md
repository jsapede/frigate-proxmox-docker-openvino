# frigate-proxmox-docker-openvino
Complete setting for OpenVINO hardware acceleration in friggate

# Prerequisites

Intel iX > GEN6 architecture (i.e. compatible with openvino acceleration)

check in your PVE Shell that `/dev/dri/renderD128` is available

optionnally install intel GPU tools :

```
apt install intel-gpu-tools
```

now you can check GPU access / usage : 

```
intel_gpu_top
```
should lead to something like this : 

![image](https://github.com/user-attachments/assets/0474d76c-e4c7-45df-8023-5dc10809c01c)


# create Docker LXC :

The easiest way is to use [Tteck's scripts](https://tteck.github.io/Proxmox/)

first in the PVE console launch the tteck's script to install a new docker LXC : 

```
bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/ct/docker.sh)"
```

during installation : 
-  switch to "advanced mode"
-  select debian 12
-  make the LXC **PRIVILEGED**
-  you'd better choose 8Go ram and 2 or 4 cores
-  add portainer if needed
-  add docker compose

Once the LXC is created you have to can also install intel-gpu-tools **inside** the LXC

```
apt install intel-gpu-tools
```

Next you have to add GPU passthrough to the LXC to allow frigate access the OpenVINO acceleratons. On your LXC "Ressources" add "Device Passtrough" : 

![image](https://github.com/user-attachments/assets/071007bb-ad90-43c9-92ac-0c79313b83eb)

and specify the path you want to add : `/dev/dri/renderD128`

**Reboot**

now your LXC has access to the GPU.

# Frigate Docker

## create folders

On the LXC shell, create folders to organize your frigate storage for videos / captures / models and configs.

Here are my usually settings : 

/opt/frigate : docker-compose.yml location
/opt/frigate/media : storage
/opt/frigate/config: config storage

create the forlders according to your needs

next we will build the docker container. 

create a docker-compose.yml at the root folder or create a stack in portainer and add : 

```
version: "3.9"
services:
  frigate:
    container_name: frigate
    privileged: true 
    restart: unless-stopped
    image: ghcr.io/blakeblackshear/frigate:0.14.1
    cap_add:
      - CAP_PERFMON
    shm_size: "256mb"
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
      - /dev/dri/card0:/dev/dri/card0
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /opt/frigate/config:/config
      - /opt/media:/media/frigate
      - type: tmpfs
        target: /tmp/cache
        tmpfs:
          size: 1G
    ports:
      - "5000:5000"
      - "8971:8971"
      - "1984:1984"
      - "8554:8554" # RTSP feeds
      - "8555:8555/tcp" # WebRTC over tcp
      - "8555:8555/udp" # WebRTC over udp
    environment:
      FRIGATE_RTSP_PASSWORD: ****
      PLUS_API_KEY: ****
```

as you can see : 

- container is **privileged**
- /dev/dri/renderD128 is passtrhough from the LXC to the container
- created folders are bind to frigate usual folders
- shm_size has to be set according to [documentation](https://docs.frigate.video/frigate/installation/#calculating-required-shm-size)
- tmpfs has to be adjusted to your configuration, see [documentation](https://docs.frigate.video/frigate/installation/#storage)



