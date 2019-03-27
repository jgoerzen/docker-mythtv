# Docker MythTV Images

This is a set of images that makes it simple to set up a basic
headless server for [MythTV](http://www.mythtv.org).  These images
run on top
of my [Debian base system](http://github.com/jgoerzen/docker-debian-base),
which provides excellent logging capabilities.  This image is part of the
[docker-mythtv](https://github.com/jgoerzen/docker-mythtv) image set.

MythTV is a large and complex piece of software, and will require customization.
Some people use the MythTV backend to talk to local PCI or USB TV framegrabber/tuner
devices.  Such use, if it is even possible under Docker, is out of scope of
this document.  However, many TV tuners are now network-accessible, and thus
may be used over a local LAN with a Docker container without incident.

These images, therefore, handle the installation of MythTV for you.  You will
be responsible for the configuration to your own situation.  Please familiarize
yourself with the information on the MythTV website before proceeding.

You can view the [documentation for these images](https://github.com/jgoerzen/docker-mythtv)
on their Github page.

These images are provided:

 - [jgoerzen/mythtv-backend](https://github.com/jgoerzen/docker-mythtv-backend) - the MythTV backend server processes
 - [jgoerzen/mythtv-backend-mysql](https://github.com/jgoerzen/docker-mythtv-backend-mysql) - as mythtv-backend, but with an integrated MySQL/MariaDB server in
   the container

If you do not already have a database server on your network, selecting the
mysql variant will simplify your installation, though it is optional.

You can download with:

    docker pull jgoerzen/mythtv-backend-mysql

And run with something like this:

    docker run -td -p 6554:6554 -p 6543:6543 -p 6544:6544 -p 6549:6549 -p 5901:5901 \
    --stop-signal=SIGPWR \
    --hostname=mythtv-backend \
    -v /musicdir:/music:ro \
    -v /playlistdir:/playlists:rw \
    --name=mythtv jgoerzen/mythtv-backend-mysql

(Omit the `-mysql` from both commands if you have a MySQL server elsewhere that you
will connect to.)

Note: it is **critical** that the hostname be specified.  MythTV uses the
hostname as a key into its settings database, and Docker's randomly-assigned
hostnames will cause issues in this scenario.  See
[more about hostnames on the MythTV site](https://www.mythtv.org/wiki/Database_Backup_and_Restore#Change_the_hostname_of_a_MythTV_frontend_or_backend).

# Initial Setup Background

Although the MythTV backend is able to be run as a headless server, it
nevertheless requires setup to be done via a graphical program.  This
poses a bit of an annoyance for the normally all-text Docker environment.

I have, however, provided you with two options for accessing it.

Option 1 is to use VNC.  If Docker is running on localhost, or a host you
can ssh to with port forwarding, it will expose VNC screen 1 on port 5901.
You can connect to this with a VNC viewer and control it that way.

Option 2 is to use SSH with X11 forwarding.  This is a more advanced
option that is largely outside the scope of this document.  The basic steps
are to add `-e DEBBASE_SSH=enabled` to your `docker run` command, forward
port 23 into the container, provision a password for a user inside the container
with `passwd` or similar, and then use `ssh -X` to connect to it.

Please note that SSH is an encrypted protocol, but VNC generally is not;
it is not secure to expose the VNC port over the Internet.

Either way, you will need to know how to get a shell prompt within your
container.  If you, for instance, named it mythtv-backend, then
`docker exec -ti mythtv-backend bash` will do the trick.

As we go along, I will try to make it clear what steps you can do with
your own Dockerfile (which will be most of them).

## jgoerzen/mythtv-backend (non-mysql) only: Prepare database

The `mythtv-database` package generally wants to be installed on the database server itself.
You may have some trickery to do here.

First, let's assign a username and password.  Run:

    dpkg-reconfigure -plow mythtv-common

Next, if you do not already have a MythTV database,
install the package that configures or sets up the MythTV database.
Because it requires a database to be present to install, it is only provided by
default in the mythtv-backend-mysql package.  This package will configure your database to an
existing server.  Open a shell in the container and run:

    apt-get update
    rm /etc/apt/apt.conf.d/docker-clean
    apt-get install --no-install-recommends mythtv-database

It will ask for a password for the administrator account.
It will then create the needed database.  Please note that this only needs
to be done once; you do not need this package installed in your
ongoing containers.

## All setups: configure the backend

Now you're ready to configure the backend.  Fire up the VNC server with:

    su - mythtv -c startvnc
    su - mythtv -c "DISPLAY=:1 xterm"

The first time you run `startvnc`, it will prompt you to enter a password.
Make up something secure.

On your workstation, a command like this should connect you to the GUI:

   xvnc4viewer localhost:1

Enter the same password you set before.

In the GUI that appears, run:

   mythtv-setup

On the general settings page, the IP address of the backend should be set to
the interal IP address (more on that below).

When you are all done, stop the VNC server with:

    su - mythtv -c 'tigervncserver -kill :1'

# Hints for your own Dockerfiles

Here are some files you may want to install:

 - ~mythtv/.mythtv/config.xml (/var/lib/mythtv)
 - /etc/mythtv/config.xml

You could prime the Debconf database with the MythTV passwords with:

    echo 'mythtv-common mythtv/mysql_mythtv_password password foo' | debconf-set-selections
    echo 'mythtv-common mythtv/mysql_mythtv_user string mythtv' | debconf-set-selections
    echo 'mythtv-common mythtv/mysql_mythtv_dbname string mythconverg' | debconf-set-selections
    echo 'mythtv-common mythtv/mysql_host string localhost' | debconf-set-selections

# IP address notes

So this is pretty annoying.  MythTV insists on storing the IP
address as seen by the backend in its database, and other
machines use that IP to reach it.  With docker, there is generally
no way tihs works because of NAT.

Also, if you use a capture device like the HDHomeRun which communicates
back to MythTV via random UDP ports, it can be just about impossible to
make things work with the standard docker port forwarding.

However, there are some workarounds.

You can:

 - Use NAT reflection on your firewall to forward packets
   back in to your network.
 - [Bridge your Docker containers to the network](https://developer.ibm.com/recipes/tutorials/bridge-the-docker-containers-to-external-network/)
   - An example: `docker network create --driver=bridge --ip-range=192.168.0.192/29 --subnet=192.168.0.0/24 --aux-address "DefaultGatewayIPv4=192.168.0.1" -o "com.docker.network.bridge.name=brlan1" brlan1`
   - After that, you can add `network=brlan1 --ip=192.168.0.193` do your `docker run`, and you do not need
     any `-p` because it will be directly accessible on the new IP.
 - Add egress iptables rules to your frontends
 - You can set the BackendServerIP and MasterServerIP to the "visible" IP
   of the backend (will probably have to do this via mysql).  mythbackend
   will fail to bind to a visible IP, but a userland redirector like `redir`
   may do the trick.

I tried adding `-O BackendServerIP=blah -O MasterServerIP=blah` to my
mythfrontend command line.  That let it boot, but wasn't sufficient for
streaming.

# Bugs

mythbackend doesn't seem to properly write its PID to /var/run/mythtv, and
therefore commands that try to kill it won't work.

# Source

This is prepared by John Goerzen <jgoerzen@complete.org> and the source
can be found at https://github.com/jgoerzen/docker-mythtv

# Security Status

The Debian operating system is configured to automatically apply security patches.
MythTV, however, does not have such a feature.

# Tags

Because this is built from the deb-multimedia.org sources, I cannot easily
provide historical builds, since deb-multimedia itself does not.  Threfore,
only current deb-multimedia.org builds are available.

# Copyright

Docker scripts, etc. are
Copyright (c) 2017 John Goerzen
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions
are met:
1. Redistributions of source code must retain the above copyright
   notice, this list of conditions and the following disclaimer.
2. Redistributions in binary form must reproduce the above copyright
   notice, this list of conditions and the following disclaimer in the
   documentation and/or other materials provided with the distribution.
3. Neither the name of the University nor the names of its contributors
   may be used to endorse or promote products derived from this software
   without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE AUTHORS AND CONTRIBUTORS ``AS IS'' AND
ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
ARE DISCLAIMED.  IN NO EVENT SHALL THE REGENTS OR CONTRIBUTORS BE LIABLE
FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS
OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION)
HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY
OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF
SUCH DAMAGE.

Additional software copyrights as noted.

