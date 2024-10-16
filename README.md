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

```
mkdir /opt/frigate
mkdir /opt/frigate/media
mkdir /opt/frigate/config
```


create the forlders according to your needs

next we will build the docker container. 

create a docker-compose.yml at the root folder

```
cd /opt/frigate
nano docker-compose.yml
```

or create a stack in portainer :

![image](https://github.com/user-attachments/assets/3a4add3d-38ec-4313-a9b1-d6761057726c)

and add : 

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
- ports for UI, RTSP and webRTC are forwarded
- define some FRIGATE_RTSP_PASSWORD and PLUS_API_KEY if needed

From now the docker container is ready, and have access to the GPU.

**Do not start it right now as you have to provide frigate configuraton !**

## Setup Frigate for OpenVINO acceleration

add your frigate configutration :

```
cd /opt/frigate/config
nano config.yml
```
edit it accroding to your setup and now you must add the [following lines](https://docs.frigate.video/configuration/object_detectors/#openvino-detector) to your frigate config : 

```
detectors:
  ov:
    type: openvino
    device: GPU

model:
  width: 300
  height: 300
  input_tensor: nhwc
  input_pixel_format: bgr
  path: /openvino-model/ssdlite_mobilenet_v2.xml
  labelmap_path: /openvino-model/coco_91cl_bkgr.txt
```

Once your `config.yml` is ready build your container by either `docker compose up` or "deploy Stack" if you're using portainer


reboot all, and go to frigate UI to check everything is working : 

![image](https://github.com/user-attachments/assets/abad95f0-f0c9-4b59-853f-56a252e6bb65)

you should see : 
-  low inference time : ~20 ms
-  low CPU usage
-  GPU usage

you can also check with `intel_gpu_top` inside the LXC console and see that Render/3D has some loads according to frigate detections 

![image](https://github.com/user-attachments/assets/ce307fb5-e856-4846-b1ee-94a6bda9758a)

and on your PROXMOX, you can see that CPU load is drastically lowered : 

![image](https://github.com/user-attachments/assets/10f23477-ee7b-4ec3-8572-749d0a33c995)




