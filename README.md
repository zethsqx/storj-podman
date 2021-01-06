# storj-podman

## Overview
Storj node uses docker as the container engine. There is some difference for updating the storj node when using docker vs podman

1. Setup storj  
https://documentation.storj.io/setup/cli/storage-node
2. Configure auto update  
https://documentation.storj.io/setup/cli/software-updates

> ** When in doubt, just follow official docs and use docker**  
> ** Deploying production storj node differently from the official docs might risk it being disqualified or suspended**
>> Possible problem includes:
>> a. consistent fail update of the storj node version  
>> b. update crashes the node  
>> c. container mounting created problem with disk and data becomes corrupted  

#### Docker
- recommended by storj to use their watchtower
- uses watch tower to update storj node container 
- underlying docker is its daemon

#### Podman
- uses podman built in auto update services
- container is manage as systemd unit
- podman auto update requires fully-qualified image name(FQIN) (eg. docker.io/storj/storagenode:latest)

## Steps for Deployment

1. Update podman 
```
yum update podman -y
```

2. Set --label "io.containers.autoupdate=image"
4. Run container with fully-qualified image name (FQIN). If it is running, stop and rerun.
```
podman run --privileged -d \
--label "io.containers.autoupdate=image" \
--restart unless-stopped \
--stop-timeout 300 \
-p 28967:28967 -p 14002:14002 \
-e WALLET=<redacted> \
-e EMAIL=<example@email.com> \
-e ADDRESS=<ddns:port> \
-e STORAGE="838GB" \
--mount type=bind,source="/root/.local/share/storj/identity/storagenode",destination=/app/identity \
--mount type=bind,source="/mnt/storj",destination=/app/config \
--name storagenode \
docker.io/storjlabs/storagenode:latest
```

5. Generate user systemd to the userfolder and softlink it to system level
6. Reload, enable and start the service (the running container will restart)
```
cd /etc/systemd/user
podman generate systemd --new --files --name storagenode
ln -s /etc/systemd/user/container-storagenode.service /etc/systemd/system/container-storagenode.service
systemctl daemon-reload
systemctl enable container-storagenode.service
systemctl start container-storagenode.service
```

7. Generate user systemd to the userfolder and softlink it to system level
8. If required, change podman-auto-update to desired (eg. OnCalendar=*-*-* 14:27:00) 
```
cat /usr/lib/systemd/system/podman-auto-update.timer
systemctl daemon-reload
systemctl enable podman-auto-update.timer
systemctl start podman-auto-update.timer
systemctl status podman-auto-update.timer
```

Useful links
- https://www.redhat.com/sysadmin/improved-systemd-podman
- http://docs.podman.io/en/latest/markdown/podman-generate-systemd.1.html
- https://github.com/containers/podman/blob/v2.0/docs/source/markdown/podman-auto-update.1.md

## Testing podman auto update (using storjlabs/watchtower)
```
podman pull storjlabs/watchtower:i386-0.3.8-patch1
podman image tag storjlabs/watchtower:i386-0.3.8-patch1 docker.io/storjlabs/watchtower:latest
podman rmi docker.io/storjlabs/watchtower:i386-0.3.8-patch1
podman run --privileged -d --label "io.containers.autoupdate=image" --restart=always --name watchtower -v /var/run/podman/podman.sock:/var/run/docker.sock docker.io/storjlabs/watchtower --stop-timeout 300s

podman auto-update

podman stop watchtower
podman rm watchtower
```

