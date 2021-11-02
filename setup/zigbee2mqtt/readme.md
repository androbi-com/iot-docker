# zigbee2mqtt

We map `data` to `/app/data` inside the container. An initial configuration
is supplied in `configuration.yaml` which permits devices to join. Once
you have paired your devices you should set `permit_join` to `false`.

The configuration in `docker-compose.yml` also maps the device file
(where the USB stick is connected) to `/dev/ttyACM0` inside the container.
Check the devices that show up in your Pi with `ls -l /dev/tty*` and
adjust the device mapping in docker-compose.yml. If your device is
`/dev/ttyACM0`, you would set

    devices:
      - /dev/ttyACM0:/dev/ttyACM0

or, if your device is `/dev/ttyUSB0`, you would use

    devices:
      - /dev/ttyUSB0:/dev/ttyACM0

The user needs permissions to write to the device on the host, on 
Ubuntu 20.04 we get

    ubuntu@raspi:~$ ls -l /dev/ttyUSB0 
    crw-rw---- 1 root dialout 188, 0 Sep  7 18:37 /dev/ttyUSB0

In this case we need to add `ubuntu` to the dialout group

  sudo usermod -aG dialout ubuntu


