# IMPORTANT NOTE

This image is not yet ready for use!

# Docker MythTV Images

This is a set of images that makes it simple to set up a basic
headless server for [MythTV](http://www.mythtv.org).  These images
run on top
of my [Debian base system](http://github.com/jgoerzen/docker-debian-base),
which provides excellent logging capabilities.

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

 - jgoerzen/mythtv-backend - the MythTV backend server processes
 - jgoerzen/mythtv-backend-mysql - as mythtv-backend, but with an integrated MySQL/MariaDB server in
   the container

If you do not already have a database server on your network, selecting the
mysql variant will simplify your installation, though it is optional.

You can download with:

    docker pull jgoerzen/mythtv-backend-mysql

FIXME

And run with something like this:

    docker run -td -p 8080:80 -p 80443:443 -p 5901:5901 --stop-signal=SIGPWR \
    --hostname=mythtv-backend \
    -v /musicdir:/music:ro \
    -v /playlistdir:/playlists:rw \
    --name=ampache jgoerzen/mythtv-backend-mysql

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

Now you're ready to configure the backend.  Fire up the VNC server with:

    su - mythtv -c startvnc
    su - mythtv -c "DISPLAY=:1 xterm"

On your workstation, a command like this should connect you to the GUI:

   xvnc4viewer localhost:1

In the GUI that appears, run:

   mythtv-setup

su - mythtv -c 'tigervncserver -kill :1'


 - MySQL (administrative) password: ampache
 - Create database: uncheck

Other suggestions:

 - Template configuration: ffmpeg

Once configured, add a catalog pointing to `/music` at <http://localhost:8080/ampache/index.php#admin/catalog.php?action=show_add_catalog>, and another for `/playlists`.

# Ports

By default, this image exposes a HTTP server on port 80, HTTPS on port 443, and
also exposes port 81 in case you wish to use it separately for certbot or another
Letsencrypt validation system.  HTTPS will require additional configuration.

Ampache is exposed at path `/ampache` on the configured system. 

# Source

This is prepared by John Goerzen <jgoerzen@complete.org> and the source
can be found at https://github.com/jgoerzen/docker-ampache

# Security Status

The Debian operating system is configured to automatically apply security patches.
Ampache, however, does not have such a feature, nor do most of the third-party
PHP modules it integrates.

There is some security risk in making the installation directory writable by
the web server process.  This is restricted as much as possible in this image.
A side-effect of that, however, is the disabling of the Ampache auto-update
feature.  If you wish to be able to use Ampache's built-in updates, you
should `chown -R www-data:www-data /var/www/html/ampache`.

# Tags

These Docker tags are defined:

 - latest is built against the Ampache github master branch (which they recommend)
 - Other branches use the versioned tarballs

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

