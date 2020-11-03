<div class="author">

Michael Strobler &lt;<maelstor@posteo.de>&gt;

</div>

<div class="tag">

lang-ruby

</div>

<div class="tag">

lang-yaml

</div>

<div class="tag">

audience-developers

</div>

<div class="tag">

continuous-integration

</div>

<div class="tag">

automation

</div>

<div class="sidebar">

**Logo**

<img src="_static/images/gitlab-logo.png" class="align-center" alt="image" />

</div>

GitLab
======

[GitLab](https://gitlab.com) is a web-based DevOps lifecycle tool that
provides a Git-repository manager providing wiki, issue-tracking and
continuous integration and deployment pipeline features, using an
open-source license, developed by GitLab Inc.[1]

------------------------------------------------------------------------

<div class="note">

<div class="title">

Note

</div>

For this guide you should be familiar with the basic concepts of

-   `supervisord <daemons-supervisord>`
-   `web backends <web-backends>`
-   `ssh <basics-ssh>`
-   `resources <basics-resources>`
-   `ports <basics-ports>`
-   `ruby <lang-ruby>`
-   `nodejs <lang-nodejs>`
-   `perl <lang-perl>`
-   `gcc <lang-gcc>`

</div>

Prerequisites
-------------

### Requirements

Before starting to deploy gitlab on uberspace there are some preliminary
conciderations to take.

Since one uberspace account is limited to 1536MB RAM, you will need 2
uberspace accounts for full functionality without crashes. The gitlab
instance will need around 1,4GB RAM and the sidekiq instance with
postgresql intalled will need around 750MB RAM. The sidekiq instance
doesn't need to be dedicated solely to sidekiq. For example a mediawiki
installation fits into it, but anything leightweight will do, if you
like to install anything alongside to sidekiq. Just be sure you can
spare 5GB of disk space during the setup of sidekiq and around 2.5GB
permanent disk space.

The compilation of the assets most likely won't work on your uberspace
host because the process uses more than the allowed RAM and gets
automatically killed. So you'll need around 5GB disk space left on your
home PC and in your home directory to compile the assets there. In case
you want to try compiling the assets locally before going through the
complete guide and noticing it doesn't work jump to the [Compile
assets](#compile-assets) section.

You should be aware that on the gitlab instance there will be only
around 4GB of disk space left for your repositories.

Adding all up the final gitlab instance can be used by yourself and a
small development team.

If you're finished setting up gitlab you may have noticed that the day
is gone, so be sure you can spare some hours up to a day ;)

This guide is roughly based on the official [GitLab installation from
source](https://docs.gitlab.com/13.4/ee/install/installation.html) guide
with a lot of adjustments to make it work on uberspace. The main
differences are the lack of root access (this is why to install GitLab
from source and the need to build some software from source), a
different user than the default `git` user and the resource limitations,
described above.

Finally you'll need a piece of paper or some sort of digital scratchpad
to note things down.

### Structure of this guide

Because it is clearer we first setup the `gitlab` host and then the
`sidekiq` host as far it is possible. Setting up the `sidekiq` host is
in parts similar to the setup of `gitlab` host and once you're done with
`gitlab` and be warmed up, the `sidekiq` host is set up pretty quick. Be
sure you have both accounts created before starting. If not otherwise
stated the commands besides in the [Installation
sidekiq](#installation-sidekiq) section are meant to be run on the
`gitlab` host.

Throughout the guide I'm using `isabell@gitlab` for the gitlab instance,
`isabell@sidekiq` for the sidekiq instance and `isabell@home` for your
home PC's account. `isabell` can't exist on both uberspace hosts but for
the sake of simplicity I'll keep it written that way.

### Variables

This guide is pretty long and in order to refer to created passwords,
ips, domain names etc. in following sections I'll use upper case
variable names, which you have to replace with their values when adviced
to do so.

Let's start with the `SIDEKIQ_FQDN` and the `GITLAB_FQDN`. You can get
them with

``` console
[isabell@gitlab ~]$ hostname -f
gitlab.uberspace.de
[isabell@gitlab ~]$
```

Do the same on your `sidekiq` host and write them down. Your usernames
are referred to as `SIDEKIQ_USERNAME` and `GITLAB_USERNAME`. In our case
this would be `isabell` for both of them.

We also need the `GITLAB_EXTERNAL_FQDN`. That is your external address
at which you want to be gitlab reachable from your browser. It is
composed like that

<div class="warning">

<div class="title">

Warning

</div>

Replace `GITLAB_USERNAME`

</div>

``` console
GITLAB_USERNAME.uber.space
```

### Passwords

A quick word about passwords. The `REDIS_PASSWORD`,
`POSTGRESQL_GITLAB_PASSWORD` and `POSTGRESQL_SUPERUSER_PASSWORD` need to
be url encoded in some parts of the guide. To avoid the confusion when
an when not to url encode, you can choose to use an alphanumeric
password `[a-zA-Z0-9]` with a higher length. Something between 64 and
128 characters should provide a very secure password without the need to
go through the url encoding. A 128 character alphanumeric password can
easily be created

``` console
[isabell@gitlab ~]$ echo -n 'my secret passphrase' | sha512sum | sed 's/[ -]//g'
1d143ea6fb069e71fa8c90b3f81283cc71bf8d182448fda3e8cc3ae6ee8955b8baa6e616adfa2ffbb436791df91f07fdeac3c8083e1fabfd597398a97801c4a2
[isabell@gitlab ~]$
```

For a 96 character password use `sha384sum` (64 chars =&gt; `sha256sum`
...). If you don't care about a passphrase you can increase the security
of your password

``` console
[isabell@gitlab ~]$ head -c 128 /dev/random | sha512sum | sed 's/[ -]//g'
[isabell@gitlab ~]$
```

The `128` value is chosen by me and you can use anything you want, say
from 64 characters onwards to end up with a very secure password. The
more characters the lower the speed of the password creation but the
higher the security.

However if you really don't want an alphanumeric password you can url
encode it

``` console
[isabell@gitlab ~]$ python3
Python 3.6.8 (default, Apr  2 2020, 13:34:55)
[GCC 4.8.5 20150623 (Red Hat 4.8.5-39)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import urllib.parse as p
>>> password = '# {my secret password as string or as bytes!}'
>>> p.quote(password)
'%23%20%7Bmy%20secret%20password%20as%20string%20or%20as%20bytes%21%7D'
>>> quit()
[isabell@gitlab ~]$
```

Installation Dependencies
-------------------------

There are a lot of them and some need to be built from source. I'll just
tell about those which need attention or are not installed by default on
uberspace hosts. You can have a look at all dependencies at [GitLab
installation from source
dependencies](https://docs.gitlab.com/13.4/ee/install/installation.html#1-packages-and-dependencies)
. The basic working directory is `$HOME/workspace`. You can choose a
different one if you like. Since we dont' need it anymore when
[GitLab](https://gitlab.com) is installed, delete it when you reached
the end of the gitlab installation process to get back some disk space.

### Cmake

Check the current available version of `cmake` with

``` console
[isabell@gitlab ~]$ cmake --version
cmake version 2.8.12.2
[isabell@gitlab ~]$
```

If it is below `3.0.0` than we'll need to install cmake from source in a
more recent version :

``` console
[isabell@gitlab ~]$ mkdir -p workspace/cmake
[isabell@gitlab ~]$ cd workspace/cmake
[isabell@gitlab ~/workspace/cmake]$ wget https://github.com/Kitware/CMake/releases/download/v3.18.3/cmake-3.18.3.tar.gz
[isabell@gitlab ~/workspace/cmake]$ tar xzf cmake-3.18.3.tar.gz && cd cmake-3.18.3
[isabell@gitlab ~/workspace/cmake/cmake-3.18.3]$ ./bootstrap --prefix=$HOME/.local
[isabell@gitlab ~/workspace/cmake/cmake-3.18.3]$ make
[isabell@gitlab ~/workspace/cmake/cmake-3.18.3]$
```

You can skip the `test` if you like to, because it takes a while to
complete but to be sure that your installation will work you should run
it. I suggest

``` console
[isabell@gitlab ~/workspace/cmake/cmake-3.18.3]$ make test ARGS='--output-on-failure -E "^RunCMake\.CPack_RPM\.SUGGESTS$$"'
[isabell@gitlab ~/workspace/cmake/cmake-3.18.3]$
```

-   `ARGS`: passes the arguments to ./bin/ctest which runs the test
    suite
    -   `--output-on-failure`: Shows the error output if a test fails.
    -   `-E`: Excludes the specified tests. Accepts regular expressions.
        The one specified above fails on centos 7 for some reason but is
        not critical for cmake to run (See
        [Issue](https://gitlab.kitware.com/cmake/cmake/-/issues/21043#note_808821)).
        I encountered another failure `346 - RunCMake.Make (Failed)`
        which just happens sometimes. I've ran the test isolated with
        success. (See below `-R`). Be cautions with the `$` sign it
        needs to be escaped the Makefile way, so it is doubled.
    -   `-R`: Run only selected tests. Accepts regular expressions.

Finally install `cmake`

``` console
[isabell@gitlab ~/workspace/cmake/cmake-3.18.3]$ make install
[isabell@gitlab ~/workspace/cmake/cmake-3.18.3]$
```

and add the `$HOME/.local/bin` dir to the PATH. Edit `$HOME/.bashrc` to
include the following

``` bash
export PATH="$HOME/.local/bin:$PATH"
```

``` console
[isabell@gitlab ~]$ source $HOME/.bashrc
[isabell@gitlab ~]$
```

Check the cmake version

``` console
[isabell@gitlab ~]$ cmake --version
cmake version 3.18.3

CMake suite maintained and supported by Kitware (kitware.com/cmake).
[isabell@gitlab ~]$
```

If your output looks similar you're all set.

### Git

Check that git has minimum version `>=2.24.0` and is compiled with
`libpcre2`.

``` console
[isabell@gitlab ~]$ git --version && ldd $(command -v git) | grep pcre2
git version 2.24.3
    libpcre2-8.so.0 => /lib64/libpcre2-8.so.0 (0x00007fc2bf65c000)
[isabell@gitlab ~]$
```

This is normally the case and if the output looks similar then
everything's fine.

### Exiftool

GitLab Workhorse needs `exiftool` to remove EXIF data from uploaded
images. As of `Uberspace 7.7.9.0` it is not available but you can check
yourself with

``` console
[isabell@gitlab ~]$ command -v exiftool || echo nope
nope
[isabell@gitlab ~]$
```

We'll install it from source into the `$HOME/.local` hierarchy

``` console
[isabell@gitlab ~]$ mkdir -p workspace/exiftool
[isabell@gitlab ~]$ cd workspace/exiftool
[isabell@gitlab ~/workspace/exiftool]$ wget 'https://exiftool.org/Image-ExifTool-12.07.tar.gz'
[isabell@gitlab ~/workspace/exiftool]$ tar xzf Image-ExifTool-12.07.tar.gz
[isabell@gitlab ~/workspace/exiftool]$ cd Image-ExifTool-12.07
[isabell@gitlab ~/workspace/exiftool/Image-ExifTool-12.07]$ perl Makefile.PL INSTALL_BASE="$HOME/.local"
[isabell@gitlab ~/workspace/exiftool/Image-ExifTool-12.07]$ make test
[isabell@gitlab ~/workspace/exiftool/Image-ExifTool-12.07]$ make install
[isabell@gitlab ~/workspace/exiftool/Image-ExifTool-12.07]$
```

The man pages of `exiftool` aren't installed into the right location so
we have to fix that:

``` console
[isabell@gitlab ~/workspace/exiftool/Image-ExifTool-12.07]$ cd ~/.local
[isabell@gitlab ~/.local]$ mkdir -p share
[isabell@gitlab ~/.local]$ cp -af man share
[isabell@gitlab ~/.local]$ rm -rf man
[isabell@gitlab ~/.local]$
```

Optionally add the `$HOME/.local/share/man` and
`$HOME/.local/share/info` paths to MANPATH and INFOPATH in your
`$HOME/.bashrc`

``` bash
export MANPATH="$HOME/.local/share/man:$MANPATH"
export INFOPATH="$HOME/.local/share/info:$INFOPATH"
```

Next add the local perl libraries to the INC path of perl. Edit
`$HOME/.bashrc` and add

``` bash
export PERL5LIB="$HOME/.local/lib/perl5:$PERL5LIB"
```

Now source your `$HOME/.bashrc`

``` console
[isabell@gitlab ~]$ source $HOME/.bashrc
[isabell@gitlab ~]$
```

Check that the INC path includes
`/home/GITLAB_USERNAME/.local/lib/perl5`

``` console
[isabell@gitlab ~]$ perl -V
# ... a lot of output we don't care about right now
@INC:
    /home/<username>/.local/lib/perl5
    /usr/local/lib64/perl5
    /usr/local/share/perl5
    /usr/lib64/perl5/vendor_perl
    /usr/share/perl5/vendor_perl
    /usr/lib64/perl5
    /usr/share/perl5
[isabell@gitlab ~]$
```

At the very bottom of this pretty long output you'll see the INC path
where `<username>` is your `GITLAB_USERNAME`.

### Ruby

We'll need ruby in version `2.6.x`. Check the current version with

``` console
[isabell@gitlab ~]$ ruby --version
[isabell@gitlab ~]$
```

If it is below or higher than the required version we'll change to `2.6`

``` console
[isabell@gitlab ~]$ uberspace tools version use ruby 2.6
Selected Ruby version 2.6
The new configuration is adapted immediately. Patch updates will be applied automatically.

[isabell@gitlab ~]$
```

Bundler is needed in a version `>=1.5.2, < 2`. Install bundler with

``` console
[isabell@gitlab ~]$ gem install bundler --no-document --version '>=1.5.2, < 2'
Fetching bundler-1.17.3.gem
WARNING:  You don't have /home/<username>/.gem/ruby/2.6.0/bin in your PATH,
          gem executables will not run.
Successfully installed bundler-1.17.3
1 gem installed
[isabell@gitlab ~]$
```

We can ignore the `WARNING` (where `<username>` is your
`GITLAB_USERNAME`) because we have
`/opt/uberspace/etc/<username>/binpaths/ruby` in our PATH, which
resolves to the path `gem` is moaning about that it's not there

<div class="warning">

<div class="title">

Warning

</div>

Replace `GITLAB_USERNAME`

</div>

``` console
[isabell@gitlab ~]$ readlink -e /opt/uberspace/etc/GITLAB_USERNAME/binpaths/ruby
/home/<username>/.gem/ruby/2.6.0/bin
[isabell@gitlab ~]$
```

### Re2

Usually `libre2` isn't installed on uberspace hosts but you can check it
yourself with

:

``` console
[isabell@gitlab ~]$ /sbin/ldconfig -p | grep 'libre2\.so' || echo nope
nope
[isabell@gitlab ~]$
```

So let's install it from source

``` console
[isabell@gitlab ~]$ mkdir -p workspace/libre2 && cd workspace
[isabell@gitlab ~/workspace]$ git clone 'https://github.com/google/re2.git' libre2
[isabell@gitlab ~/workspace]$ cd libre2
[isabell@gitlab ~/workspace/libre2]$ make prefix="$HOME/.local"
[isabell@gitlab ~/workspace/libre2]$ make test prefix="$HOME/.local"
[isabell@gitlab ~/workspace/libre2]$ make install prefix="$HOME/.local"
[isabell@gitlab ~/workspace/libre2]$ make testinstall prefix="$HOME/.local"
[isabell@gitlab ~/workspace/libre2]$
```

Make sure that `LD_LIBRARY_PATH` is set to include `$HOME/.local/lib` in
your `$HOME/.bashrc`

``` bash
export LD_LIBRARY_PATH="$HOME/.local/lib:$LD_LIBRARY_PATH"
```

and source your `$HOME/.bashrc`

``` console
[isabell@gitlab ~]$ source $HOME/.bashrc
[isabell@gitlab ~]$
```

bundler needs to know about the `libre2` path too

``` console
[isabell@gitlab ~]$ bundler config set --global build.re2 "--with-re2-dir=$HOME/.local"
[isabell@gitlab ~]$
```

### Runit

Only the `runit` binaries are required but since they are not installed
on uberspace we need to compile them ourselves.

``` console
[isabell@gitlab ~]$ mkdir -p workspace/runit
[isabell@gitlab ~]$ cd workspace/runit
[isabell@gitlab ~/workspace/runit]$ wget 'http://smarden.org/runit/runit-2.1.2.tar.gz'
[isabell@gitlab ~/workspace/runit]$ tar xzpf runit-2.1.2.tar.gz
[isabell@gitlab ~/workspace/runit]$ cd admin/runit-2.1.2
[isabell@gitlab ~/workspace/runit/admin/runit-2.1.2]$ sed -i 's/ -static//g' src/Makefile
[isabell@gitlab ~/workspace/runit/admin/runit-2.1.2]$ sed -i 's:/service/:'"$HOME"'/.local/var/service/:g' src/sv.c
[isabell@gitlab ~/workspace/runit/admin/runit-2.1.2]$ echo 'gcc' > src/conf-cc
[isabell@gitlab ~/workspace/runit/admin/runit-2.1.2]$ echo 'gcc -s' > src/conf-ld
[isabell@gitlab ~/workspace/runit/admin/runit-2.1.2]$ make -C src
[isabell@gitlab ~/workspace/runit/admin/runit-2.1.2]$ for c in $(<package/commands); do cp -a "src/$c" "$HOME/.local/bin/$c"; done
[isabell@gitlab ~/workspace/runit/admin/runit-2.1.2]$
```

If you've not already added `$HOME/.local/bin` to your `PATH` do it now.

### Node

nodejs is required with a minimum version of `>= 10.13.0` but node
`12.x` is faster. Check you're version with

``` console
[isabell@gitlab ~]$ node --version
v12.19.0
[isabell@gitlab ~]$
```

and if necessary change it with

``` console
[isabell@gitlab ~]$ uberspace tools version use node 12
Selected Node.js version 12
The new configuration is adapted immediately. Minor updates will be applied automatically.

[isabell@gitlab ~]$
```

yarn is needed with a minimum version of `>=1.10.0` and is installed on
uberspace but check with

``` console
[isabell@gitlab ~]$ yarn --version
[isabell@gitlab ~]$
```

### PostgreSQL

This is the only supported database by [GitLab](https://gitlab.com). We
need postgresql installed on both the `gitlab` host for the libraries
and on the `sidekiq` host for the functionality.

#### PostgreSQL Installation

<div class="note">

<div class="title">

Note

</div>

To keep things in line start the [PostgreSQL
Installation](#postgresql-installation) on the `sidekiq` host and then
on the `gitlab` host before [Creating the Database
Cluster](#creating-the-database-cluster) on the `sidekiq` host.

</div>

Follow the uberlab guide `PostgreSQL <guide_postgresql>` until
:lab\_anchor:<span class="title-ref">Step 3
&lt;guide\_postgresql.html\#step-3-environment-settings&gt;</span>. I
strongly recommend configuring with `./configure --prefix $HOME/.local`
into the `$HOME/.local` hierarchy because I will refer to this directory
structure in this guide. It also makes sure that your PATH settings
don't need to be adjusted once you've added `$HOME/.local/bin` to your
PATH in the `$HOME/.bashrc` like described above.

#### Environment Settings

In your `$HOME/.bashrc` adjust the `LD_LIBRARY_PATH` to include
`$HOME/.local/lib/postgresql` and `PGPASSFILE` environment variables

``` bash
export LD_LIBRARY_PATH="$HOME/.local/lib/postgresql:$HOME/.local/lib:$LD_LIBRARY_PATH"
export PGPASSFILE="$HOME/.pgpass"
```

and source `$HOME/.bashrc` to make you're current shell recognize the
changes.

Create and edit the `$HOME/.pgpass` file on the `sidekiq` host with the
following content

<div class="warning">

<div class="title">

Warning

</div>

Replace `SIDEKIQ_USERNAME`. Replace `POSTGRESQL_SUPERUSER_PASSWORD` with
a secure password. (See the [Passwords](#passwords) section)

</div>

``` console
*:*:*:SIDEKIQ_USERNAME:POSTGRESQL_SUPERUSER_PASSWORD
```

Change permissions to

``` console
[isabell@sidekiq ~]$ chmod 0600 "$HOME/.pgpass"
[isabell@sidekiq ~]$
```

On your `gitlab` host edit `$HOME/.pgpass` to match the following
content

<div class="warning">

<div class="title">

Warning

</div>

Replace `POSTGRESQL_GITLAB_PASSWORD` with a secure password. (See the
[Passwords](#passwords) section)

</div>

``` console
*:*:gitlab:gitlab:POSTGRESQL_GITLAB_PASSWORD
```

and change the permissions to

``` console
[isabell@gitlab ~]$ chmod 0600 "$HOME/.pgpass"
[isabell@gitlab ~]$
```

#### Creating the Database Cluster

<div class="note">

<div class="title">

Note

</div>

We need the database cluster only on the `sidekiq` host.

</div>

Dump your plain `POSTGRESQL_SUPERUSER_PASSWORD` from above into
`$HOME/pgpass.temp`

<div class="warning">

<div class="title">

Warning

</div>

Replace `POSTGRESQL_SUPERUSER_PASSWORD`

</div>

``` console
POSTGRESQL_SUPERUSER_PASSWORD
```

Create the database cluster and remove the temporary `$HOME/pgpass.temp`
file.

``` console
[isabell@sidekiq ~]$ initdb --pwfile ~/pgpass.temp --auth=scram-sha-256 -E UTF8 -D ~/.local/var/postgresql
[isabell@sidekiq ~]$ rm "$HOME/pgpass.temp"
[isabell@sidekiq ~]$
```

#### PostgreSQL Port

<div class="note">

<div class="title">

Note

</div>

This step is also only needed on the `sidekiq` host

</div>

We need `postgresql` to be available from the outside so the `gitlab`
host can communicate with `postgresql`. On the `sidekiq` host execute

``` console
[isabell@sidekiq ~]$ uberspace port add
Port 55555 will be open for TCP and UDP traffic in a few minutes.
[isabell@sidekiq ~]$
```

Write this port number down, in this case `55555`. We'll need it in
different places later. I'll refer to it with `POSTGRESQL_PORT`.

#### PostgreSQL Configuration

<div class="note">

<div class="title">

Note

</div>

Only on the `sidekiq` host if not otherwise noted

</div>

We need the ip address of the `veth` interface to know the bind address
for postgresql

``` console
[isabell@sidekiq ~]$ /sbin/ifconfig | grep -A1 veth
veth_isabell: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
    inet 100.64.4.53  netmask 255.255.255.252  broadcast 0.0.0.0
[isabell@sidekiq ~]$
```

Write the ip down (in the case above `100.64.4.53`). I'll refer to it as
`SIDEKIQ_VETH_IP`.

Edit the `$HOME/.local/var/postgresql/postgresql.conf` configuration
file and adjust the following values:

<div class="warning">

<div class="title">

Warning

</div>

Replace `POSTGRESQL_PORT`, `SIDEKIQ_VETH_IP` and `SIDEKIQ_USERNAME`

</div>

``` kconfig
#------------------------------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#------------------------------------------------------------------------------

# - Connection Settings -

listen_addresses = 'SIDEKIQ_VETH_IP'    # what IP address(es) to listen on;
                                        # comma-separated list of addresses;
                                        # defaults to 'localhost'; use '*' for all
                                        # (change requires restart)
port = POSTGRESQL_PORT                  # (change requires restart)
unix_socket_directories = '/home/SIDEKIQ_USERNAME/.local/tmp' # comma-separated list of directories
unix_socket_permissions = 0700          # begin with 0 to use octal notation
```

In the next step we need the external ip address of our `gitlab` host.
You can get the ip address by executing

<div class="warning">

<div class="title">

Warning

</div>

Replace `GITLAB_FQDN`

</div>

``` console
[isabell@sidekiq ~]$ dig GITLAB_FQDN +short
185.26.156.230
[isabell@sidekiq ~]$
```

for example on your `sidekiq` host. I'll refer to this ip with
`GITLAB_IP`.

Furthermore we need the gateway of the `sidekiq` host. Execute on the
`sidekiq` host.

``` console
[isabell@sidekiq ~]$ /sbin/ip route | head -1
default via 100.65.17.1 dev veth_<username> proto static
[isabell@sidekiq ~]$
```

where `<username>` matches your `SIDEKIQ_USERNAME`. The important part
for us is the ip address, in the case above `100.65.17.1`. I'll refer to
it as `SIDEKIQ_GATEWAY_IP`.

Finally edit the `$HOME/.local/var/postgresql/pg_hba.conf` configuration
file on your `sidekiq` host:

<div class="warning">

<div class="title">

Warning

</div>

Replace `GITLAB_IP` and `SIDEKIQ_GATEWAY_IP`

</div>

``` kconfig
# TYPE  DATABASE        USER            ADDRESS                 METHOD

local   all             all                                   peer
local   gitlab          gitlab                                scram-sha-256
host    gitlab,postgres gitlab          GITLAB_IP/32          scram-sha-256
host    gitlab,postgres gitlab          SIDEKIQ_GATEWAY_IP/32 scram-sha-256
# host    all             all             127.0.0.1/32        scram-sha-256
# host    all             all             ::1/32              scram-sha-256
# local   replication     all                                 scram-sha-256
# host    replication     all             127.0.0.1/32        scram-sha-256
# host    replication     all             ::1/128             scram-sha-256
```

The `local` type sets the connection method for linux `unix` sockets. We
use `peer` as long we haven't set a password for the postgresql
superuser. We also comment out all other connections we don't need.
`host` configures `tcp` sockets. The 3rd and 4th line ensure that you
can connect from your `gitlab` host to the `sidekiq` host. Since this
depends on the location of the servers you're either routed via the
gateway or external. We accept both methods to be sure.

Add or adjust the following environment variables in your
`$HOME/.bashrc`

<div class="warning">

<div class="title">

Warning

</div>

Replace `POSTGRESQL_PORT`

</div>

``` bash
# PostgreSQL configuration on the sidekiq host

export PGPASSFILE="$HOME/.pgpass"
export PGHOST="$HOME/.local/tmp"
export PGPORT="POSTGRESQL_PORT"
export PGDATA="$HOME/.local/var/postgresql"
export PGUSER="gitlab"
export PGDATABASE="gitlab"
```

and source it with `source $HOME/.bashrc`.

On the `gitlab` host add the following to your `$HOME/.bashrc`

<div class="warning">

<div class="title">

Warning

</div>

Replace `POSTGRESQL_PORT` and `SIDEKIQ_FQDN`.

</div>

``` bash
# PostgreSQL configuration on the gitlab host

export PGPASSFILE="$HOME/.pgpass"
export PGHOST="SIDEKIQ_FQDN"
export PGPORT="POSTGRESQL_PORT"
export PGUSER="gitlab"
export PGDATABASE="gitlab"
```

and source it.

#### Setup the supervisor PostgreSQL service

<div class="note">

<div class="title">

Note

</div>

Only on the `sidekiq` host

</div>

Create `$HOME/etc/services.d/postgresql.ini` with the folllowing
content:

``` ini
[program:postgresql]
command=%(ENV_HOME)s/.local/bin/postgres -D %(ENV_HOME)s/.local/var/postgresql/
autostart=yes
autorestart=yes
redirect_stderr=true
stdout_logfile=%(ENV_HOME)s/logs/postgresql.log
```

Reload configuration files, start the service and check the status with

``` console
[isabell@sidekiq ~]$ supervisorctl reread
postgresql: available
[isabell@sidekiq ~]$ supervisorctl update postgresql
postgresql: added process group
[isabell@sidekiq ~]$ supervisorctl status postgresql
postgresql                       RUNNING   pid 11631, uptime 0:00:06
[isabell@sidekiq ~]$
```

Make sure postgresql is listening on the configured port and addresses

``` console
[isabell@sidekiq ~]$ netstat -xtlpn | grep postgres
tcp        0      0 <sidekiq_veth_ip>:<port>       0.0.0.0:*               LISTEN      28577/postgres
unix  2      [ ACC  ]     STREAM     LISTENING     821724058 19224/postgres /home/<username>/.local/tmp/.s.PGSQL.<port>
[isabell@sidekiq ~]$
```

The output should look similar to the output above with your
`POSTGRESQL_PORT` as `<port>`, `<username>` as your `SIDEKIQ_USERNAME`
and `SIDEKIQ_VETH_IP` as `<sidekiq_veth_ip>` . The process numbers in
`28577/postgres` may differ.

#### Check PostgreSQL connection from sidekiq host

Check that you can connect to the database from your `sidekiq` host as
superuser.

<div class="warning">

<div class="title">

Warning

</div>

Replace `SIDEKIQ_USERNAME`

</div>

``` console
[isabell@sidekiq ~]$ psql -U SIDEKIQ_USERNAME -d postgres
psql (12.4)
Type "help" for help.

postgres=#
```

You should see the postgresql command prompt. You can exit with `CTRL-D`
or by typing `\q` and hitting the `ENTER` key. But first list the
current database users and verify that everything's alright

``` psql
postgres=# \du
                                  List of roles
Role name |                         Attributes                         | Member of
----------+------------------------------------------------------------+-----------
<username>| Superuser, Create role, Create DB, Replication, Bypass RLS | {}
postgres=#
```

where `<username>` in the `Role name` column should be your
`SIDEKIQ_USERNAME`. For the next step stay in the prompt.

#### Set up the superuser password

<div class="warning">

<div class="title">

Warning

</div>

Replace `SIDEKIQ_USERNAME` and `POSTGRESQL_SUPERUSER_PASSWORD`

</div>

``` psql
postgres=# alter user SIDEKIQ_USERNAME with password 'POSTGRESQL_SUPERUSER_PASSWORD';
postgres=# \q
[isabell@sidekiq ~]$
```

Back in the bash prompt we'll now harden the
`$HOME/.local/var/postgresql/pg_hba.conf` file

<div class="warning">

<div class="title">

Warning

</div>

Replace `SIDEKIQ_USERNAME`

</div>

``` kconfig
# TYPE  DATABASE        USER            ADDRESS                 METHOD

local   all             SIDEKIQ_USERNAME                        scram-sha-256
# ... other configuration values we don't need to change
```

Every change to the `$HOME/.local/var/postgresql/pg_hba.conf` or
`$HOME/.local/var/postgresql/postgresql.conf` file needs a restart of
the `postgresql` supervisor service.

``` console
[isabell@sidekiq ~]$ supervisorctl restart postgresql
postgresql: stopped
postgresql: started
[isabell@sidekiq ~]$ supervisorctl status postgresql
postgresql                       RUNNING   pid 28577, uptime 0:00:05
[isabell@sidekiq ~]$
```

Let's try to connect to the database again as superuser this time with
password

<div class="warning">

<div class="title">

Warning

</div>

Replace `SIDEKIQ_USERNAME`

</div>

``` console
[isabell@sidekiq ~]$ psql -U SIDEKIQ_USERNAME -d postgres
psql (12.4)
Type "help" for help.

postgres=#
```

If you see the psql prompt without error messages then everything's
fine. Stay in the prompt for the next step.

#### Create the GitLab Database

Supposed you're still connected as your user to the `postgres` database
we now setup the `gitlab` database user and database. The actual
production database and user should be different from the superuser to
limit its power as far as possible.

<div class="warning">

<div class="title">

Warning

</div>

Replace `POSTGRESQL_GITLAB_PASSWORD` with a password different from the
`POSTGRESQL_SUPERUSER_PASSWORD`. (See the [Passwords](#passwords)
section.)

</div>

``` psql
postgres=# \c template1
You are now connected to database "template1" as user "<sidekiq_username>".
template1=# CREATE USER gitlab WITH PASSWORD 'POSTGRESQL_GITLAB_PASSWORD' CREATEDB;
CREATE ROLE
template1=# CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION
template1=# CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE EXTENSION
template1=# CREATE DATABASE gitlab OWNER gitlab;
CREATE DATABASE
template1=# \q
```

where `<sidekiq_username>` in the output of the first command matches
your `SIDEKIQ_USERNAME`. Now add the password from above to the
`$HOME/.pgpass` file on the `sidekiq` and `gitlab` host which should now
like like

<div class="warning">

<div class="title">

Warning

</div>

Replace `SIDEKIQ_USERNAME`, `POSTGRESQL_SUPERUSER_PASSWORD` and
`POSTGRESQL_GITLAB_PASSWORD`

</div>

``` console
*:*:*:SIDEKIQ_USERNAME:POSTGRESQL_SUPERUSER_PASSWORD
*:*:gitlab:gitlab:POSTGRESQL_GITLAB_PASSWORD
```

Try to connect to the `gitlab` database as `gitlab` user

``` console
[isabell@sidekiq ~]$ psql -U gitlab -d gitlab
psql (12.4)
Type "help" for help.

gitlab=#
```

and check that the extensions are enabled

``` psql
gitlab=# SELECT true AS enabled
gitlab=# FROM pg_available_extensions
gitlab=# WHERE name = 'pg_trgm'
gitlab=# AND installed_version IS NOT NULL;
 enabled
---------
 t
(1 row)

gitlab=# SELECT true AS enabled
gitlab=# FROM pg_available_extensions
gitlab=# WHERE name = 'btree_gist'
gitlab=# AND installed_version IS NOT NULL;
 enabled
---------
 t
(1 row)

gitlab=# \q
```

The enabled column should contain a row with `t`. The extensions are
required by [GitLab](https://gitlab.com) 13.1+.

#### Check PostgreSQL connection from the gitlab host

With all the environment set in the `$HOME/.bashrc` and the database
password stored in `$HOME/.pgpass` it's just one command away to connect
to the database from the `gitlab` host as gitlab user

``` console
[isabell@gitlab ~]$ psql
psql (12.4)
Type "help" for help.

gitlab=> \q
[isabell@gitlab ~]$
```

### Redis

Redis is installed on uberspace hosts so just quickly check that the
redis version is `>=6.x`

``` console
[isabell@gitlab ~]$ redis-server --version
Redis server v=6.0.8 sha=00000000:0 malloc=jemalloc-5.1.0 bits=64 build=611051ae86733216
[isabell@gitlab ~]$
```

For an in-depth guide see the `Redis <guide_redis>` lab guide. Here in
short. We need redis to accept outside connections for `sidekiq` in
addition to its socket so let's aquire a port for it. I'll refer to it
as `REDIS_PORT`

``` console
[isabell@gitlab ~]$ uberspace port add
Port 66666 will be open for TCP and UDP traffic in a few minutes.
[isabell@gitlab ~]$
```

We further need the bind url

``` console
[isabell@gitlab ~]$ /sbin/ifconfig | grep -A1 veth
veth_isabell: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
    inet 100.64.4.52  netmask 255.255.255.252  broadcast 0.0.0.0
[isabell@gitlab ~]$
```

Look out for the `inet` key in the last line and write the ip down. The
`veth_isabell` line will look different from yours and the ip in the
case above would be `100.64.4.52`. I'll refer to it as `GITLAB_VETH_IP`

Now create the redis directory

``` console
[isabell@gitlab ~]$ mkdir "$HOME/.redis"
[isabell@gitlab ~]$
```

and the `$HOME/.redis/conf` file in it

<div class="warning">

<div class="title">

Warning

</div>

Replace `GITLAB_VETH_IP`, `REDIS_PORT`, `GITLAB_USERNAME` and
`REDIS_PASSWORD` with a secure password. (See [Passwords](#passwords)).
`REDIS_SOCKET` is put together like `/home/GITLAB_USERNAME/.redis/sock`
(replace `GITLAB_USERNAME`).

</div>

``` ruby
bind GITLAB_VETH_IP
port REDIS_PORT
unixsocket REDIS_SOCKET
requirepass REDIS_PASSWORD
daemonize no
```

Setup the damon and create `$HOME/etc/services.d/redis.ini` with the
following content:

``` ini
[program:redis]
command=redis-server %(ENV_HOME)s/.redis/conf
autostart=yes
autorestart=yes
redirect_stderr=true
stdout_logfile=%(ENV_HOME)s/logs/redis.log
```

Let's start redis

``` console
[isabell@gitlab ~]$ supervisorctl reread
redis: available
[isabell@gitlab ~]$ supervisorctl update redis
redis: added process group
[isabell@gitlab ~]$ supervisorctl status redis
redis                            RUNNING   pid 18520, uptime 0:00:06
[isabell@gitlab ~]$
```

and eventually check that redis works as expected (you'll end up in a
redis prompt)

<div class="note">

<div class="title">

Note

</div>

Do this from your `sidekiq` host and `gitlab` host. On the `gitlab` host
you don't need to specify the `REDIS_PORT` and `GITLAB_FQDN` since you
can connect via the redis socket. A `$ redis-cli -s REDIS_SOCKET` will
do that.

</div>

<div class="warning">

<div class="title">

Warning

</div>

Replace `REDIS_PORT`, `GITLAB_FQDN` and `REDIS_PASSWORD`

</div>

``` console
[isabell@sidekiq ~]$ redis-cli -p REDIS_PORT -h GITLAB_FQDN
host:port> ping
(error) NOAUTH Authentication required.
host:port> auth REDIS_PASSWORD
OK
host:port> ping
PONG
host:port> quit
[isabell@sidekiq ~]$
```

`host` should match `GITLAB_FQDN` and `port` the `REDIS_PORT`. If your
prompt looks like that after typing all commands you're all set. If not
you should go through the [Redis](#redis) section again.

Installation GitLab
-------------------

<div class="note">

<div class="title">

Note

</div>

This guide is tested with [GitLab](https://gitlab.com) 13.4.2.

</div>

Pull the source

``` console
[isabell@gitlab ~]$ git clone https://gitlab.com/gitlab-org/gitlab-foss.git -b 13-4-stable gitlab
[isabell@gitlab ~]$ cd gitlab
[isabell@gitlab ~/gitlab]$ git checkout v13.4.2
[isabell@gitlab ~/gitlab]$
```

### Configuration

Change the current directory to the location of the configuration files

``` console
[isabell@gitlab ~/gitlab]$ cd config
[isabell@gitlab ~/gitlab/config]$
```

and copy the example files

``` console
[isabell@gitlab ~/gitlab/config]$ cp gitlab.yml.example gitlab.yml

[isabell@gitlab ~/gitlab/config]$ cp secrets.yml.example secrets.yml
[isabell@gitlab ~/gitlab/config]$ chmod 0600 secrets.yml

[isabell@gitlab ~/gitlab/config]$ cp puma.rb.example puma.rb

[isabell@gitlab ~/gitlab/config]$ cp resque.yml.example resque.yml
[isabell@gitlab ~/gitlab/config]$ chmod 0600 resque.yml

[isabell@gitlab ~/gitlab/config]$ cp database.yml.postgresql database.yml
[isabell@gitlab ~/gitlab/config]$ chmod 0600 database.yml

[isabell@gitlab ~/gitlab/config]$ cp initializers/smtp_settings.rb.sample initializers/smtp_settings.rb
[isabell@gitlab ~/gitlab/config]$ chmod 0600 initializers/smtp_settings.rb
[isabell@gitlab ~/gitlab/config]$
```

We need to change all occurences of `/home/git/` to your actual home
`/home/GITLAB_USERNAME/`

<div class="warning">

<div class="title">

Warning

</div>

Replace `GITLAB_USERNAME`

</div>

``` console
[isabell@gitlab ~/gitlab/config]$ sed -i 's:/home/git/:/home/GITLAB_USERNAME/:g' gitlab.yml puma.rb resque.yml database.yml
[isabell@gitlab ~/gitlab/config]$
```

Edit `$HOME/gitlab/config/gitlab.yml` to match the following (only
required changes are listed)

<div class="warning">

<div class="title">

Warning

</div>

Replace `GITLAB_USERNAME`, `GITLAB_EXTERNAL_FQDN`

The file is big so you may need to scroll down a lot to get to the
configuration key.

</div>

``` yaml
production: &base

    # ... line 30
    gitlab:
        host: GITLAB_EXTERNAL_FQDN
        port: 443
        https: true

        # ... line 84
        # Uncomment and customize if you can't use the default user to run GitLab (default: 'git')
        user: GITLAB_USERNAME

        # ... line 94
        email_enabled: true
        email_from: GITLAB_USERNAME@uber.space
        email_display_name: GitLab
        email_reply_to: GITLAB_USERNAME@uber.space

    # ... line 1026
    repositories:
        # ... line 1031
        storages:
            default:
                path: /home/GITLAB_USERNAME/repositories/
                gitaly_address: unix:/home/GITLAB_USERNAME/gitlab/tmp/sockets/private/gitaly.socket

    # ...
```

Edit `$HOME/gitlab/config/puma.rb` to match the following (only required
changes are listed)

<div class="warning">

<div class="title">

Warning

</div>

Replace `GITLAB_USERNAME`

</div>

``` ruby
# ... line 19
threads 1, 16

# ... line 33
bind 'unix:///home/GITLAB_USERNAME/gitlab/tmp/sockets/gitlab.socket'

workers 1

# ... line 69
worker_timeout 100
```

Setting the `workers` count to 1 limits the RAM usage of puma. The main
puma process and one puma worker use up to 600MB each. This sums up to
around 1-1.2GB and leaves some headroom but not enough for another
worker.

Edit `$HOME/gitlab/config/resque.yml` to match the following (only
required changes are listed)

<div class="warning">

<div class="title">

Warning

</div>

Replace `GITLAB_USERNAME` and `REDIS_PASSWORD`

</div>

``` yaml
# ... line 14
production:
    # Redis (single instance)
    url: unix:/home/GITLAB_USERNAME/.redis.sock
    password: REDIS_PASSWORD

# ...
```

Edit `$HOME/gitlab/config/database.yml` to match the following (only
required changes are listed)

<div class="warning">

<div class="title">

Warning

</div>

Replace `POSTGRESQL_PORT`, `SIDEKIQ_FQDN` and
`POSTGRESQL_GITLAB_PASSWORD`

</div>

``` yaml
#
# PRODUCTION
#
production:
  adapter: postgresql
  encoding: unicode
  database: gitlab
  username: gitlab
  password: POSTGRESQL_GITLAB_PASSWORD
  host: SIDEKIQ_FQDN
  port: POSTGRESQL_PORT
  # ... some settings commented out to setup a load balancer

# ... other environments we don't need to change
```

Edit `$HOME/gitlab/config/initializers/smtp_settings.rb` to match the
following

<div class="warning">

<div class="title">

Warning

</div>

Replace `GITLAB_FQDN`, `GITLAB_USERNAME`, `GITLAB_EXTERNAL_FQDN` and
`GITLAB_EMAIL_PASSWORD`. The `GITLAB_EMAIL_PASSWORD` is the one you've
set in the [uberspace
dashboard](https://dashboard.uberspace.de/dashboard/authentication)
section `SSH ACCESS TO YOUR UBERSPACE`.

</div>

``` ruby
# ... line 9
if Rails.env.production?
Rails.application.config.action_mailer.delivery_method = :smtp

ActionMailer::Base.delivery_method = :smtp
ActionMailer::Base.smtp_settings = {
  address: "GITLAB_FQDN",
  port: 587,
  user_name: "GITLAB_USERNAME@uber.space",
  password: "GITLAB_EMAIL_PASSWORD",
  domain: "GITLAB_EXTERNAL_FQDN",
  authentication: :plain,
  enable_starttls_auto: true,
  tls: false,
  ssl: false,
  openssl_verify_mode: 'none' # See ActionMailer documentation for other possible options
  }
end
```

Puma suffers from memory leaks and the memory consumption increases over
time. As a counter measurement gitlab uses puma worker killer but the
default memory limit is too high for uberspace. Edit
`$HOME/gitlab/lib/gitlab/cluster/puma_worker_killer_initializer.rb` to
match the following

``` ruby
# ... scroll until here
module Gitlab
  module Cluster
    class PumaWorkerKillerInitializer
      def self.start(
        puma_options,
          puma_per_worker_max_memory_mb: 750,
          puma_master_max_memory_mb: 550,
          additional_puma_dev_max_memory_mb: 200
    )

# ... not important
```

You will notice higher loading times the moment the puma worker is
restarted and you're currently browsing. From my experience the restart
happens once or twice the hour but I haven't noticed it very often. Feel
free to adjust the `puma_per_worker_max_memory_mb` value to something
you feel comfortable with but don't stress the overall memory limit.

### Directories

Ensure that the basic directories exist and have the correct permissions

:

``` console
[isabell@gitlab ~]$ cd gitlab
[isabell@gitlab ~/gitlab]$ chmod -R 0770 log/

[isabell@gitlab ~/gitlab]$ chmod -R 0770 tmp/
[isabell@gitlab ~/gitlab]$ chmod -R 0770 tmp/pids/
[isabell@gitlab ~/gitlab]$ chmod -R 0770 tmp/sockets/
[isabell@gitlab ~/gitlab]$ chmod 0700 tmp/sockets/private

[isabell@gitlab ~/gitlab]$ mkdir -p public/uploads
[isabell@gitlab ~/gitlab]$ chmod -R 0700 public/uploads

[isabell@gitlab ~/gitlab]$ chmod -R 0770 builds/

[isabell@gitlab ~/gitlab]$ chmod -R 0770 shared/artifacts
[isabell@gitlab ~/gitlab]$ chmod -R 0770 shared/pages/
[isabell@gitlab ~/gitlab]$
```

### Git

Configure git

``` console
[isabell@gitlab ~]$ git config --global core.autocrlf input
[isabell@gitlab ~]$ git config --global gc.auto 0
[isabell@gitlab ~]$ git config --global repack.writeBitmaps true
[isabell@gitlab ~]$ git config --global receive.advertisePushOptions true
[isabell@gitlab ~]$ git config --global core.fsyncObjectFiles true
[isabell@gitlab ~]$
```

### Install Gems

This step fails if the bundler build configuration hasn't the correct
`re2` path. Double check the [Ruby](#ruby) section.

``` console
[isabell@gitlab ~]$ cd gitlab
[isabell@gitlab ~/gitlab]$ bundle install --deployment --without development test mysql aws kerberos
[isabell@gitlab ~/gitlab]$
```

### Install GitLab Shell

``` console
[isabell@gitlab ~/gitlab]$ bundle exec rake gitlab:shell:install RAILS_ENV=production
[isabell@gitlab ~/gitlab]$
```

The gitlab-shell configuration is auto generated from the configuration
values above but it doesn't harm to double check. This is what the
`$HOME/gitlab-shell/config.yml` configuration file should look like

<div class="warning">

<div class="title">

Warning

</div>

Replace `GITLAB_USERNAME` and `GITLAB_EXTERNAL_FQDN`

</div>

``` yaml
---
user: GITLAB_USERNAME
gitlab_url: "https://GITLAB_EXTERNAL_FQDN"
http_settings:
    ca_path: '/home/GITLAB_USERNAME/etc/certificates'
    self_signed_cert: false
auth_file: '/home/GITLAB_USERNAME/.ssh/authorized_keys'
log_level: INFO
audit_usernames: false
```

### Install GitLab Workhorse

Under normal cirumstances the nginx weberver communicates over a socket
with gitlab-workhorse but since we don't have the option to install a
nginx configuration file to set this up and apache doesn't support
reverse proxying to sockets we use a tcp port on the loopback interface.
Make sure that port 8181 isn't occupied by another processes. I'll refer
to it as `GITLAB_WORKHORSE_PORT`.

``` console
[isabell@gitlab ~]$ netstat -tulpn | grep '\b8181\b' || echo nope
nope
[isabell@gitlab ~]$
```

If there is a process occupying the port just choose another one that is
above `1024`. Connections only operating on the loopback interface don't
need to be unlocked by the firewall with `uberspace port add`.

<div class="warning">

<div class="title">

Warning

</div>

Replace `GITLAB_USERNAME`

</div>

``` console
[isabell@gitlab ~/gitlab]$ bundle exec rake "gitlab:workhorse:install[/home/GITLAB_USERNAME/gitlab-workhorse]" RAILS_ENV=production
[isabell@gitlab ~/gitlab]$
```

The gitlab-workhorse needs a supervisor service in
`$HOME/etc/services.d/workhorse.ini`

<div class="warning">

<div class="title">

Warning

</div>

Replace `GITLAB_WORKHORSE_PORT` on the marked line

</div>

``` ini
[program:workhorse]
directory=%(ENV_HOME)s/gitlab
command=
    %(ENV_HOME)s/gitlab-workhorse/gitlab-workhorse
    -listenUmask 0
    -listenNetwork tcp
    -listenAddr 0.0.0.0:GITLAB_WORKHORSE_PORT
    -authBackend http://127.0.0.1:9292
    -authSocket %(ENV_HOME)s/gitlab/tmp/sockets/gitlab.socket
    -documentRoot %(ENV_HOME)s/gitlab/public
environment=
    PERL5LIB="%(ENV_HOME)s/.local/lib/perl5",
    LD_LIBRARY_PATH="%(ENV_HOME)s/.local/lib:%(ENV_HOME)s/.local/postgresql/lib:/opt/rh/devtoolset-9/root/usr/lib64:/opt/rh/devtoolset-9/root/usr/lib:/opt/rh/devtoolset-9/root/usr/lib64/dyninst:/opt/rh/devtoolset-9/root/usr/lib/dyninst",
    PATH="%(ENV_HOME)s/gitlab-workhorse:%(ENV_HOME)s/.local/sbin:%(ENV_HOME)s/.local/bin:%(ENV_HOME)s/bin:/opt/uberspace/etc/%(ENV_USER)s/binpaths/ruby:/opt/rh/devtoolset-9/root/usr/bin:%(ENV_HOME)s/.cargo/bin:%(ENV_HOME)s/.luarocks/bin:%(ENV_HOME)s/go/bin:%(ENV_HOME)s/.deno/bin:%(ENV_HOME)s/.config/composer/vendor/bin:/bin:/usr/bin:/usr/ucb:/usr/local/bin:%(ENV_HOME)s/.dotnet/tools"
redirect_stderr=true
stdout_logfile=%(ENV_HOME)s/gitlab/log/gitlab-workhorse.log
```

The `-authBackend` settings isn't needed actually because the
`-authSocket` takes precedence but if you would like to change the
method `gitlab-workhorse` and `puma` communicate then change it to the
port number `puma` is listening on. The default is `9292`.

Reread the supervisor configuration but do **not** start
`gitlab-workhorse` yet.

``` console
[isabell@gitlab ~]$ supervisorctl reread
[isabell@gitlab ~]$
```

### Install Gitaly

<div class="warning">

<div class="title">

Warning

</div>

Replace `GITLAB_USERNAME`

</div>

``` console
[isabell@gitlab ~/gitlab]$ bundle exec rake "gitlab:gitaly:install[/home/GITLAB_USERNAME/gitaly,/home/GITLAB_USERNAME/repositories]" RAILS_ENV=production
[isabell@gitlab ~/gitlab]$
```

The `sidekiq` host needs to communicate with gitaly and we need another
open port. The last one. I'll refer to it with `GITALY_PORT`.

``` console
[isabell@gitlab ~]$ uberspace port add
Port 77777 will be open for TCP and UDP traffic in a few minutes.
[isabell@gitlab ~]$
```

Now let's check the auto-generated gitaly configuration file
`$HOME/gitaly/config.toml` and add or change configuration options to
match the following:

<div class="warning">

<div class="title">

Warning

</div>

Replace `GITLAB_USERNAME`, `GITLAB_VETH_IP`, `GITALY_PORT` and
`GITLAB_EXTERNAL_FQDN`

</div>

``` toml
bin_dir = "/home/GITLAB_USERNAME/gitaly"
internal_socket_dir = "/home/GITLAB_USERNAME/gitaly/internal_sockets"
socket_path = "/home/GITLAB_USERNAME/gitlab/tmp/sockets/private/gitaly.socket"
listen_addr = "GITLAB_VETH_IP:GITALY_PORT"
[gitaly-ruby]
dir = "/home/GITLAB_USERNAME/gitaly/ruby"
[gitlab]
url = "https://GITLAB_EXTERNAL_FQDN"
[gitlab-shell]
dir = "/home/GITLAB_USERNAME/gitlab-shell"
[[storage]]
name = "default"
path = "/home/GITLAB_USERNAME/repositories"
```

Setup the supervisor service and create the file in
`$HOME/etc/services.d/gitaly.ini`

``` ini
[program:gitaly]
directory=%(ENV_HOME)s/gitlab
command=nohup %(ENV_HOME)s/gitaly/gitaly %(ENV_HOME)s/gitaly/config.toml
environment=
    LD_LIBRARY_PATH="%(ENV_HOME)s/.local/lib:%(ENV_HOME)s/.local/postgresql/lib:/opt/rh/devtoolset-9/root/usr/lib64:/opt/rh/devtoolset-9/root/usr/lib:/opt/rh/devtoolset-9/root/usr/lib64/dyninst:/opt/rh/devtoolset-9/root/usr/lib/dyninst",
    PATH="%(ENV_HOME)s/.local/sbin:%(ENV_HOME)s/.local/bin:%(ENV_HOME)s/bin:/opt/uberspace/etc/%(ENV_USER)s/binpaths/ruby:/opt/rh/devtoolset-9/root/usr/bin:%(ENV_HOME)s/.cargo/bin:%(ENV_HOME)s/.luarocks/bin:%(ENV_HOME)s/go/bin:%(ENV_HOME)s/.deno/bin:%(ENV_HOME)s/.config/composer/vendor/bin:/bin:/usr/bin:/usr/ucb:/usr/local/bin:%(ENV_HOME)s/.dotnet/tools"
autostart=yes
autorestart=yes
redirect_stderr=true
stdout_logfile=%(ENV_HOME)s/gitlab/log/gitaly.log
```

Gitaly must be running for the next section to work

:

``` console
[isabell@gitlab ~]$ supervisorctl reread
[isabell@gitlab ~]$ supervisorctl update gitaly
[isabell@gitlab ~]$ supervisorctl status gitaly
[isabell@gitlab ~]$
```

### Initialize Database

<div class="warning">

<div class="title">

Warning

</div>

The command below with all given flags is destructive! Only use it if
you need to setup the database the first time or if you know what you're
doing!

</div>

``` console
[isabell@gitlab ~]$ cd gitlab
[isabell@gitlab ~/gitlab]$ bundle exec rake gitlab:setup RAILS_ENV=production DISABLE_DATABASE_ENVIRONMENT_CHECK=1 force=yes
[isabell@gitlab ~/gitlab]$ # It takes a while to complete. Do not abort even if the process looks dead.
```

`DISABLE_DATABASE_ENVIRONMENT_CHECK=1` lets you drop the production
database and `force=yes` disables the interactive questions if you
really want to do that.

### Post installation steps

Stop the gitaly process for the moment

``` console
[isabell@gitlab ~]$ supervisorctl stop gitaly
[isabell@gitlab ~]$
```

and check the application status

``` console
[isabell@gitlab ~]$ cd gitlab
[isabell@gitlab ~/gitlab]$ bundle exec rake gitlab:env:info RAILS_ENV=production
System information
System:
Current User:   <username>
Using RVM:      no
Ruby Version:   2.6.6p146
Gem Version:    3.1.4
Bundler Version:1.17.3
Rake Version:   12.3.3
Redis Version:  6.0.8
Git Version:    2.24.3
Sidekiq Version:5.2.9
Go Version:     go1.15.1 linux/amd64

GitLab information
Version:        13.4.2
Revision:       b08b36dccc3
Directory:      /home/<username>/gitlab
DB Adapter:     PostgreSQL
DB Version:     12.4
URL:            https://<username>.uber.space
HTTP Clone URL: https://<username>.uber.space/some-group/some-project.git
SSH Clone URL:  <username>@<username>.uber.space:some-group/some-project.git
Using LDAP:     no
Using Omniauth: yes
Omniauth Providers:

GitLab Shell
Version:        13.7.0
Repository storage paths:
1. default:      /home/<username>/repositories
GitLab Shell path:              /home/<username>/gitlab-shell
Git:            /usr/bin/git
[isabell@gitlab ~/gitlab]$
```

Your output should look similar to this but `<username>` replaced with
your `GITLAB_USERNAME`.

If you're doing this on your `sidekiq` host during the [Postinstallation
(sidekiq)](#postinstallation-sidekiq) step then you'll see this output.

<div class="note">

<div class="title">

Note

</div>

You don't need to do this step on your `sidekiq` host if you haven't
started [Installation sidekiq](#installation-sidekiq) yet

</div>

``` console
[isabell@sidekiq ~]$ cd gitlab
[isabell@sidekiq ~/gitlab]$ bundle exec rake gitlab:env:info RAILS_ENV=production
WARNING: This version of GitLab depends on gitlab-shell 13.7.0, but you're running Unknown. Please update gitlab-shell.

System information
System:
Current User:   <sidekiq_username>
Using RVM:      no
Ruby Version:   2.6.6p146
Gem Version:    3.1.4
Bundler Version:1.17.3
Rake Version:   12.3.3
Redis Version:  6.0.8
Git Version:    2.24.3
Sidekiq Version:5.2.9
Go Version:     go1.15.1 linux/amd64

GitLab information
Version:        13.4.2
Revision:       b08b36dccc3
Directory:      /home/<sidekiq_username>/gitlab
DB Adapter:     PostgreSQL
DB Version:     12.4
URL:            https://<gitlab_username>.uber.space
HTTP Clone URL: https://<gitlab_username>.uber.space/some-group/some-project.git
SSH Clone URL:  <gitlab_username>@<gitlab_username>.uber.space:some-group/some-project.git
Using LDAP:     no
Using Omniauth: yes
Omniauth Providers:

GitLab Shell
Version:        unknown
Repository storage paths:
1. default:      /home/<gitlab_username>/repositories
GitLab Shell path:              /home/<sidekiq_usernam>/gitlab-shell
Git:            /usr/bin/git
```

where `<sidekiq_username>` matches your `SIDEKIQ_USERNAME` and
`<gitlab_username>` your `GITLAB_USERNAME`.

### Compile GetText PO files

``` console
[isabell@gitlab ~]$ cd gitlab
[isabell@gitlab ~/gitlab]$ bundle exec rake gettext:compile RAILS_ENV=production
[isabell@gitlab ~/gitlab]$
```

### Install yarn packages

``` console
[isabell@gitlab ~]$ cd gitlab
[isabell@gitlab ~/gitlab]$ yarn install --production --pure-lockfile
[isabell@gitlab ~/gitlab]$
```

### Compile assets

This step hasn't worked for me on my uberspace host because the
`webpack` process uses too much RAM and gets killed in the middle. The
solution is to compile the assets locally on your home PC.

Make sure you have the minimum `node` and `yarn` versions installed
otherwise install them with your package manager. You'll also need cmake
`>= 3.x` and the libre2-dev package. The package name may differ
depending on your OS. If something goes wrong this is most likely
because you're missing some dependencies. Check the dependencies section
at [GitLab installation from source
dependencies](https://docs.gitlab.com/13.4/ee/install/installation.html#1-packages-and-dependencies).
Next we go through some of the steps again to be able to compile the
assets locally. It's not that much anymore like before.

<div class="warning">

<div class="title">

Warning

</div>

Checkout the same gitlab version on your local host than the one on your
`gitlab` host.

</div>

``` console
[isabell@home ~]$ mkdir -p workspace/
[isabell@home ~/workspace]$ cd workspace/
[isabell@home ~/workspace]$ git clone https://gitlab.com/gitlab-org/gitlab-foss.git -b 13-4-stable gitlab
[isabell@home ~/workspace]$ cd gitlab
[isabell@home ~/workspace/gitlab]$ gem install bundler --version '>=1.5.2, < 2'
[isabell@home ~/workspace/gitlab]$ git checkout v13.4.2
[isabell@home ~/workspace/gitlab]$ bundle install --deployment --without development test mysql aws kerberos
[isabell@home ~/workspace/gitlab]$
```

For the next step to work you'll need some configuration files from your
`gitlab` host. Download them for example with

<div class="warning">

<div class="title">

Warning

</div>

Replace `GITLAB_USERNAME` and `GITLAB_FQDN`

</div>

``` console
[isabell@home ~/workspace/gitlab]$ scp GITLAB_USERNAME@GITLAB_FQDN:'gitlab/config/{database.yml,gitlab.yml,secrets.yml}' config
[isabell@home ~/workspace/gitlab]$
```

Now compile the assets

``` console
[isabell@home ~/workspace/gitlab]$ bundle exec rake gettext:compile RAILS_ENV=production
[isabell@home ~/workspace/gitlab]$ yarn install --production --pure-lockfile
[isabell@home ~/workspace/gitlab]$ bundle exec rake gitlab:assets:compile RAILS_ENV=production NODE_ENV=production
[isabell@home ~/workspace/gitlab]$
```

This may take a while to complete but in the next step we can upload the
assets to the `gitlab` host. You can either upload it directly or
compress them first like described here

<div class="warning">

<div class="title">

Warning

</div>

Replace `GITLAB_USERNAME` and `GITLAB_FQDN`

</div>

``` console
[isabell@home ~/workspace/gitlab]$ cd public
[isabell@home ~/workspace/gitlab/public]$ tar czf assets.tar.gz assets/
[isabell@home ~/workspace/gitlab/public]$ scp assets.tar.gz GITLAB_USERNAME@GITLAB_FQDN:gitlab/public
[isabell@home ~/workspace/gitlab/public]$
```

On your `gitlab` host

``` console
[isabell@gitlab ~]$ cd gitlab/public
[isabell@gitlab ~/gitlab/public]$ rm -rf assets/ # Deletes the assets directory if it exists
[isabell@gitlab ~/gitlab/public]$ tar xzf assets.tar.gz
[isabell@gitlab ~/gitlab/public]$ rm assets.tar.gz
[isabell@gitlab ~/gitlab/public]$
```

Do not delete anything yet in your home directory of your home PC. We'll
need the assets and hash files on the `sidekiq` host too, but it's not
ready yet.

To finish the compilation we'll enter the rails console

``` console
[isabell@gitlab ~]$ cd gitlab
[isabell@gitlab ~/gitlab]$ RAILS_ENV=production NODE_ENV=production bundle exec rails console -e production
--------------------------------------------------------------------------------
GitLab:       13.4.2 (b08b36dccc3) FOSS
GitLab Shell: 13.7.0
PostgreSQL:   12.4
--------------------------------------------------------------------------------
Loading production environment (Rails 6.0.3.1)
irb(main):001:0>
```

Within the console we transform po to json files, precompile the assets,
generate the md5 hash in `assets-hash.txt`, fix urls and finally check
for side effects

``` ruby
irb(main):001:0> Rails.application.load_tasks
# a lot of output
irb(main):002:0> Rake::Task['yarn:check'].invoke
warning Resolution field "ts-jest@24.0.0" is incompatible with requested version "ts-jest@^23.10.5"
warning Resolution field "monaco-editor@0.20.0" is incompatible with requested version "monaco-yaml#monaco-editor@^0.19.2"
warning Resolution field "chokidar@3.4.0" is incompatible with requested version "watchpack#watchpack-chokidar2#chokidar@^2.1.8"
=> [#<Proc:0x0000000007f58460@/home/<username>/gitlab/lib/tasks/yarn.rake:16>]
irb(main):003:0> Rake::Task['gettext:po_to_json'].invoke
# Creates the app.js files in /home/<username>/gitlab/app/assets/javascripts/locale/**
irb(main):004:0> Rake::Task['rake:assets:precompile'].invoke
# Does some yarn stuff. Ignore warnings
irb(main):005:0> File.write('assets-hash.txt', Tasks::Gitlab::Assets.md5_of_assets_impacting_webpack_compilation)
irb(main):006:0> Rake::Task['gitlab:assets:fix_urls'].invoke
# Fixes urls
irb(main):007:0> Rake::Task['gitlab:assets:check_page_bundle_mixins_css_for_sideeffects'].invoke
Checking public/assets/page_bundles/_mixins_and_variables_and_functions-e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855.css for side effects
The file does not introduce any side effects, we are all good.
=> [#<Proc:0x000000001e0c6278@/home/<username>/gitlab/lib/tasks/gitlab/assets.rake:133>]
irb(main):008:0> quit
```

where `<username>` in the output matches your `GITLAB_USERNAME`.

### Finish installation on the gitlab host

Set up the last service needed on the `gitlab` host, which starts the
puma webserver. Edit `$HOME/etc/services.d/webserver.ini`

``` ini
[program:webserver]
directory=%(ENV_HOME)s/gitlab
command=%(ENV_HOME)s/gitlab/bin/web start_foreground
environment=
    RAILS_ENV="production",
    USE_WEB_SERVER="puma",
    LD_LIBRARY_PATH="%(ENV_HOME)s/.local/lib:%(ENV_HOME)s/.local/postgresql/lib:/opt/rh/devtoolset-9/root/usr/lib64:/opt/rh/devtoolset-9/root/usr/lib:/opt/rh/devtoolset-9/root/usr/lib64/dyninst:/opt/rh/devtoolset-9/root/usr/lib/dyninst",
    PATH="%(ENV_HOME)s/.local/sbin:%(ENV_HOME)s/.local/bin:%(ENV_HOME)s/bin:/opt/uberspace/etc/%(ENV_USER)s/binpaths/ruby:/opt/rh/devtoolset-9/root/usr/bin:%(ENV_HOME)s/.cargo/bin:%(ENV_HOME)s/.luarocks/bin:%(ENV_HOME)s/go/bin:%(ENV_HOME)s/.deno/bin:%(ENV_HOME)s/.config/composer/vendor/bin:/bin:/usr/bin:/usr/ucb:/usr/local/bin:%(ENV_HOME)s/.dotnet/tools"
autostart=yes
autorestart=yes
redirect_stderr=true
stdout_logfile=%(ENV_HOME)s/gitlab/log/gitaly.log
```

We don't need to start it yet and just load the configuration

``` console
[isabell@gitlab ~]$ supervisorctl reread
[isabell@gitlab ~]$
```

As a last point on the `gitlab` host check that all services are
available

``` console
[isabell@gitlab ~]$ supervisorctl avail
gitaly                           in use   auto      999:999
redis                            in use   auto      999:999
webserver                        avail    auto      999:999
workhorse                        avail    auto      999:999
[isabell@gitlab ~]$
```

If all services are listed we're good.

Installation sidekiq
--------------------

### Prerequisites

Have your `POSTGRESQL_PORT`, `REDIS_PORT` and `GITALY_PORT` ready. You
should also know the `GITLAB_FQDN`.

### Install dependencies

You'll need go through some of the steps for the `gitlab` host again.
These are

1.  [Ruby](#ruby)
2.  [Re2](#re2)
3.  [Runit](#runit)
4.  [Node](#node)

Don't forget to adjust your `$HOME/.bashrc` on the `sidekiq` host to
include all `PATH`'s and the `LD_LIBRARY_PATH` adjustments.

### Install sidekiq

The steps are pretty much the same like in the [Installation
gitlab](#installation-gitlab) section and mainly differ in the
configuration. For details look there, we're going through all commands
quickly here

<div class="warning">

<div class="title">

Warning

</div>

Checkout the same version like on the `gitlab` host

</div>

``` console
[isabell@sidekiq ~]$ git clone https://gitlab.com/gitlab-org/gitlab-foss.git -b 13-4-stable gitlab
[isabell@sidekiq ~]$ cd gitlab
[isabell@sidekiq ~/gitlab]$ git checkout v13.4.2
[isabell@sidekiq ~/gitlab]$ cd config

[isabell@sidekiq ~/gitlab/config]$ cp gitlab.yml.example gitlab.yml

[isabell@sidekiq ~/gitlab/config]$ cp resque.yml.example resque.yml
[isabell@sidekiq ~/gitlab/config]$ chmod 0600 resque.yml

[isabell@sidekiq ~/gitlab/config]$ cp database.yml.postgresql database.yml
[isabell@sidekiq ~/gitlab/config]$ chmod 0600 database.yml
[isabell@sidekiq ~/gitlab/config]$
```

We also need some files from the `gitlab` host. Copy them over either
directly or your local PC as intermediate. These files are (relative to
the `/home/GITLAB_USERNAME/gitlab` directory)

``` none
config/secrets.yml
config/initializers/smtp_settings.rb
.gitlab_kas_secret
.gitlab_shell_secret
.gitlab_workhorse_secret
```

Secure access to these files with

``` console
[isabell@gitlab ~]$ cd gitlab
[isabell@sidekiq ~/gitlab]$ chmod 0600 config/{secrets.yml,initializers/smtp_settings.rb} .gitlab_{kas,shell,workhorse}_secret
[isabell@sidekiq ~/gitlab]$
```

Next change all occurences of `/home/git/` to `/home/SIDEKIQ_USERNAME/`

<div class="warning">

<div class="title">

Warning

</div>

Replace `SIDEKIQ_USERNAME`

</div>

``` console
[isabell@sidekiq ~/gitlab/config]$ sed -i 's:/home/git/:/home/SIDEKIQ_USERNAME/:g' gitlab.yml
[isabell@sidekiq ~/gitlab/config]$
```

Edit `$HOME/gitlab/config/gitlab.yml` to match the following (only
required changes are listed)

<div class="warning">

<div class="title">

Warning

</div>

Replace `GITLAB_USERNAME`, `GITLAB_EXTERNAL_FQDN`, `GITLAB_FQDN` and
`GITALY_PORT`

</div>

``` yaml
production: &base

    # ... line 31
    gitlab:
        host: GITLAB_EXTERNAL_FQDN
        port: 443
        https: true

        # ... line 84
        # Uncomment and customize if you can't use the default user to run GitLab (default: 'git')
        user: GITLAB_USERNAME

        # ... line 94
        email_enabled: true
        email_from: GITLAB_USERNAME@uber.space
        email_display_name: GitLab
        email_reply_to: GITLAB_USERNAME@uber.space

    # ... line 1026
    repositories:
        # ... line 1031
        storages:
            default:
                path: /home/GITLAB_USERNAME/repositories/
                gitaly_address: tcp://GITLAB_FQDN:GITALY_PORT
```

Edit `$HOME/gitlab/config/resque.yml` to match the following (only
required changes are listed)

<div class="warning">

<div class="title">

Warning

</div>

Replace `REDIS_PASSWORD`, `GITLAB_FQDN` and `REDIS_PORT`. Notice the
colon `:` in front of the `REDIS_PASSWORD` which is necessary

</div>

``` yaml
# ... line 14
production:
    # Redis (single instance)
    url: redis://:REDIS_PASSWORD@GITLAB_FQDN:REDIS_PORT
```

Edit `$HOME/gitlab/config/database.yml` to match the following

<div class="warning">

<div class="title">

Warning

</div>

Replace `POSTGRESQL_GITLAB_PASSWORD`, `SIDEKIQ_USERNAME` and
`POSTGRESQL_PORT`

</div>

``` yaml
#
# PRODUCTION
#
production:
  adapter: postgresql
  encoding: unicode
  database: gitlab
  username: gitlab
  password: 'POSTGRESQL_GITLAB_PASSWORD'
  host: /home/SIDEKIQ_USERNAME/.local/tmp/
  port: POSTGRESQL_PORT

 # ...
```

#### Install Gems (sidekiq)

``` console
[isabell@sidekiq ~]$ cd gitlab
[isabell@sidekiq ~/gitlab]$ bundle install --deployment --without development test mysql aws kerberos
[isabell@sidekiq ~/gitlab]$
```

#### Postinstallation (sidekiq)

Repeat every step from the [Post installation
steps](#post-installation-steps) section. You do not need to compile the
assets again on your home PC, just upload the `assets.tar.gz` file this
time to the `sidekiq` host, uncompress the archive and finish the
compilation in the rails console. Stay in the rails for the next step.

#### Verify email configuration

If not already in the rails console enter it with

``` console
[isabell@sidekiq ~]$ cd gitlab
[isabell@sidekiq ~/gitlab]$ bundle exec rails console -e production
--------------------------------------------------------------------------------
GitLab:       13.4.2 (5049b3cd236) FOSS
GitLab Shell: Unknown
PostgreSQL:   12.4
--------------------------------------------------------------------------------
Loading production environment (Rails 6.0.3.1)
irb(main):001:0>
```

Within the console we'll verify the email settings and finally send a
test email

<div class="warning">

<div class="title">

Warning

</div>

Replace `TEST_EMAIL_ACCOUNT` with a valid email address in the form
<test@example.com> and you have access to.

</div>

``` ruby
irb(main):001:0> ActionMailer::Base.delivery_method
=> :smtp
irb(main):002:0> ActionMailer::Base.smtp_settings
=> {:address=>"<gitlab_fqdn>", :port=>587, :user_name=>"<username>@uber.space", :password=>"<password>", :domain=>"<username>.uber.space", :authentication=>:plain, :enable_starttls_auto=>true, :tls=>false, :ssl=>false, :openssl_verify_mode=>"none"}
irb(main):003:0> Notify.test_email('TEST_EMAIL_ACCOUNT', 'Test', 'This is a test message').deliver_now
```

where `<gitlab_fqdn>` matches your `GITLAB_FQDN`, `<password>` your
`GITLAB_EMAIL_PASSWORD` and `<username>` your `GITLAB_USERNAME`. If
there are no exceptions you should have received an email in your
`TEST_EMAIL_ACCOUNT`'s inbox. If not check your junk mail inbox just to
be sure. In the case you haven't received an email you should go through
the `$HOME/gitlab/config/initializers/smtp_settings.rb` again (See also
[Configuration](#configuration)).

#### Setup the sidekiq supervisor daemon

Now that installation is done we need a supervisor service for sidekiq
in `$HOME/etc/services.d/sidekiq.ini`

``` ini
[program:sidekiq]
directory=%(ENV_HOME)s/gitlab
command=%(ENV_HOME)s/gitlab/bin/background_jobs start_foreground
environment=
    RAILS_ENV="production",
    SIDEKIQ_WORKERS="1",
    LD_LIBRARY_PATH="%(ENV_HOME)s/.local/lib:%(ENV_HOME)s/.local/postgresql/lib:/opt/rh/devtoolset-9/root/usr/lib64:/opt/rh/devtoolset-9/root/usr/lib:/opt/rh/devtoolset-9/root/usr/lib64/dyninst:/opt/rh/devtoolset-9/root/usr/lib/dyninst",
    PATH="%(ENV_HOME)s/.local/sbin:%(ENV_HOME)s/.local/bin:%(ENV_HOME)s/bin:/opt/uberspace/etc/%(ENV_USER)s/binpaths/ruby:/opt/rh/devtoolset-9/root/usr/bin:%(ENV_HOME)s/.cargo/bin:%(ENV_HOME)s/.luarocks/bin:%(ENV_HOME)s/go/bin:%(ENV_HOME)s/.deno/bin:%(ENV_HOME)s/.config/composer/vendor/bin:/bin:/usr/bin:/usr/ucb:/usr/local/bin:%(ENV_HOME)s/.dotnet/tools"
autostart=yes
autorestart=yes
redirect_stderr=yes
stdout_logfile=%(ENV_HOME)s/gitlab/log/sidekiq.log
```

The `SIDEKIQ_WORKERS=1` setting ensures to run only one worker which
uses around 500-600MB and leaves enough headroom to run another app on
the same host, so your `sidekiq` host doesn't need to be dedicated to
sidekiq alone.

Final steps
-----------

Reread the configuration and start sidekiq

``` console
[isabell@sidekiq ~]$ supervisorctl reread
sidekiq: available
[isabell@sidekiq ~]$ supervisorctl update sidekiq
sidekiq: added process group
[isabell@sidekiq ~]$ supervisorctl status
postgresql                       RUNNING   pid 6821, uptime 18:25:57
sidekiq                          RUNNING   pid 11634, uptime 0:00:29
[isabell@sidekiq ~]$
```

Now change to your `gitlab` host and start all the services

``` console
[isabell@sidekiq ~]$ supervisorctl reread
webserver: available
workhorse: available
[isabell@sidekiq ~]$ supervisorctl update
webserver: added process group
workhorse: added process group
[isabell@sidekiq ~]$ supervisorctl start gitaly
gitaly: started
[isabell@sidekiq ~]$ supervisorctl status
gitaly                           RUNNING   pid 31321  uptime 0:00:05
redis                            RUNNING   pid 18520, uptime 20:09:37
webserver                        RUNNING   pid 32227, uptime 0:00:06
workhorse                        RUNNING   pid 32229, uptime 0:00:06
[isabell@sidekiq ~]$
```

### Web Backend

Configure the web backend to use the `GITLAB_WORKHORSE_PORT` on your
`gitlab` host

<div class="warning">

<div class="title">

Warning

</div>

Replace `GITLAB_WORKHORSE_PORT`

</div>

``` console
[isabell@gitlab ~]$ uberspace web backend set / --remove-prefix --http --port GITLAB_WORKHORSE_PORT
[isabell@gitlab ~]$
```

Double check the backend

``` console
[isabell@gitlab ~]$ uberspace web backend list
/ http:<port>, --remove-prefix => OK, listening: PID 21435, /home/<username>/gitlab-workhorse/gitlab-workhorse -listenUmask 0 -listenNetwork tcp -listenAddr 0.0.0.0:<port> -authBackend http://127.0.0.1:9292 -authSocket /home/<username>/gitlab/tmp/sockets/gitlab.socket -documentRoot /home/<username>/gitlab/public
[isabell@gitlab ~]$
```

your output should look similar to the one above where `<port>` is to
your `GITLAB_WORKHORSE_PORT` and `<username>` is your `GITLAB_USERNAME`.

### Double Check the application status

Run a thourough check of your `gitlab` instance

``` console
[isabell@gitlab ~]$ cd gitlab
[isabell@gitlab ~/gitlab]$ bundle exec rake gitlab:check RAILS_ENV=production
Checking Incoming Email ...
Checking GitLab subtasks ...

Checking GitLab Shell ...

GitLab Shell: ... GitLab Shell version >= 13.7.0 ? ... OK (13.7.0)
Running /home/<username>/gitlab-shell/bin/check
Internal API available: OK
Redis available via internal API: OK
gitlab-shell self-check successful

Checking GitLab Shell ... Finished

Checking Gitaly ...

Gitaly: ... default ... OK

Checking Gitaly ... Finished

Checking Sidekiq ...

Sidekiq: ... Running? ... no
  Try fixing it:
  sudo -u <username> -H RAILS_ENV=production bin/background_jobs start
  For more information see:
  doc/install/installation.md in section "Install Init Script"
  see log/sidekiq.log for possible errors
  Please fix the error above and rerun the checks.

Checking Sidekiq ... Finished
Incoming Email: ... Reply by email is disabled in config/gitlab.yml

Checking Incoming Email ... Finished

Checking LDAP ...

LDAP: ... LDAP is disabled in config/gitlab.yml

Checking LDAP ... Finished

Checking GitLab App ...

Git configured correctly? ... yes
Database config exists? ... yes
All migrations up? ... yes
Database contains orphaned GroupMembers? ... no
GitLab config exists? ... yes
GitLab config up to date? ... yes
Log directory writable? ... yes
Tmp directory writable? ... yes
Uploads directory exists? ... yes
Uploads directory has correct permissions? ... yes
Uploads directory tmp has correct permissions? ... skipped (no tmp uploads folder yet)
Init script exists? ... no
  Try fixing it:
  Install the init script
  For more information see:
  doc/install/installation.md in section "Install Init Script"
  Please fix the error above and rerun the checks.
Init script up-to-date? ... can't check because of previous errors
Projects have namespace: ...
GitLab Instance / Monitoring ... yes
Redis version >= 4.0.0? ... yes
Ruby version >= 2.5.3 ? ... yes (2.6.6)
Git version >= 2.24.0 ? ... yes (2.24.3)
Git user has default SSH configuration? ... yes
Active users: ... 1
Is authorized keys file accessible? ... yes
GitLab configured to store new projects in hashed storage? ... yes
All projects are in hashed storage? ... yes

Checking GitLab App ... Finished


Checking GitLab subtasks ... Finished
[isabell@gitlab ~/gitlab]$
```

where `<username>` is your `GITLAB_USERNAME`. You should see only two
errors which we can ignore. `sidekiq` isn't running because we've set it
up on a different host and the init script is replaced by our supervisor
services.

### Finish it

Point your browser to `https://GITLAB_EXTERNAL_FQDN`, follow the
instructions and reset the password. You can login with `root` as user
and your new password.

Done :)

#### Cleanup

Finally we can cleanup our home a bit to gain some disk space back.
Remove the entire workspace directory and some of the cache. You can do
so on both hosts.

``` console
[isabell@gitlab ~]$ rm -rfI $HOME/workspace/
[isabell@gitlab ~]$ rm -rfI $HOME/.cache/{yarn,go-build}
[isabell@gitlab ~]$
```

Check your quota

``` console
[isabell@gitlab ~]$ quota -gsl
[isabell@gitlab ~]$
```

you should have left around 4-5GB for your repositories.

Troubleshooting
---------------

### General

Whenever you encounter errors you should check the log file(s) of the
process which cause the headaches. supervisor related logs are saved in
`$HOME/logs`. Inspect them if you encounter problems with the `redis` or
`postgresql` services. `gitlab` services log files are located in the
`$HOME/gitlab/log` directories

-   **webserver**: `$HOME/gitlab/log/puma.stderr.log` and
    `$HOME/gitlab/log/puma.stdout.log`
-   **gitaly**: `$HOME/gitlab/log/gitaly.log`
-   **sidekiq**: `$HOME/gitlab/log/sidekiq.log` for the service itself
    (startup, activites and errors) and
    `$HOME/gitlab/log/sidekiq_client.log` for client related activities.
-   **gitlab-workhorse**: `$HOME/gitlab/log/gitlab-workhorse.log`

You can also visit the official [Issue
Board](https://gitlab.com/gitlab-org/gitlab-foss/-/issues) and search
there for answers.

### X errors prohibited this user from being saved

If you encounter this error message the first time you visit
<https://GITLAB_EXTERNAL_FQDN> after you've entered the new root
password something went wrong during the creation of the `root` user. To
fix this you need to enter the rails console on your `gitlab` host.

``` console
[isabell@gitlab ~]$ cd gitlab
[isabell@gitlab ~]$ bundle exec rails console -e production
irb(main):001:0>
```

In the console type the following commands

<div class="warning">

<div class="title">

Warning

</div>

Replace `ROOT_PASSWORD` with a secure password (See
[Passwords](#passwords))

</div>

``` ruby
irb(main):001:0> user = User.where(id: 1).first
irb(main):002:0> user.password = 'ROOT_PASSWORD'
=> "<root_password>"
irb(main):003:0> user.password_confirmation = 'ROOT_PASSWORD'
=> "<root_password>"
irb(main):004:0> user.save!
Enqueued ActionMailer::MailDeliveryJob (Job ID: ab6ad2bf-cd1a-40de-a3fd-6287c3be4a9a) to Sidekiq(mailers) with arguments: "DeviseMailer", "password_change", "deliver_now", {:args=>[#<GlobalID:0x000000001805b5f0 @uri=#<URI::GID gid://gitlab/User/1>>]}
=> true
irb(main):005:0> quit
```

where `<root_password>` in the output matches your `ROOT_PASSWORD`.

Best Practices
--------------

### Backup

<div class="warning">

<div class="title">

Warning

</div>

GitLab's backup system is not compatible with uberspace when it comes to
the `$HOME/.ssh/authorized_keys` file. After a restore and you accepted
to restore the `authorized_keys` keys file your ssh key and the
uberspace ssh key are gone!! You have to take care of the
`authorized_keys` file yourself and decline the restoration of the
`authorized_key` file when you're asked to do so during a
`bundle exec rake gitlab:backup:restore`.

</div>

You should create an initial backup and create backups on a regular
basis. To create a backup it's best to shut down everything what
accesses the database besides `gitaly`. On your `gitlab` host

``` console
[isabell@gitlab ~]$ supervisorctl stop webserver workhorse
[isabell@gitlab ~]$
```

and on your `sidekiq` host

``` console
[isabell@sidekiq ~]$ supervisorctl stop sidekiq
[isabell@sidekiq ~]$
```

Then back on your `gitlab` host

``` console
[isabell@gitlab ~]$ cd gitlab
[isabell@gitlab ~/gitlab]$ bundle exec rake gitlab:backup:create RAILS_ENV=production
[isabell@gitlab ~/gitlab]$
```

You should also save all configuration files we've created during the
installation from both hosts. Besides these I recommend backing up the
secret files `$HOME/gitlab/.gitlab_*_secret`. Also back up the
`$HOME/.ssh/authorized_keys` file from your `gitlab` host and everything
else you think is worthy to be saved (`~/.bashrc`, `~/.pgpass`, ...)

Store all these files in a safe place for example on your home machine.

**Footnotes**

------------------------------------------------------------------------

Tested with GitLab 13.4.2, Uberspace 7.7.9.0

[1] <https://en.wikipedia.org/wiki/GitLab>
