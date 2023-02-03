# VPS-stream-relay-server
VPS Stream relay server SRT/SRTLA


# About

Collection of notes to build a cheap ARM vps on AWS or similar as a SRT/SRTLA -> RTMP/HLS relay including FFMPEG disconnect protection.

Can be run on low and cheapest VPS on a site near your location and do audio gain adjustment or even re-encode if you upgrade to a more powerful server for HEVC.

Not step-by-step, but adapted notes that will save you time and get you mostly there.

Uses systemd combined with script-timeout to handle reset and auto-end stream.
Default 3 sec SRT buffer, stream will end after 30 sec with no input.
5 second restart delay for processes

Suggested AWS:
```
For remux/H264: t4g.nano
If you wish to re-encode on server and use HEVC with Twitch:
For 1080@30 hevc: t4g.2xlarge 
For 1440@30 hevc: c6g.4xlarge
```

# Requirements
```bash
sudo apt-get install srt-tools ffmpeg
```

- Add to these files to /home/ubuntu or adjust script for your own location

# Relay.sh
```bash
#!/bin/bash
srt-live-transmit "srt://:8989?passphrase=privatepassword&transtype=live&latency=2000&fc=102400&rcvbuf=48234496" "udp://:9191" -tm:1 -t:30
```

# srt.sh

Example with direct feed to Twitch
- Configure srt.sh with your streaming key

```bash
#!/bin/bash
ffmpeg -hide_banner -i "udp://:3001?timeout=30000000&fifo_size=1000000" -c copy "volume=16dB" -f flv "rtmp://TWITCH_with_/KEY"
```

# Make excecutable
```bash
sudo chmod u+x srt.sh
sudo chmod u+x relay.sh
```

- Add the following files to /etc/systemd/system/

srt.service

relay.service

# relay.service
```bash
[Unit]
Description=srt

[Service]
ExecStart=/home/ubuntu/srt.sh
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

# srt.service
```bash
[Unit]
Description=srt

[Service]
ExecStart=/home/ubuntu/srt.sh
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```
# Start the automated process
```bash
sudo systemctr start relay
sudo systemctr enable relay
sudo systemctr enable srt
sudo systemctr start srt
```

# srtla.sh

If you want to add SRTLA, add another script that pipes srtla into your modified relay script with something like:
```bash
#!/bin/bash
/home/ubuntu/srtla/srtla_rec 6666 127.0.0.1 7777
```
Modify relay.sh to accept connect with patched SRT to convert to SRT.
```bash
/home/ubuntu/srt/srt-live-transmit -st:yes "srt://127.0.0.1:7777?mode=listener&lossmaxttl=40&latency=2000" "udp://:9191" -tm:1 -t:30
```

## Build Instructions for patched SRT required for SRTLA 
Fork of the patched belabox srt + buildfix from tt2468
```bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install tclsh pkg-config cmake libssl-dev build-essential
git clone https://github.com/thetobby/srt.git
cd srt
./configure
make -j12
```

# status.sh
If you want to monitor the status of your server live in SSH.
Needs modification, but inspiration.

```bash
#!/bin/bash
#sudo journalctl --rotate
#sudo journalctl --vacuum-time=3600s
for (( ; ; ))
do
clear

disk_use=`df -P | grep /dev | grep -v -E '(tmp|boot)' | awk '{print $5}' | cut -f 1 -d "%"`
echo "Disk usage : $disk_use %"


E=`systemctl is-active relay`
echo "Relay: $E"
journalctl -e -u relay  | tail -n 4 | cut -c 44-
echo ""
H=`systemctl is-active srt`
echo "FFmpeg: $H"
journalctl -e -u srt | tail -n 6 | cut -c 44-

sleep 3
done
```



