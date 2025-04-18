# Share your Raspberry Debian 64-bit physical desktop remotely with x11vnc

&nbsp;

Read this and other Raspberry Pi guides here
* [How to get Argon One v2 fan control working on Raspberry 4B and Debian 64](https://nemozny.github.io/argonone-debian-64/)
* [Running Interactive Brokers gateway on Raspberry 4B and Debian 64-bit](https://nemozny.github.io/ibgateway-raspberry-64/)
* [Share your Raspberry Debian 64-bit physical desktop remotely with x11vnc](https://nemozny.github.io/vnc-share-physical-monitor/)

&nbsp;

### Intro
I wanted to replace my hosted virtual server, that I was paying monthly fees for, with Raspberry Pi 4B / 8GB with Debian 64-bit OS. Originally I ran the server in a reliable setup with xrdp + xvbf + x11vnc that I used for several years without issues. The downside of this setup is that you have nothing to show on your physical desktop, since xrdp creates a virtual desktop unrelated to your physical screen. Linux xrdp is a different beast than Windows RDP. 
Also the virtual framebuffer is not very convenient to use, since it is quite cumbersome to resize windows.

My goal was to 
1. run the X server on a monitor connected to RPI via HDMI, with a lightweight window manager, while simulaneously sharing the same screen via VNC. Not RDP, mind you!
Or in other words - **I wanted to move the mouse cursor on my physical monitor using a remote connection.**
2. I prefer to to run all services only on localhost, and connect to them through SSH forwarding, so you don't have to bother with any passwords other than SSH. You need to make sure **all services were listening only on localhost!**
3. Avoid using NoMachine and other resource hungry tools.

I will describe a few options here, but really only #2 and #3 allow for sharing your physical screen.
1. **Virtual session (xrdp + xvfb + x11vnc)**
2. **One session sharing with login screen (xdm + x11vnc)**
3. **One session sharing with autologin (slim + x11vnc)**
Please see the appropriate sections below.

&nbsp;

### 1. Virtual session (xrdp + xvfb + x11vnc)
The original setup that will provide only virtual desktop, not physical. In case it might come useful to someone.

```
# apt-get install xrdp xvfb x11vnc
```
&nbsp;

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

&nbsp;

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

&nbsp;

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
&nbsp;

#### Enable services
Now reload systemd and enable both services to run on a system boot.

```
# systemctl daemon-reload
# systemctl enable xrdp
# systemctl enable xvfb
# systemctl enable x11vnc
```
&nbsp;

#### Set up RDP through an SSH tunnel
You can use putty, kitty or whichever SSH client you like. You need to set up port forwarding from remote 3389 to any local port (1234) and use this local port 1234 in the Remote Desktop - connection 127.0.0.1:1234
&nbsp;

Or you can use Mobaxterm, which is an all-in-one package.
&nbsp;
* Mobaxterm - RDP
* Remote host: 127.0.0.1, Port: 3389
* Network settings - SSH Gateway - enter your SSH credentials.

&nbsp;

### 2. One session sharing with login screen (xdm + x11vnc)
You will need to login to XDM every time, either remotely or locally, before you can start any apps with GUI.

Take your pick of a window manager. I went with Fluxbox, since it needed to install the least number of packages of all I have tried.

#### Install window manager
```
# apt-get install fluxbox
```
&nbsp;

#### Install display manager
To be able to login on a physical screen. Either using xdm or whatever else you like.
```
# apt-get install xdm
```
&nbsp;

#### Stop xrdp and xvfb (if you installed it previously)
xvfb is running its own X server, so you need to stop and disable it.

```
# systemctl stop xrdp
# systemctl disable xrdp
# systemctl disable xvfb
```
&nbsp;

#### Create (or edit) x11vnc.service
Create this file in /etc/systemd/system/x11vnc.service
It has to be linked to XDM, not xrdp this time.

File contents:
```
[Unit]
Description=VNC Server for X11
Requires=xdm.service
After=xdm.service

[Service]
Type=simple
ExecStart=/usr/bin/x11vnc -ncache 10 -ncache_cr -display :0 -geometry 1400x800 -noxrecord -noxdamage -forever -shared -o /var/log/x11vnc.log -noipv6 -localhost -loop -nopw -auth guess
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```
&nbsp;

#### Enable services
Now reload systemd and enable services to run on a system boot.

```
# systemctl daemon-reload
# systemctl enable xdm
# systemctl enable x11vnc
```

#### Setup VNC client
In Mobaxterm create session VNC:
* host 127.0.0.1 port 5900
* SSH Gateway - enter your SSH credentials

&nbsp;

#### Reboot
And then either:

1. Login to XDM on your physical screen and then open your Mobaxterm VNC session and connect to the same session remotely.
2. Open your Mobaxterm VNC session, it should show XDM. Login and you will see the same sesion on your physical screen.

Done.

&nbsp;

### 3. Login to physical screen session
Follow [this Raspberry guide](https://www.raspberrypi.com/documentation/computers/remote-access.html#vnc).
TigerVNC is no longer maintained on Windows and TightVNC did not work for me, neither did Mobaxterm VNC. Try and use UltraVNC viewer.

&nbsp;

### Firewall
Optionally check ports your services are running on
```
# netstat -atupn
```
and block all ports on firewall except for the SSH server.

Where

* 0.0.0.0:22

and

* :::22

is SSHd listening on all interfaces and 

* 127.0.0.1:5900

and 

* ::1:5900

is x11vnc listening only on localhost.


All services, except SSH, in your Debian were bound exclusively to the localhost, so if you had other computer sitting on your network right next to your Debian, it will not be able to connect to anything, except SSH.

Then you can login from your client (let's say Windows) and use SSH port forwarding to basically map your Debian services (ports) to your client Windows ports. So ultimately your running VNC service 127.0.0.1:5900 in Debian will get port forwarded/mapped to your Windows client 127.0.0.1:1234 (you can choose any port on the client). Your Windows VNC client would then open 127.0.0.1:1234 and find that it actually opened the screen in Debian.

For added security you can also add an extra SSH jumpbox.
