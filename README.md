# Share your Raspberry Debian 64-bit physical desktop remotely with x11vnc

### Intro
I was about to replace my virtual server, I was paying monthly fees for, with Raspberry Pi 4B / 8GB with Debian 64-bit OS. Originally I ran the server in a reliable setup with XRDP + xvbf + x11vnc. The downside of this setup is that you have nothing to show on your physical desktop, since XRDP creates a virtual desktop unrelated to your physical screen. Linux XRDP is a different mechanism than Windows RDP. And the xvbf/virtual framebuffer is not very convenient to use.

The goal was to run X server on a monitor connected to RPI, with a lightweight window manager, while simulaneously sharing the same screen via VNC. Via VNC, not via RDP!
Or in other words - **I wanted to move the mouse cursor on my physical monitor using a remote connection.**

The idea is to run everything through SSH, on a localhost, so **no ports should be exposed remotely.**

1. Virtual session (xrdp + xvfb + x11vnc)
2. One session sharing

### Virtual session (xrdp + xvfb + x11vnc)
The original setup.

```
# apt-get install xrdp xvfb x11vnc
```

#### Edit xrdp.ini
Location - /etc/xrdp/xrdp.ini

In the section *Session types* and a new, first entry:
```
[xrdp]
name=Remote Desktop
lib=libvnc.so
username=ask
password=ask
ip=127.0.0.1
port=5900 
```
You can call it whatever you like. The important thing is the port and bind to localhost.

#### Create xvfb.service
Create this file anywhere you like (I suggest some Git repo), and link it to /etc/systemd/system/xvfb.service
File contents:
```
[Unit]
Description=X virtual framebuffer
Requires=xrdp.service
After=xrdp.service
BindsTo=xrdp.service
PartOf=xrdp.service

[Service]
Type=simple
ExecStart=/usr/bin/Xvfb :0 -ac -screen 0 1920x1080x24
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```
Link it like this
```
# ln -s /home/myownself/xvfb.service /etc/systemd/system/xvfb.service
```

#### Create x11vnc.service
File contents:
```
[Unit]
Description=VNC Server for X11
Requires=xvfb.service
After=xvfb.service
BindsTo=xrdp.service
PartOf=xrdp.service

[Service]
Type=simple
ExecStart=/usr/bin/x11vnc -ncache 10 -ncache_cr -display :0 -clip 1920x1080+0+0 -forever -shared -o /var/log/x11vnc.log -noipv6 -localhost -loop -nopw
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```
Link it the same way
```
# ln -s /home/myownself/x11vnc.service /etc/systemd/system/x11vnc.service
```

#### Enable services
Now reload systemd and enable both services to run on a system boot.

```
# systemctl daemon-reload
# systemctl enable xrdp
# systemctl enable xvfb
# systemctl enable x11vnc
```


#### Set up RDP through an SSH tunnel
You can use putty, kitty or whichever SSH client you like. You need to set up port forwarding from remote 3389 to any local port (1234) and use this local port 1234 in the Remote Desktop - connection 127.0.0.1:1234

Or you can use Mobaxterm, which is an all-in-one package.

Mobaxterm - RDP

Remote host: 127.0.0.1, Port: 3389

Network settings - SSH Gateway - enter your SSH credentials.


### One session sharing
Take your pick of a window manager. I went with Fluxbox, since it needed to install the least number of packages of all I have tried.

#### Install window manager
```
# apt-get install fluxbox
```

#### Install login manager
To be able to login on a physical screen.
```
# apt-get install xdm
```

#### Stop xrdp and xvfb (if you installed it previously)
xvfb is running its own X server, so you need to stop and disable it.

```
# systemctl stop xrdp
# systemctl disable xrdp
# systemctl disable xvfb
```

#### Create (or edit) xvfb.service
Create this file in /etc/systemd/system/xvfb.service

File contents:
```
[Unit]
Description=X virtual framebuffer
Requires=xdm.service
After=xdm.service

[Service]
Type=simple
ExecStart=/usr/bin/x11vnc -ncache 10 -ncache_cr -display :0 -noxrecord -noxdamage -forever -shared -o /var/log/x11vnc.log -noipv6 -localhost -loop -nopw -auth guess
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

#### Enable services
Now reload systemd and enable services to run on a system boot.

```
# systemctl daemon-reload
# systemctl enable xdm
# systemctl enable x11vnc
```

#### Setup VNC client
In Mobaxterm create session VNC:

host 127.0.0.1 port 5900

SSH Gateway - enter your SSH credentials


#### Reboot
Either:

1. Login to XDM on your physical screen and then open your Mobaxterm VNC session and connect to the same session remotely.
2. Open your Mobaxterm VNC session, it should show XDM. Login and you will see the same sesion on your physical screen.


