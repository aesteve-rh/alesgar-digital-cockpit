---
title: "Wayland in the QM Container"
date: 2024-07-19T09:33:24+02:00
draft: false
---

Lately we have been working, with Roberto Majadas, in designing a
scenario capable of running a [Wayland](https://wayland.freedesktop.org/)
compositor in the [QM Container](https://github.com/containers/qm).
The required bits to support this scenario in the QM have been recently
upstreamed, and will be available in the next release (>=0.6.6). This post
explains the process and all the configuration required to make it work.

## Creating a new session

First of all, a Wayland compositor requires an active session and a
seat (i.e., `seat0`). The seat contains devices (e.g., `/dev/dri/card0`) that
the compositor will need to access. Furthermore, the session needs a
TTY associated (typically, TTY7, since it provides a graphical environmnet).

Since we want the session to start automatically when the QM container
starts, we will employ systemd to create the session.

- Create a new service file at `/etc/systemd/system/wayland-session.service`:

```ini
[Unit]
Description=Wayland Session Creation Handling
After=systemd-user-sessions.service

[Service]
Type=simple
Environment=XDG_SESSION_TYPE=wayland
UnsetEnvironment=TERM
ExecStart=/bin/sleep infinity
Restart=no

# Run the session as root (required by PAMName)
User=0
Group=0

# Set up a full user session for the user, required by Wayland.
PAMName=login

# Fail to start if not controlling the tty.
StandardInput=tty-fail

# Defaults to journal.
StandardError=journal
StandardOutput=journal

# A virtual terminal is needed.
TTYPath=/dev/tty7
TTYReset=yes
TTYVHangup=yes
TTYVTDisallocate=yes

# Log this user with utmp.
UtmpIdentifier=tty7
UtmpMode=user

[Install]
WantedBy=graphical.target
```

A few remarks:

*`After=systemd-user-sessions.service`*: To make sure service is started after
logins are permitted.

*`PAMName`*: Sets the PAM service name to set up a session as. Requires *`User=`*
or else the setting is ignored.

*`ExecStart=/bin/sleep infinity`*: We want to keep the service active as long
as the QM container is running. However, we do not want to execute the
compositor directly from the service, as it should be run as a container.
On the other hand, this can not be a [Quadlet](https://www.redhat.com/sysadmin/quadlet-podman)
either, since we want the *`Type=simple`* or the session will not start.
As a consequence, we just let it sleep.

- Make sure the service is enabled:

```shell
$ systemctl enable wayland-session
```

So that looks good enough, but unfortunately will not work. We are logging in
as root, which is mapped to the
[SELinux](https://www.redhat.com/en/topics/linux/what-is-selinux)' root user.
As part of the PAM login process, SELinux tries to perform a context switch,
which falls back to an `unconfined_u:unconfined_r:unconfined_t:s0` target
context. QM fobids the context switch, as it is unsafe, and the login fails.

The problem is within the PAM configuration files, at `/etc/pam.d/*`.
Specifically, we are running the login service at `/etc/pam.d/login` (given
by the `PAMName` setting of the service file), which includes a `pam_selinux`
step performing the context switch based on the user mappings. We want to
avoid any context switch, so that the resulting logged user stays in the
safe `qm_t` context (check `id -Z`).

- To do this and keep the login configuration as is (to avoid other attemps to login),
let's create a new configuration file at `/etc/pam.d/wayland`:

```
#%PAM-1.0
auth       substack     system-auth
auth       include      postlogin
account    required     pam_nologin.so
account    include      system-auth
password   include      system-auth
session    required     pam_loginuid.so
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      system-auth
session    include      postlogin
-session   optional     pam_ck_connector.so
```

And we change the PAMName to `wayland` in our `wayland-session.service` file.
Is that good enough? Well, not really. As part of the login process, the
user@ service is launched (i.e., `user@0.service`), which executes
`/usr/lib/systemd/systemd --user`, and another context switch is attempted and
fails. There is no way around it this time, we need to explicitely modify
the system configuration.

- Edit file at `/etc/pam.d/systemd-user` and remove/comment the lines
containing `pam_selinux`, resulting in something similar to this:

```
# This file is part of systemd.
#
# Used by systemd --user instances.

account  sufficient pam_unix.so no_pass_expiry
account  include system-auth

session  required pam_loginuid.so
session  optional pam_keyinit.so force revoke
session  optional pam_umask.so silent
session  required pam_namespace.so
session  include system-auth
```

If we start the container now, we should be able to see the session available:

```bash
bash-5.1# loginctl  
SESSION UID USER SEAT  TTY  STATE  IDLE SINCE
      1   0 root seat0 tty7 online no        

1 sessions listed.
```

However, the session is not active yet. In order to active the session
automatically before the compositor starts, we need to run `loginctl active`.

- Use a Quadlet file at `/etc/containers/systemd/session-activate.container`
to run the command inside a container:

```ini
[Unit]
Description=session-activate container

[Container]
ContainerName=session-activate
Environment=XDG_RUNTIME_DIR=/run/user/0
Environment=DBUS_SESSION_BUS_ADDRESS=unix:path=/run/dbus/system_bus_socket
Exec=/usr/bin/entrypoint.sh
Image=session-activate:latest
SecurityLabelType=qm_container_wayland_t
Volume=/run/systemd:/run/systemd:ro
Volume=/run/dbus/system_bus_socket:/run/dbus/system_bus_socket
Volume=/run/user/0:/run/user/0

[Install]
WantedBy=multi-user.target

[Service]
Restart=always
```

Notice the relabel to `qm_container_wayland_t`. This SELinux type must be
used by all containers running a Wayland application. We will see this label
used in other Quadlets in this guide. Otherwise the applications may lack
the required permissions to perform their tasks.

I will omit the `Containerfile` for this and other containers in this guide,
they are not too important and relatively simple in general.
In this case, however, it is worth showing what the `entrypoint.sh` script
looks like:

```bash
#!/bin/bash

SESSION=
while [ -z "$SESSION" ]; do
    sleep 1
    SESSION=$(loginctl list-sessions -o json | jq -re '.[] | select(.seat=="seat0").session')
done

loginctl activate $SESSION

exit 0
```

This changes the state of the session to active. But one more thing is required.
We explained how to create and activate a new session for `root`. However,
we need to create a user directory at `/run/user/0`, or else the
`session-activate.service` will fail to mount the volume.

- Create a new tmpfile drop-in configuration at `/usr/lib/tmpfiles.d/wayland-xdg-directory.conf`:
```
#Type Path                 Mode UID       GID          Age Argument
d     /run/user/0          0700 0         0            -   -
```

And this finishes the first step toward our goal.

## Start a user DBus daemon

This may be considered optional, but recommended. In order to avoid any process
to flood the system bus socket, which may jeopardize system stability, we are
going to create a new bus socket for the compositor. This can be done in a
similarly as how the system socket is created.

- Create a systemd socket file at `/etc/systemd/system/qm-dbus.socket`:

```ini
[Unit]
Description=QM D-Bus User Message Bus Socket
After=dbus.socket

[Socket]
ListenStream=%t/dbus/qm_bus_socket

[Install]
WantedBy=sockets.target
```

Make sure the systemd socket is enabled:

```shell
$ systemctl enable qm-dbus.socket
```

This creates the socket at `/run/dbus/qm_bus_socket`. 

- Run the DBus daemon with a Quadlet file at `/etc/containers/systemd/qm-dbus-broker.container`:

```ini
[Unit]
After=qm-dbus.socket
Description=qm-dbus-broker container
Requires=qm-dbus.socket

[Container]
ContainerName=qm-dbus-broker
Environment=XDG_RUNTIME_DIR=/run/user/0
Environment=DBUS_SESSION_BUS_ADDRESS=unix:path=/run/dbus/qm_bus_socket
Exec=/usr/bin/dbus-broker-launch --scope user
Image=qm-dbus-broker:latest
SecurityLabelType=qm_container_wayland_t
Volume=/run/dbus/qm_bus_socket:/run/dbus/qm_bus_socket
Volume=/run/systemd:/run/systemd:ro
Volume=/run/user/0:/run/user/0
Volume=/etc/machine-id:/etc/machine-id:ro

[Install]
Alias=qm-dbus.service
WantedBy=multi-user.target

[Service]
Restart=always
Sockets=qm-dbus.socket
```

## Launching Wayland container

For this section we are going to assume [Mutter](https://mutter.gnome.org/),
but similar steps can be used for other compositors.

First of all, we need to mount all required devices in the QM container.

- To this end, place a drop-in configuration file at
`/etc/containers/systemd/qm.container.d/wayland-extra-devices.conf` in the
ASIL layer:

```ini
[Container]
AddDevice=/dev/dri/renderD128
AddDevice=/dev/dri/card0
AddDevice=/dev/tty0
AddDevice=/dev/tty1
AddDevice=/dev/tty2
AddDevice=/dev/tty3
AddDevice=/dev/tty4
AddDevice=/dev/tty5
AddDevice=/dev/tty6
AddDevice=/dev/tty7
AddDevice=/dev/input/event0
AddDevice=/dev/input/event1
AddDevice=/dev/input/event2
AddDevice=/dev/input/event3
AddDevice=/dev/input/event4
Volume=/run/udev:/run/udev:ro,Z
```

- We can now just create a new Quadlet file for Mutter:

```ini
[Unit]
After=qm-dbus.socket
Description=mutter container
Requires=qm-dbus.socket

[Container]
ContainerName=mutter
Environment=XDG_RUNTIME_DIR=/run/user/0
Environment=XDG_SESSION_TYPE=wayland
Environment=DBUS_SESSION_BUS_ADDRESS=unix:path=/run/dbus/qm_bus_socket
Exec=mutter --no-x11 --wayland --sm-disable --wayland-display=wayland-0
Image=mutter:latest
SecurityLabelType=qm_container_wayland_t
Volume=/run/systemd:/run/systemd:ro
Volume=/run/udev:/run/udev:ro
Volume=/run/dbus/qm_bus_socket:/run/dbus/qm_bus_socket
Volume=/run/dbus/system_bus_socket:/run/dbus/system_bus_socket
Volume=/run/user/0:/run/user/0
AddDevice=/dev/dri/renderD128
AddDevice=/dev/dri/card0
AddDevice=/dev/tty0
AddDevice=/dev/tty1
AddDevice=/dev/tty2
AddDevice=/dev/tty3
AddDevice=/dev/tty4
AddDevice=/dev/tty5
AddDevice=/dev/tty6
AddDevice=/dev/tty7
AddDevice=/dev/input/event0
AddDevice=/dev/input/event1
AddDevice=/dev/input/event2
AddDevice=/dev/input/event3
AddDevice=/dev/input/event4

[Install]
WantedBy=multi-user.target

[Service]
Restart=always
```

This should get the Mutter compositor running inside the QM container!

However, the screen is empty, with a nice blue background. Now we can finally
add GUI applications that we want to be composited on the screen.

## Painting applications on screen

We will try to make a weston terminal appear in the screen.
This is just an example to illustrate the configuration.

After all we did so far, this should be easy enough in comparison. Just
take into account that workloads need to be containerized and relabeled
to `qm_container_wayland_t`.

- Quadlet configuration file at `/etc/containers/systemd/weston_terminal.container`
would look like this:

```ini
[Unit]
After=mutter.service
Description=weston_terminal container
Requires=mutter.service

[Container]
ContainerName=weston_terminal
Environment=XDG_RUNTIME_DIR=/run/user/0
Environment=WAYLAND_DISPLAY=wayland-0
Exec=/usr/bin/weston-terminal
Image=localhost/weston_terminal:latest
SecurityLabelType=qm_container_wayland_t
Volume=/run/user/0:/run/user/0

[Install]
WantedBy=multi-user.target

[Service]
Restart=always
```

If everything went well, you can now restart the QM container,
and the terminal will appear with the familiar Mutter background:

![mutter-in-qm](/img/mutter.png)

Enjoy!
