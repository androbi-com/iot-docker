# iot-docker stack

## What is this?

This repository contains a docker-compose stack that I use for my Raspberry 
Pi 4 (4 Gb) which is dedicated mainly to my IOT experiments (I have various 
sensors and actuators deployed and I use node-red as a control central, 
with zigbee2mqtt for the zigbee devices and influxdb for storing (and 
displaying) the values. The docker stack currently deploys

* `traefik` as a reverse proxy
* `nginx` for displaying a simple web page with links to the applications
* `portainer-ce` for controlling the deployed containers
* `mosquitto` as a mqtt server
* `zigbee2mqtt` for access to zigbee devices (I use CC2652R USB stick in the Pi)
* `node-red` as a programmable controller
* `influxdb` (2.0) to store and visualize data
* more to come ..

If you don't need zigbee, you can just comment out the zigbee2mqtt section
in `docker-compose.yml`.

I wanted to include the possibility to access the mqtt server from outside
my local network without using a vpn. This requires careful security 
considerations (we need tls for the mqtt service and all other services
must be protected). For this reason I employ `traefik` as a "gate keeper"
which handles the certificates via https://letsencrypt.org/ and
restricts access to the local resources.

`node-red` comes with a flow and a dashboard preinstalled that checks 
the current connection status with the mqtt server. After starting 
the stack you can add your own flows or nodes. `portainer-ce` lets 
you control the deployed containers, you have to set up an admin user
when entering for the first time.

The frontend for `zigbee2mqtt` is also exposed (currently 
without authentication on the local network), `influxdb` guides the
user through a setup process with its nice UI.

I have also included a very simple web page that is available on 
http://raspi.local (substitute `raspi` by your hostname) which
provides links to the above mentioned applications.

## Preparations

If you intend to use influxdb v 2.0, you'll need a 64 bit operating system 
on your Raspberry Pi for this stack (infuxdb 2.0 is not available on 32 bit). 
If influxdb v 1.8 is ok for you a 32 bit OS should suffice. When using 
the Raspian OS you have to take care to use `pi` instead of `ubuntu` 
as username in the following commands.

In my setup I have used Ubuntu server 20.04 64 bit (see 
https://ubuntu.com/tutorials/how-to-install-ubuntu-on-your-raspberry-pi)
as the official Raspberry Pi OS (64 bit) was still in beta at the time of 
this writing. 

You will also need Docker (I have followed the instuctions in
https://docs.docker.com/engine/install/ubuntu/ using the "Install using the 
repository" method. After the install I have added the user `ubuntu` 
to the `docker` group and have installed `docker-compose` with 
`sudo apt-get install docker-compose`. 

I have renamed my hostname (`sudo vi /etc/hostname`) to be `raspi` and I
have installed the `avahi-daemon` on the Pi

    sudo apt-get install avahi-daemon

so that `raspi.local` can be resolved on the local network. Nevertheless
it is a good idea to add an entry for `raspi.local` to `/etc/hosts` on
the clients you use to access your Raspberry Pi. It will make name resolution 
faster (and more reliable - for me the mDNS resolution does not always
seem to work, I have not yet found the reason).

If you choose another hostname you have to adapt the links in `index.html` in 
`setup/nginx/html/web_local` accordingly.

For access from outside your local network you'll probably be using
some sort of dynamic DNS resolver like duckdns.org. For this to work 
you need to open ports 80,443, 1883 and 8883 on your router and redirect
requests to the local IP of your Pi.

You need to create a file `.env` in this directory (which is not included
in the repository) which contains the following environment variables

    LOCAL_HOST=raspi.local
    REMOTE_HOST=duckduck.duckdns.org

where I have used `duckduck` as an example for the host name for remote 
access, please adjust accordingly.

A recommended but not obligatory install is `log2ram`, see
https://github.com/azlux/log2ram, I installed it using the apt method. This 
helps reducing wear on the SD from the system logs.

If you want this setup to work for a longer timer span (and be faster), 
the `volumes` directory should be mounted from a SSD instead of a SD.

## Installation

The directory `setup` is a template for the `volumes` directory,
which contains all volumes (represented as bind mounted directories)
necessary for the applications in this stack. The idea is that the command

    cp -r setup volumes 

should create a starting point for running the stack, providing all 
necessary directories and configuration files. For some applications 
you might need to make some modifications to configuration files
for your specific setup

### zigbee2mqtt

The `zigbee2mqtt` configuration in `docker-compose.yml` maps the 
device corresponding to the USB stick to `/dev/ttyACM0` inside the 
container. In my case, the device is `/dev/ttyUSB0`, so I map 
`/dev/ttyUSB0` to `/dev/ttyACM0`. Check your device with 
`ls /dev/tty*` and adjust the `docker-compose.yml` accordingly. 

We also need to give permissions for the user to write to the device. 
On Ubuntu 20.04 this can be done by adding the user `ubuntu` to the 
dialout group

    sudo usermod -aG dialout ubuntu
    su - $USER

### setting your own passwords

The IOT-Docker stack uses a predefined user/password for the mqtt server, 
so everything should work right out of the box in a local setup. This is
a big no-no if you'll expose your mqtt server outside your local
network. The used default username and password can be seen in the
`zigbee2mqtt/data/configuration.yaml` file. If you just want to give it 
a try on your local Pi you can skip this section, but it is good 
practice to change the password even in this case. 

If order to change the default user and password before running the stack
first remove the existing password entry for user `mqtt` with the 
following command 

    docker run -i --rm -v ${PWD}/volumes/mosquitto/config:/mosquitto/config \
      eclipse-mosquitto mosquitto_passwd -D /
      mosquitto/config/mosquitto.passwd mqtt

The command may take a while to start if you don't have a local copy
of the `eclipse-mosquitto` image. Then set a new password for the 
user of your choice (change `user` in what follows)

    docker run -i --rm -v ${PWD}/volumes/mosquitto/config:/mosquitto/config \
      eclipse-mosquitto mosquitto_passwd /
      mosquitto/config/mosquitto.passwd user

The default user/password combination is referenced from within Node-RED 
in the global configuration node `mqtt-broker/mosquitto`. In Node-RED, 
credentials are encrypted in `data/flows_cred.json`. The key used 
to encrypt these credentials is configured in `settings.js`, 
see `credentialSecret`. After changing the mosquitto password
you should first change the setting `credentialSecret` in 
`volumes/node-red/data/settings.js`.

Node-RED will then invalidate the current credentials and you have 
to enter them again by editing the `mqtt-broker/mosquitto` node 
(tab security) after starting the stack (next section). 

The same user/password combination also has to be updated in 
zigbee2mqtt, as it uses the mqtt server as a backend. Unfortunately 
zigbee2mqtt stores the mqtt user and password as cleartext in 
`data/configuration.yaml`. Edit the file and update user and 
password to the new value.

### start services

We now are ready to start the stack.

    docker-compose up -d 

and open http://raspi.local:8080 (substitute your host name). 
Open the Node-Red dashboard and check if there is a connection with
the mqtt server.


## Using telegraf with InfluxDB

In order to obtain data about your Raspberry current CPU usage, disk I/O etc. you
can use the Telegraf agent which is executed directly on your Pi and communicates the
data to the InfluxDB. Here is how to set this up:

* follow https://docs.influxdata.com/telegraf/v1.20/introduction/installation/ to install telegraf
* use the Influx UI to create a configuration: Data -> Telegraf -> Create Configuration
* run telegraf interactively to see if it works as indicated
* download configuration and include token in file. Copy to /etc/telegraf/telegraf.conf
* run `sudo systemctl start telegraf` 

## TODO list

* try pivpn with ubuntu 64 bit?
