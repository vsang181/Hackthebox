# Remote Desktop Protocols in Linux

Remote desktop protocols allow you to **interact with a system through a graphical interface from another machine**. While command-line access is often enough, there are situations where a graphical session is required—for administration, troubleshooting, or interacting with GUI-based applications. Understanding how these protocols work is important both for system management and for identifying security weaknesses during assessments 

When you connect remotely, you choose the protocol based on the operating system and the type of access you need.

---

## Common Remote Desktop Protocols

### Remote Desktop Protocol (RDP)

RDP is primarily associated with Windows systems. It allows you to connect to a Windows host and interact with its desktop as if you were physically present.

While RDP is less common on Linux, you may still encounter it in mixed environments or on Linux systems configured to provide RDP access for compatibility reasons.

---

### Virtual Network Computing (VNC)

VNC is widely used in Linux environments and is also cross-platform. It provides remote access to a graphical desktop by transmitting screen updates and input events over the network.

You will encounter VNC frequently because:

* It works across operating systems
* It is simple to deploy
* It supports both shared desktops and virtual sessions

From a penetration testing perspective, exposed or weakly protected VNC services are common findings.

---

## X Server (X11)

The **X Server** is a core component of the X Window System (X11), which provides graphical capabilities on Unix-like systems. It acts as the intermediary between applications and the display hardware.

When a Linux desktop starts, the graphical interface communicates with the operating system through the X server. Even on a local machine, this communication uses networking concepts internally.

A key feature of X11 is **network transparency**. Applications do not have to run on the same machine as the display. An application can run on a remote system while being rendered locally.

### X11 Communication Details

* Transport: TCP/IP or Unix sockets
* Common ports: TCP **6000–6009**
* Display `:0` typically maps to TCP port `6000`

This design allows powerful flexibility but also introduces security concerns.

---

## X11 Forwarding over SSH

By default, X11 traffic is **unencrypted**, which makes it unsafe on untrusted networks. To mitigate this, X11 is commonly tunneled through SSH.

To allow X11 forwarding, the SSH server must permit it.

### Checking X11 Forwarding Configuration

```text
cat /etc/ssh/sshd_config | grep X11Forwarding
```

Expected output:

```text
X11Forwarding yes
```

Once enabled, you can launch graphical applications remotely while rendering them locally.

### Example: Running Firefox via SSH with X11 Forwarding

```text
ssh -X htb-student@10.129.23.11 /usr/bin/firefox
```

Here, the application runs on the remote host, but the window appears on your local machine. This approach reduces load on the remote system and avoids sending raw screen images over the network.

---

## X11 Security Considerations

X11 is **not secure by design**. If exposed without encryption or access controls, it can leak sensitive information.

Key risks include:

* Unencrypted keystrokes and screen data
* Window content capture by other users on the network
* Abuse of X11 utilities such as `xwd` (window dumps)

Historically, X11 has also suffered from serious vulnerabilities. In 2017, multiple flaws in the Xorg server (CVE-2017-2624, CVE-2017-2625, CVE-2017-2626) allowed attackers to execute code in another user’s session under certain conditions.

When assessing Linux systems, always pay attention to **open ports in the 6000–6010 range** and investigate whether X11 services are exposed.

---

## X Display Manager Control Protocol (XDMCP)

**XDMCP** is used by display managers to provide remote graphical login sessions. It communicates over **UDP port 177** and can redirect a full desktop environment such as GNOME or KDE to a remote client.

While functional, XDMCP is **inherently insecure**:

* Traffic is unencrypted
* Susceptible to man-in-the-middle attacks
* Can expose full desktop sessions

Because of these risks, XDMCP should not be used in secure environments. From a penetration testing perspective, exposed XDMCP services are high-value targets.

---

## Virtual Network Computing (VNC) in Practice

VNC uses the **RFB (Remote Framebuffer)** protocol to transmit graphical output. Unlike X11 forwarding, VNC renders the desktop on the remote machine and sends screen updates to the client.

VNC is commonly used for:

* Remote system administration
* Accessing headless servers
* User support and troubleshooting

### VNC Server Models

You will encounter two main types of VNC servers:

1. **Shared desktop servers**

   * Expose the actual screen of the host
   * Local keyboard and mouse remain active

2. **Virtual session servers**

   * Provide independent desktop sessions
   * Similar to terminal server concepts

---

## VNC Ports and Displays

By default:

* Display `:0` → TCP port **5900**
* Display `:1` → TCP port **5901**
* Display `:2` → TCP port **5902**

Each additional session increments the port number.

---

## Common VNC Implementations

You will see several VNC implementations in the wild:

* TigerVNC
* TightVNC
* RealVNC
* UltraVNC

RealVNC and UltraVNC are commonly preferred due to better encryption and security features.

---

## Setting Up TigerVNC (Example)

In this example, you set up a TigerVNC server with the XFCE desktop environment.

### Installing Required Packages

```text
sudo apt install xfce4 xfce4-goodies tigervnc-standalone-server -y
```

### Creating a VNC Password

```text
vncpasswd
```

This creates a hidden `.vnc` directory in your home folder.

---

## Configuring the VNC Session

Create the required configuration files:

```text
touch ~/.vnc/xstartup ~/.vnc/config
```

### Example `xstartup`

```text
#!/bin/bash
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
/usr/bin/startxfce4
[ -x /etc/vnc/xstartup ] && exec /etc/vnc/xstartup
[ -r $HOME/.Xresources ] && xrdb $HOME/.Xresources
x-window-manager &
```

### Example `config`

```text
geometry=1920x1080
dpi=96
```

Make the startup script executable:

```text
chmod +x ~/.vnc/xstartup
```

---

## Starting and Listing VNC Sessions

Start the VNC server:

```text
vncserver
```

List active sessions:

```text
vncserver -list
```

Example output:

```text
X DISPLAY #     RFB PORT #      PROCESS ID
:1              5901            79746
```

---

## Securing VNC with SSH Tunnelling

VNC traffic should **never** be exposed directly to untrusted networks. A common mitigation is to tunnel VNC through SSH.

### Creating an SSH Tunnel

```text
ssh -L 5901:127.0.0.1:5901 -N -f -l htb-student 10.129.14.130
```

This forwards local port `5901` to the remote VNC server securely.

### Connecting Through the Tunnel

```text
xtightvncviewer localhost:5901
```

All VNC traffic now travels inside the encrypted SSH tunnel.

---

## Why This Matters

Remote desktop protocols are powerful, but they also introduce significant risk when misconfigured.

As you continue learning, always ask yourself:

* Is the protocol encrypted?
* Is authentication strong?
* Is the service exposed unnecessarily?

Understanding how graphical remote access works gives you both administrative confidence and a strong advantage during security assessments.
