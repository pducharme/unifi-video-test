# Docker image for UniFi Video 

UniFi Video is end-of-life and not supported anymore by Ubiquiti. But if you really have to you still can run the latest UniFi Video v3.10.13 using Docker on AMD64-based systems. This image contains a patch for log4j vulnerability [CVE-2021-44228](https://www.cvedetails.com/cve/CVE-2021-44228/).

This is a fork of [pducharme/UniFi-Video-Controller](https://github.com/pducharme/UniFi-Video-Controller).

- Changed base image from `phusion/baseimage:0.11` to `ghcr.io/linuxserver/baseimage-ubuntu:bionic`.
- Changed the mitigation method from disguising log4j v2.17.0 as v2.1.0 to [removing the JndiLookup class from the classpath](https://logging.apache.org/log4j/2.x/security.html#log4j-2-x-mitigation-3) of the included log4j v2.1.0 in UniFi Video.
- Various other small fixes and optimizations.

Grab a copy of the [UniFi Video v3.10.13 Debian installer package](https://dl.ubnt.com/firmwares/ufv/v3.10.13/unifi-video.Ubuntu18.04_amd64.v3.10.13.deb) if you want to archive it. The Dockerfile used to build the image depends on the availability of this package.

## Camera compatibility

The latest supported G3 camera firmware for UniFi Video is v4.23.8. Downgrade the firmware if needed, as v4.30.0 is for Protect and is not compatible with UniFi Video v3.10.13.

- [UVC-G3](https://dl.ui.com/firmwares/uvc/v4.30.0/UVC.S2L_4.30.0.bin)
- [UVC-G3-AF](https://dl.ui.com/firmwares/uvc/v4.30.0/UVC.S2L_4.30.0.bin)
- [UVC-G3-BULLET](https://dl.ui.com/firmwares/uvc/v4.30.0/UVC.S2L_4.30.0.bin)
- [UVC-G3-DOME](https://dl.ui.com/firmwares/uvc/v4.30.0/UVC.S2L_4.30.0.bin)
- [UVC-G3-FLEX](https://dl.ui.com/firmwares/uvc/v4.30.0/UVC.S2L_4.30.0.bin)
- [UVC-G3-MICRO](https://dl.ui.com/firmwares/uvc/v4.23.8/uvc.s2lm.v4.23.8.bin)
- [UVC-G3-PRO](https://dl.ui.com/firmwares/uvc/v4.30.0/UVC.S2L_4.30.0.bin)

## Usage

1. Build the Docker image using the `docker build https://github.com/vuhuy/docker-unifi-video.git -t vuhuy/unifi-video:latest` command. You do not need to clone or checkout the repository.
2. Set your local data and video directories in the `docker run` command or Compose file. 
3. Make sure the ownership and permissions of the attached volumes are set correctly, e.g. owned by the same PUID and PGUID.
4. Start the container and visit http://localhost:7080 to start the UniFi Video wizard.

### Environment variables

| Name | Default | Description |
| ---- | ------- | ----------- |
| DEBUG | `0` | Verbose output from container and UniFi Video. Set to `0` to disable and `1` to enable. |
| USE_UNIFI_TMPFS | `no` | Set to `yes` if you want the UniFi Video application to create a TMPFS memory cache for video recordings. |
| USE_HOST_TMPFS | `no` | Set to `yes` if you want to use a host TMPFS mount as a memory cache for video recordings. |
| PUID | `1000` | The user identifier used to run the daemon. |
| PGID | `1000` | The group identifier used to run the daemon. |
| UMASK | `002` | Umask for writing files. |
| UNIFI_VIDEO_VERSION | `3.10.13` | UniFi Video version to fetch from the Ubiquiti repositories. |
| MONGODB_VERSION | `4.0.28` | MongoDB 4.x version to fetch from the MongoDB repositories. |

UniFi Video needs a cache for storing recordings. Using a disk cache can degrade the expected lifespan and performance of your storage. It is recommended to use a TMPFS memory cache. Docker can mount a TMPFS for you. Set USE_UNIFI_TMPFS to `no`, USE_HOST_TMPFS to `yes` and define a TMPFS mount on `/var/cache/unifi-video` in your Docker run command or Compose file.

### Run example

```
docker run -d \
  --name=unifi-video \
  -p 6666:6666 \
  -p 7080:7080 \
  -p 7442:7442 \
  -p 7443:7443 \
  -p 7445:7445 \
  -p 7446:7446 \
  -p 7447:7447 \
  -v /opt/unifi-video/data:/var/lib/unifi-video \
  -v /srv/unifi-video/videos:/var/lib/unifi-video/videos \
  -e TZ=Europe/Amsterdam \
  -e PUID=1000 \
  -e PGID=1000 \
  -e USE_UNIFI_TMPFS=no \
  -e USE_HOST_TMPFS=yes \
  -e DEBUG=1 \
  --tmpfs /var/cache/unifi-video \
  --cap-add DAC_READ_SEARCH \
  --restart unless-stopped \
  vuhuy/unifi-video:latest
```

### Compose example

```
version: '3'
services:
  unifi-video:
    image: vuhuy/unifi-video:latest
    container_name: unifi-video
    ports:
      - 6666:6666
      - 7080:7080
      - 7442:7442
      - 7443:7443
      - 7445:7445
      - 7446:7446
      - 7447:7447
    volumes:
      - /opt/unifi-video/data:/var/lib/unifi-video
      - /srv/unifi-video/videos:/var/lib/unifi-video/videos
    environment:
      - TZ=Europe/Amsterdam
      - PUID=1000
      - PGID=1000
      - USE_UNIFI_TMPFS=no
      - USE_HOST_TMPFS=yes
      - DEBUG=1
    tmpfs:
      - /var/cache/unifi-video
    cap_add:
      - DAC_READ_SEARCH
    restart: unless-stopped
```

### Expected errors

When starting with a new fresh setup, you are greeted with a few errors. This is not Docker-related and expected behavior of UniFi Video v3.10.13 when there is no configuration present. Everything should work just fine and the errors will disappear once everything is configured.

From the Docker output:

```
2023-03-13 12:32:22,234 ERROR Unable to locate appender ConsoleAppender for logger
Exception in thread "EmsInitTask" java.lang.NullPointerException
 at com.ubnt.airvision.service.ems.C.void.new(Unknown Source)
 at com.ubnt.airvision.service.ems.C$1.run(Unknown Source)
 at java.lang.Thread.run(Thread.java:748)
```

From the UniFi Video logs:

```
1678707143.638 2023-03-13 12:32:23.638/CET: ERROR  [uv.db.svc] Failed to acquire client connection null in MongoDb-Connecting
```

## Troubleshooting

### Upgrading UniFi Video to v3.10.13

UniFi Video has always been annoying about updates. To ensure everything works fine, it is recommended to create a backup and unmanage all cameras in your existing setup. Start with a new clean container for Unifi Video v3.10.13. Import your backup using the setup wizard and make your cameras managed again. Do not forget to restore your recordings if needed.

### Docker for Windows

On Windows, UniFi Video can hang on the "upgrading" screen on startup. You must create a volume (e.g. `docker volume create UnifiVideoDataVolume`) and use that volume for `/var/lib/unifi-video` by changing the `docker run` command or Composer file (e.g. `UnifiVideoDataVolume:/var/lib/unifi-video`). After upgrading, you can change it back to any directory outside the Docker volume (e.g. `D:\Recordings:/var/lib/unifi-video/videos`).