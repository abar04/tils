# Hot-Swapping Drives with Docker

> :warning: **Warning:** The following was undertaken with security of the container being a low priority

## Table of Contents

1. [References](#references)
2. [Overview](#overview)
3. [Docker Run-time Configuration](#configuration)
4. [Drives when container starts](#starting)
5. [Drives when container is running](#running)

## <a name="references"></a> References

- [How can I add a volume to an existing Docker container?](https://stackoverflow.com/questions/28302178/how-can-i-add-a-volume-to-an-existing-docker-container)
- [Not recommend but early method](https://jpetazzo.github.io/2015/01/13/docker-mount-dynamic-volumes/)
- [More readable of above](https://medium.com/kokster/mount-volumes-into-a-running-container-65a967bee3b5)
- [Stack Overflow Overview](https://stackoverflow.com/questions/24225647/docker-a-way-to-give-access-to-a-host-usb-or-serial-device)
- [Details of udev rules](http://marc.merlins.org/perso/linux/post*2018-12-20_Accessing-USB-Devices-In-Docker-_ttyUSB0*-dev-bus-usb-_-for-fastboot_-adb_-without-using-privileged.html)
- [Docker reference of device-cgroup-rule](https://docs.docker.com/engine/reference/commandline/create/#dealing-with-dynamically-created-devices---device-cgroup-rule)

## <a name="overview"></a> Overview

Controlling devices such as hard-drives from within a docker container is a tricky. Trying to support both devices present when the container starts and drives plugged into the host after the container starts is trickier. The following is a solution to setting up a system to allow for drive management from within a docker container.

## <a name="configuration"></a> Docker Run-Time Configuration

The run-time operations required to allow a container to manage drives can be encapsulated by the following [docker-compose file](docker-compose.yml):

```yaml
version: "2.3"
services:
  backend:
    image: ubuntu:latest
    privileged: true
    cap_add:
      - SYS_ADMIN
    device_cgroup_rules:
      - "c 8:* rmw"
    volumes:
      - /opt/data:/opt/data
      - /etc/fstab:/etc/fstab
```

| Option                | Value                   | Rationale                                                   |
| --------------------- | ----------------------- | ----------------------------------------------------------- |
| `privileged`          | `true`                  | Security in this instance is not a large concern            |
| `cap_add`             | `SYS_ADMIN`             | Allows adding new drives                                    |
| `device_cgroup_rules` | `c 8:* rmw`             | Only want drives with major number 8 to be passed through   |
| `volumes`             | `/etc/fstab:/etc/fstab` | Want to be able to make lasting changes to the host system. |
|                       | `/opt/data:/opt/data`   | Common data directory between host and container            |

## <a name="starting"></a> Drives when container starts

The drives present when the container starts are taken care of by the `privileged` flag and get mapped to `/dev/sd*` as expected.

## <a name="running"></a> Drives added after container starts

The device_cgroup_rule `c 8:* rmw` ensures that any new drives with major number 8 are allowed to be accessed by the container.

This does not mean they automatically come in `/dev` though. To add drives to `/dev` where we normally interact with them we need to use `mknod`.

The chosen routine would use `lsblk` and `blkid` to discover all required information about the drive to populate an entry into `/dev`.
The normal mount commands can then be used to do drive manipulation.

[jc](https://github.com/kellyjonbrazil/jc) is a fantastic parser for CLI tools.
