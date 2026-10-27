# Investigation: snapshots and MQTT motion messages stop until reboot

Status: **analysis done, not tested on device, no code changed yet.**
Date: 2026-10-26

## Symptom

After some time the camera stops serving snapshots (web / ONVIF / HA) and
`mqttv4` stops publishing motion messages. Rebooting restores everything.

## Scope reviewed

- `src/static/static/home/yi-hack/script/system.sh`, `service.sh`, `wd.sh`, `thumb.sh`
- `src/www/httpd/cgi-bin/snapshot.sh`
- `src/ipc_cmd/ipc_cmd/ipc_multiplex.c`, `ipc2file.c`
- `src/snapshot/snapshot/imggrabber.c`
- `src/mqtt/mqttv4` submodule (was empty; fetched with
  `git submodule update --init src/mqtt/mqttv4`, commit `48cf94f`)
- wolfMQTT v2.1.0 (version from `src/libwolf_02_mqtt/config.wolfmqtt`),
  downloaded to a scratch dir for review only

Build uses wolfMQTT (not mosquitto): `USE_MOSQUITTO` is not defined.

## Event / data flow (for context)

- `dispatch` runs with `LD_PRELOAD=ipc_multiplex.so`. Every message it
  receives is copied to `/ipc_dispatch_1` ... `/ipc_dispatch_9` (non-blocking,
  max 64 messages each, drops silently with EAGAIN when full).
- `mqttv4` reads `/ipc_dispatch_1` in one IPC thread and runs callbacks there.
- `ipc2file` (ONVIF events) reads `/ipc_dispatch_2`.
- Snapshots are made by `imggrabber`, which reads the H.264/H.265 ring buffer
  `/dev/shm/fshare_frame_buf` written by `rmm`, decodes one keyframe with
  libavcodec and writes a JPEG to stdout.
- Callers of `imggrabber`: `cgi-bin/snapshot.sh` (web, ONVIF snapurl),
  `mqttv4` (motion image), `thumb.sh`, `time_lapse.sh`.
- `wd.sh` watchdog: checks RTSP, `rmm`, `mqttv4` process presence, Wi-Fi.

---

## Primary cause #1: `imggrabber` can hang forever and block everything

**Where:** `src/snapshot/snapshot/imggrabber.c:870-918`

The main read loop is `while (1)` with no timeout. It only exits when an SPS
frame for the requested resolution is found in the current buffer window. It
also `continue`s forever while the header consistency check at `:881` fails.

If the ring buffer stops updating, the loop never ends. Possible triggers:
camera switched off / privacy schedule, `rmm` stalled but alive, or buffer
filling stopping (see the `start_buffer` / `cloud` trick in `system.sh` when
`DISABLE_CLOUD=yes`). The buffer can freeze with no SPS in the window or with
inconsistent header fields.

How the hang spreads:

- **Snapshots stop.** `proc_find()` check at `imggrabber.c:776` makes every
  new `imggrabber` print "Process is already running" and `return 0` with an
  empty body. Web, ONVIF, thumbnails and timelapse all get empty images.
- **MQTT motion stops.** `src/mqtt/mqttv4/src/mqttv4.c:110-112` (and the same
  code in the human / vehicle / animal / baby-crying callbacks) calls
  `system("... imggrabber ...")` synchronously on the single IPC thread. If
  that `imggrabber` hangs, the IPC thread blocks forever, `/ipc_dispatch_1`
  fills, `ipc_multiplex` drops events, no more motion messages.
- **Watchdog blind.** `wd.sh:check_mqtt` only checks that a `mqttv4` process
  exists. Nothing watches `imggrabber`. Only reboot clears it.

Detail: the duplicate check compares `/proc/<pid>/cmdline` first token to
`basename(argv[0])`. The CGI runs it as `imggrabber`, mqttv4 runs it as
`/home/yi-hack/bin/imggrabber` (see `MQTTV4_SNAPSHOT` in
`src/mqtt/mqttv4/include/mqttv4.h:26`). So instances only partially see each
other, which is why the two symptoms can appear separately or together.

## Primary cause #2: mqttv4 reconnect loop can get stuck forever

**Where:** `src/mqtt/mqttv4/src/mqtt_wolf.c:327-354` (`mqtt_connect()`)

`mqtt_connect()` holds `wolf_mtx` and loops `MqttClient_NetConnect()` +
`MqttClient_Connect()` until success. In wolfMQTT 2.1.0
(`src/mqtt_socket.c:412-424`), `MqttSocket_Connect` skips opening a new socket
when the `MQTT_CLIENT_FLAG_IS_CONNECTED` flag is set, and that flag is cleared
only in `MqttSocket_Disconnect` (`mqtt_socket.c:596`).

So if TCP connects but CONNECT/CONNACK fails (broker restart, broker not ready,
auth rejected, Wi-Fi blip mid-handshake), every retry reuses the dead socket
and never succeeds. Process stays alive, no MQTT at all, `wd.sh` sees it
running. Explains MQTT stopping on its own (snapshots unaffected).

---

## Secondary issues

| # | Location | Issue |
|---|----------|-------|
| S1 | `imggrabber.c:231` | `cb_move` wraps with `>` instead of `>=`. A frame ending exactly at buffer end is not wrapped; comparison with `buf_idx_end` (normalised with `>=` at `:878`) is missed and the walk overruns into stale data. |
| S2 | `imggrabber.c:862` | `(frame_header_size < 0) && (frame_header_size > 40)` can never be true. Autodetect loop at `:853` has no exit / timeout; can spin forever on models with header size 0 (r35gb, r37gb). |
| S3 | `imggrabber.c` (`USE_SEMAPHORE` off) | No locking on the frame buffer; reads race with `rmm` writes, giving garbage lengths and decode failures. |
| S4 | `mqtt_wolf.c:80` | `send()` has no `SO_SNDTIMEO`. On a half-open TCP connection an image publish can block for minutes while holding `wolf_mtx`. |
| S5 | `ipc_multiplex.c:80-97` | When `original_mq_receive` returns -1, it still forwards with length `(size_t)-1` (EMSGSIZE, stderr spam) and overwrites `errno` that `dispatch` may read. |
| S6 | `mqttv4.c` `handle_config` | `malloc((char)strlen(value)+1)` truncates length to 8 bits; a config value of 256+ chars overflows the heap. |
| S7 | `wd.sh:400` | Watchdog exits when `RTSP=no`, so `mqttv4` and `rmm` are not watched at all then. |
| S8 | `ipc2file.c` `ipc_init` | `ipc_callbacks` malloc'd but not zeroed; calling an unset callback type would jump to garbage (currently only set types are dispatched). |

---

## Fix proposals

1. **Bound `imggrabber` loops** (`imggrabber.c`): add a deadline (e.g. 5 s via
   `clock_gettime(CLOCK_MONOTONIC)`) to the loops at `:853` and `:870`; on
   timeout unmap and exit non-zero. Fix `cb_move` (`>` → `>=`) and the
   condition at `:862`.
2. **Timeout in every caller**: busybox `timeout -s KILL 10 imggrabber ...` in
   `MQTTV4_SNAPSHOT` (`mqttv4.h:26`), `cgi-bin/snapshot.sh`, `thumb.sh`,
   `time_lapse.sh`.
3. **Fix mqttv4 reconnect** (`mqtt_wolf.c`): in the `do/while`, call
   `MqttClient_NetDisconnect(&wolf_client)` before `continue` after a failed
   `MqttClient_Connect`; release the mutex between retries; set `SO_SNDTIMEO`
   (and connect timeout) in `net_connect`.
4. **Move snapshots off the IPC thread** (`mqttv4.c`): run the image capture +
   publish in a detached worker thread with a single "busy" flag, so a slow
   snapshot can't block motion events. Also deduplicate the 5 copy-pasted
   callbacks.
5. **Harden `wd.sh`**: kill any `imggrabber` older than ~30 s; add mqttv4
   liveness (e.g. heartbeat file touched in main loop, restart if stale); run
   mqtt / rmm checks even when RTSP is disabled.
6. **Guard `ipc_multiplex`**: return early when `bytes_read < 0`, save and
   restore `errno` around the `mq_send` calls.
7. Minor: fix `(char)` casts in `handle_config` (S6), zero `ipc_callbacks`
   with `calloc` (S8).

Priority: 1 + 2 should alone stop the reboot-only failure; 3 next.

Note: mqttv4 is an upstream submodule (`roleoroleo/mqttv4`). Its fixes need a
fork/PR upstream, or a patch file applied from `src/mqtt/init.mqtt` (that
script runs `git reset --hard` in the submodule, so a patch step would go
after it).

---

## Questions to confirm (pending, ask user / check on device)

1. When failing, output of `ps | grep -e imggrabber -e mqttv4` over SSH? Any
   long-lived `imggrabber` confirms cause #1.
2. Does live RTSP still work at that moment? If not, frame buffer has stalled.
3. Is camera switch-off / privacy schedule used? Values of `DISABLE_CLOUD` and
   `REC_WITHOUT_CLOUD`?
4. Is `TOPIC_MOTION_IMAGE` set in `mqttv4.conf`? Does the broker status topic
   still show `online` during failure? Does failure coincide with broker
   restarts?
5. Camera model (`/home/yi-hack/model_suffix`)?
6. Go ahead implementing fixes? How to handle the mqttv4 submodule (fork vs
   patch file)?

## Useful on-device diagnostics

```sh
ps | grep -e imggrabber -e mqttv4 -e rmm
cat /proc/$(pidof mqttv4)/status | grep -e State -e Threads
ls -l /proc/$(pidof mqttv4)/fd | wc -l
hexdump -n 32 /dev/shm/fshare_frame_buf   # run twice; unchanged = buffer stalled
cat /home/yi-hack/etc/camera.conf | grep SWITCH_ON
free; df -h /tmp
```
### On-Device Diagnostics

Data collected from a camera that was in the "snapshots and MQTT motion messages stop until reboot" state.

For `ps | grep -e imggrabber -e mqttv4 -e rmm` I got:
```ini
 1437 root     53492 S    ./rmm
 1702 root      1440 S    mqttv4
 3068 root      4204 S    imggrabber -m r35gb -r low -w
18318 root       972 S    {grep} /tmp/sd/yi-hack/bin/busybox /tmp/sd/yi-hack/bin/grep -e imggrabber -e mqttv4 -e rmm
```

For `cat /proc/$(pidof mqttv4)/status | grep -e State -e Threads` I got:

```ini
State:	S (sleeping)
Threads:	2
```

For `ls -l /proc/$(pidof mqttv4)/fd` , with no `wc -l` because there is no wc command, I got:
```ini
lr-x------    1 root     root            64 Oct 26 14:11 0 -> /dev/null
l-wx------    1 root     root            64 Oct 26 14:11 1 -> /dev/null
lrwx------    1 root     root            64 Oct 26 14:11 2 -> /dev/null
lrwx------    1 root     root            64 Oct 26 14:11 3 -> socket:[2296]
lrwx------    1 root     root            64 Oct 26 14:11 4 -> /ipc_dispatch_1
```ini

For `hexdump -n 32 /dev/shm/fshare_frame_buf` , run twice, I got:

```ini
root@YiDome-Terraza-Principal:~# hexdump -n 32 /dev/shm/fshare_frame_buf
0000000 0000 0000 3fb0 001b ece2 0002 7523 0019
0000010 7573 0019 1f61 788e 97f8 001b 0000 0000
0000020

root@YiDome-Terraza-Principal:~# hexdump -n 32 /dev/shm/fshare_frame_buf
0000000 0000 0000 3162 0019 ece2 0002 0c35 0002
0000010 1ad3 0004 691e 788e 9c76 001b 0000 0000
0000020
```
so, the buffer is not stalled, it is changing between the two reads.

For `cat /tmp/sd/yi-hack/etc/camera.conf | grep SWITCH_ON` I got:
```ini
SWITCH_ON=yes
```

For `free; df -h /tmp` I got:
```ini
              total        used        free      shared  buff/cache   available
Mem:          60676       45232        8704        1980        6740       12436
Swap:         65532        1088       64444
```
