# Bluez in docker

/ref https://twitter.com/Hertz_G/status/1310306592534016003


# Result so far - 12:25 am (+1 day)

@ndessart I ended up using the nsenter (very reluctantly) as it seems like bluetooth hci is global on the system and as you mentioned is not supported in network namespaces. I ended up with the following in the end.

**Dockerfile**:
```Dockerfile
FROM alpine:latest
RUN apk add --no-cache bluez

COPY ./entrypoint.sh /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
```

**./entrypoint.sh**:
```bash
#!/bin/sh
set -e

/usr/bin/dbus-daemon --system --nofork --print-address --nopidfile --nosyslog &

sleep 1
nsenter --net=/rootns/net -- /usr/lib/bluetooth/bluetoothd --debug --nodetach
```

**Run command**:
```bash
docker run --rm -it --mount type=bind,source=/proc/1/ns/,target=/rootns --device /dev/ttyAMA0  --cap-add SYS_PTRACE --cap-add SYS_ADMIN --cap-add NET_ADMIN hertzg/blueztt
```

This is one **big security hole** (with `SYS_ADMIN` cap and `/proc/1/ns/` bind mount) but seems like the only option other than `--net host` so far :( .
