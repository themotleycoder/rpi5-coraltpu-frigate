# Coral AI PCIe on Raspberry PI5
Experiment to setup Coral AI PCIe Accelerator on a Raspberry PI 5 and run Frigate to test it all out

## Equipment
- [Raspberry PI 5](https://www.raspberrypi.com/products/raspberry-pi-5/) (I am using 8gb version)
- SD Card (I am using 64gb but bigger is better for this)
- PI 5 [Power Supply](https://www.raspberrypi.com/products/27w-power-supply/)
- Coral [M.2 Accelerator A+E key](https://coral.ai/products/m2-accelerator-ae)
- Pineberry [Hat AI](https://pineberrypi.com/products/hat-ai-for-raspberry-pi-5)

## Setup Coral

Use this shortcut file provided by [DataSlayerMedia](https://www.youtube.com/@DataSlayerMedia) to get all the requried depencies downloaded and installed.

```Curl https://gist.githubusercontent.com/dataslayermedia/714ec5a9601249d9ee754919dea49c7e/raw/11bbda5da9c9539281e9e77215e3cbc9d698a5e3/coral-ai-pcie-edge-tpu-raspberrypi-5-setup | sh```

Test thatyour pi can see the coral device

```ls /dev/apex_0```

Install Docker:
```sudo apt update```
```sudo apt install docker.io```

Check docker is working
```sudo docker ps -a```

Create a Dockerfile:
```
FROM debian:10
WORKDIR /home
ENV HOME /home
RUN cd ~
RUN apt-get update  
RUN apt-get install -y git nano python3-pip python-dev pkg-config wget usbutils curl
RUN echo "deb https://packages.cloud.google.com/apt coral-edgetpu-stable main” \
| tee /etc/apt/sources.list.d/coral-edgetpu.list
RUN curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
RUN apt-get update
RUN apt-get install -y edgetpu-examples
```

Build Docker container:
```sudo docker build -t "coral" .```


Run Docker container:
```sudo docker run -it —device /dev/apex_0:/dev/apex_0 coral /bin/bash```

In the container command prompt run the following to test that everything is working:
```python3 /usr/share/edgetpu/examples/classify_image.py --model /usr/share/edgetpu/examples/models/mobilenet_v2_1.0_224_inat_bird_quant_edgetpu.tflite --label /usr/share/edgetpu/examples/models/inat_bird_labels.txt --image /usr/share/edgetpu/examples/images/bird.bmp```

It will identify a bird image from the examples directory the output should something like...
```
---------------------------
Poecile atricapillus (Black-capped Chickadee)
Score :  0.44140625
---------------------------
Poecile carolinensis (Carolina Chickadee)
Score :  0.29296875
root@9698d00cee58:~# python3 /usr/share/edgetpu/examples/classify_image.py --model /usr/share/edgetpu/examples/models/mobilenet_v2_1.0_224_inat_bird_quant_edgetpu.tflite --label /usr/share/edgetpu/examples/models/inat_bird_labels.txt --image /usr/share/edgetpu/examples/images/bird.bmp
---------------------------
```

## Setup Frigate
This takes 3 steps
### 1. Install MQTT
Install MQTT (Mosquitto)
Getting Started Guide: https://www.howtoforge.com/how-to-install-mosquitto-mqtt-messagebroker-
on-debian-11/
```sudo apt install mosquitto mosquitto-clients```

Edit the MQTT config file.
```nano /etc/mosquitto/mosquitto.conf```

Add these two lines at the top of the config file
```
allow_anonymous true
listener 1883
```

Save & restart the service.
```systemctl restart mosquitto```
Check the status of the mosquitto service
```sudo systemctl status mosquitto```

### 2. Install MediamTX
Download to your Raspberry PI
```wget https://github.com/bluenviron/mediamtx/releases/download/v1.4.2/mediamtx_v1.4.2_linux_arm64v8.tar.gz```

Extract the downloaded file
```tar -xzvf mediamtx_v1.4.2_linux_arm64v8.tar.gz```

Run the server by passing your IP address and an open port (554, in my case)
```RTSP_RTSPADDRESS=192.XXX.X.XXX:554 ./mediamtx```

It will now be listening for an FFMEG stream

Use FFMPEG to stream a feed to the RTSP server
```sudo ffmpeg -f v4l2 -input_format mjpeg -video_size 1280x720 -framerate 30 -i /dev/video0  -f rtsp -rtsp_transport tcp rtsp://192.X.X.X:554/mystream```

It should now be sending the stream to the listener

### 3. Install Frigate (in Docker)
Now we need to setup Frigate to leverage the created camera stream

Create a directory off of root called “frigate”. Add a config.yml file (see `config.yml` in this repo)

```
cd /
mkdir frigate
cd /frigate
```

Paste the provide YAML file and now edit it

```sudo nano config.yml```

Insert your RPI IP address in the provided spaces

Run Frigate in Docker:
```
docker run -d \
 --name frigate \
 --restart=unless-stopped \
 --mount type=tmpfs,target=/tmp/cache,tmpfs-size=1000000000 \
 --device /dev/apex_0:/dev/apex_0 \
 --shm-size=512m \
 -v /frigate/storage:/media/frigate \
 -v /frigate/config.yml:/config/config.yml \
 -v /etc/localtime:/etc/localtime:ro \
 -e FRIGATE_RTSP_PASSWORD='password' \
 -p 5000:5000 \
 -p 8555:8555/tcp \
 -p 8555:8555/udp \
 ghcr.io/blakeblackshear/frigate:stable
```

In a browser goto `localhost:5000` and you should see Frigate load up, if you click 'Birdseye' you should now see a live stream of your camera

Now click on 'Logs' and look for the following line `frigate.detectors.plugins.edgetpu_tfl INFO    : TPU found` this will let you know that it has found the TPU