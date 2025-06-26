# openvino-whispercpp-wyoming
Openvino accelerated whispercpp wtih wyoming protocol for homeassistant

this repository is a tutorial for implementing Openvino hardware accleration on whisper (whispercpp) for local voice assistant usage in home assistant.

My server is build within docker over proxmox lxc but should be adapted for direct docker usage.

it relies on: 

  https://github.com/monoamin/wyoming-whispercpp-openvino-gpu
  
  https://github.com/ser/wyoming-whisper-api-client

#  1/ Environment setting

On proxmox create a Docker LXC, preferably using the "helper scripts" : https://community-scripts.github.io/ProxmoxVE/scripts?id=docker

select : 

-  advanced mode
-  privileged
-  8GB RAM
-  16GB storage
-  max CPU

  once lxc is created add you GPU passthrough to the LXC :

  ![image](https://github.com/user-attachments/assets/86990b1d-57f2-4796-a5da-c6939d571299)

reboot it !

in the LXC console add intel GPU tools to have monitoring on Openvino usage :

```
apt install intel-gpu-tools
```

you should now be abloe to see integrated GPU usage : 

```
intel_gpu_top
```

![image](https://github.com/user-attachments/assets/81270e0b-d297-4404-a5a6-e26aebbf91a1)

find your "video" group id (usualy 44)  :

```
getent group video
```

# 2/ Prepare Wyoming API

clone the wyoming whisper api client repo and build the docker image. In the lxc console : 

```
cd /opt
git clone https://github.com/ser/wyoming-whisper-api-client
cd wyoming-whisper-api-client
docker build -t wyoming-whisper-api-client:latest .
```


# 3/ Prepare WhisperCPP Openvino

clone the wyoming whispercpp openvino repo and build the docker image. In the lxc console : 

```
cd /opt
git clone https://github.com/monoamin/wyoming-whispercpp-openvino-gpu
```

Dockerfile provided in the repo should be renamed for easier usage :

```
cd wyoming-whispercpp-openvino-gpu
cp whispercpp-openvino.Dockerfile Dockerfile
```

**NOTE:**

Image building returns a warning on LD_LIBRARY_PATH, related to en ENV variable not defined in the Dockerfile at line 61 :

```
ENV LD_LIBRARY_PATH=$INTEL_OPENVINO_DIR/runtime/lib/intel64:$LD_LIBRARY_PATH
```

change it to

```
ENV LD_LIBRARY_PATH=$INTEL_OPENVINO_DIR/runtime/lib/intel64
```

now build the WhisperCPP image :

```
docker build -t whispercpp-openvino:latest .
```

now image whispercpp-openvino:latest should be locally available for docker usage.

# 4/ Get Models

GGML Models are available on Huggingface : https://huggingface.co/Intel/whisper.cpp-openvino-models/tree/main

In the LXC console, create a storage directory : 

```
cd /opt
mkdir ggml
cd ggml
```
now download one or multiple model packages using the download links and unpack them in the ggml folder :

![image](https://github.com/user-attachments/assets/f179dff2-8e08-4dc9-8719-b8160c3a6f28)

```
wget https://huggingface.co/Intel/whisper.cpp-openvino-models/resolve/main/ggml-medium-models.zip
unzip ggml-medium-models.zip
```

on low power iGPU you should use ggml-small-models.zip instead, to avoid many seconds latency.


# 5/ Build Docker Stack

a docker-compose file is provided int the wyoming-whispercpp-openvino-gpu that should ba adjusted to your configuration / language / hardware ..

create a docker-compose.yml 

```
cd /opt
nano docker-compose.yml
```

and add the stack description 

```
services:
  whisper: # wyoming endpoint for HASS
    container_name: whisper
    command:
      --uri tcp://0.0.0.0:10300
      --api http://192.168.0.233:8910/inference # adjust the IP to your framework 
      --debug
    image: wyoming-whisper-api-client:latest # the local image we built
    volumes:
      - /etc/localtime:/etc/localtime:ro
    environment:
      - TZ=Europe/Paris # adjust to your timezone
    restart: unless-stopped
    ports:
      - 10300:10300

  whispercpp: # whisper-server backend for wyoming 
    container_name: whispercpp
    privileged: true # ensure access to iGPU
    command:
      --language fr # adjust to your anguage
      --ov-e-device CPU # adjut according to your hardware CPU or GPU. iGPU doesnt work with GPU setting
      --beam-size 5
      --model /data/ggml-small.bin # specify the model you want to use. on iGPU use small
      --host 0.0.0.0
      --port 8910
      --debug-mode
    image: whispercpp-openvino:latest # the image we built
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128 # passtrhough of iGPU
      - /dev/dri/card0:/dev/dri/card0 #passthrough of iGPU. maybe card0 or card1. adjust
    security_opt:
      - seccomp=unconfined
    group_add:
      - 44 # the video group id
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /opt/ggml:/data # the folder where we store ggml models
    environment:
      - TZ=Europe/Paris #adjust to your timezone
    restart: unless-stopped
    ports:
      - 8910:8910
```

now you should be able to run the stack : 

```
docker compose up
```

or in detached mode :

```
docker compose up -d
```

# 6/ Add to Homeassistant :

in Homeassistant add the Wyoming protocol integration :

![image](https://github.com/user-attachments/assets/fd3c7054-7add-478f-84b6-cf6ff9a4d3bd)

then "add service" and specify the IP/Port of your wyoming API client : 

![image](https://github.com/user-attachments/assets/6f3cd83b-c50d-4d20-98a1-0f611a6642d4)

now WhisperCPP with Openvino Acceleration should be available for voice assistant :

![image](https://github.com/user-attachments/assets/613e688d-3705-4f46-87ef-71770e97e945)



