# Heroku buildpack for autossh
This is a Heroku buildpack that adds autossh to your heroku build.

autossh is a program to start a copy of ssh and monitor it, restarting
it as necessary should it die or stop passing traffic.
More information about autossh can be found here:
http://www.harding.motd.ca/autossh/

## Usage

```
$ heroku create --buildpack https://github.com/kollegorna/heroku-buildpack-autossh

$ git push heroku master
...

-----> Fetching custom git buildpack... done
-----> autossh app detected
-----> Installing autossh
       Downloading autossh version 1.4f...
       Building autossh...
       Cleaning up...
       Installation successful!
-----> Discovering process types
       Procfile declares types -> (none)

-----> Compressing... done, 28K
-----> Launching... done, v3
```

When the build is complete you will be able to use autossh. It can be tested by doing the following:

```
$ heroku run bash
$ autossh

usage: autossh [-V] [-M monitor_port[:echo_port]] [-f] [SSH_OPTIONS]

    -M specifies monitor port. May be overridden by environment
       variable AUTOSSH_PORT. 0 turns monitoring loop off.
       Alternatively, a port for an echo service on the remote
       machine may be specified. (Normally port 7.)
    -f run in background (autossh handles this, and does not
       pass it to ssh.)
    -V print autossh version and exit.

Environment variables are:
    AUTOSSH_GATETIME    - how long must an ssh session be established
                          before we decide it really was established
                          (in seconds). Default is 30 seconds; use of -f
                          flag sets this to 0.
    AUTOSSH_LOGFILE     - file to log to (default is to use the syslog
                          facility)
    AUTOSSH_LOGLEVEL    - level of log verbosity
    AUTOSSH_MAXLIFETIME - set the maximum time to live (seconds)
    AUTOSSH_MAXSTART    - max times to restart (default is no limit)
    AUTOSSH_MESSAGE     - message to append to echo string (max 64 bytes)
    AUTOSSH_PATH        - path to ssh if not default
    AUTOSSH_PIDFILE     - write pid to this file
    AUTOSSH_POLL        - how often to check the connection (seconds)
    AUTOSSH_FIRST_POLL  - time before first connection check (seconds)
    AUTOSSH_PORT        - port to use for monitor connection
    AUTOSSH_DEBUG       - turn logging to maximum verbosity and log to
                          stderr
```

# Advanced Usage

We will now go through setting up a persistant SSH tunnel from a Heroku build to a service provided on a remote server.

Most probably, you would like a Heroku build with more than just autossh. In this example we will use both the autossh and ruby buildpacks by leveraging Heroku's multi buildpack (https://github.com/heroku/heroku-buildpack-multi).

```
$ heroku create --buildpack https://github.com/heroku/heroku-buildpack-multi.git
```

In your project add a `.buildpacks` file.

```
# .buildpacks
https://github.com/kollegorna/heroku-buildpack-autossh.git
https://github.com/heroku/heroku-buildpack-ruby.git#v138
```

This file informs Heroku to build the project using each of these buildpacks respectively. When pushing up your project you should see the following:

```
$ git push heroku master
...

-----> Fetching custom git buildpack... done
-----> Multipack app detected
=====> Downloading Buildpack: https://github.com/kollegorna/heroku-buildpack-autossh.git
=====> Detected Framework: autossh
-----> Installing autossh
       Downloading autossh version 1.4e...
       Building autossh...
       Cleaning up...
       Installation successful!
=====> Downloading Buildpack: https://github.com/heroku/heroku-buildpack-ruby.git
=====> Detected Framework: Ruby
-----> Compiling Ruby/Rails
-----> Using Ruby version: ruby-2.0.0
...
```

Note: The building and making of autossh can be time consuming. However the binary is then cached. Consecutive builds will skip the download, build and make steps of autossh.

### Setup SSH Keys

Initialization of a SSH tunnel will require that you have the SSH keys that will grant you password free access to the remote server.

Add the keys to heroku as ENV variables:

```
$ heroku config:set HEROKU_PUBLIC_KEY="`cat path/to/public/key`"
$ heroku config:set HEROKU_PRIVATE_KEY="`cat path/to/private/key`"
```

Great! Now we have the necessary keys in place. Time to setup the tunnel.

### Setup SSH tunnel

Create a directory in your project root called '.profile.d'. In that directory we can place any number of shell scripts.
Heroku, during startup, starts a bash shell that runs scripts in the .profile.d/ directory before executing the dyno’s command.

More info about Heroku's .profile.d scripts can be found here: https://devcenter.heroku.com/articles/profiled

For our SSH tunnelling purposes, create a file called `ssh-setup.sh` in directory just created.

```
# .profile.d/ssh-setup.sh

mkdir -p ${HOME}/.ssh
chmod 700 ${HOME}/.ssh

# Copy public key env variable into a file
echo "${HEROKU_PUBLIC_KEY}" > ${HOME}/.ssh/id_rsa.pub
chmod 644 ${HOME}/.ssh/id_rsa.pub

# Copy private key env variable into a file
echo "${HEROKU_PRIVATE_KEY}" > ${HOME}/.ssh/id_rsa
chmod 600 ${HOME}/.ssh/id_rsa

# Auto add the host to known_hosts
# This is to avoid the authenticity of host question that otherwise will halt autossh from setting up the tunnel.
#
# Ex:
# The authenticity of host '[hostname] ([IP address])' can't be established.
# RSA key fingerprint is [fingerprint].
# Are you sure you want to continue connecting (yes/no)?

ssh-keyscan example.com >> ${HOME}/.ssh/known_hosts

# Setup SSH Tunnel
# This is an example of forwarding local port 3306 to the remote servers port 3306 (usually mysql)
#
# autossh will use port 33306 as a monitoring port. If the connection dies autossh will automatically set up a new one.
# ServerAliveInterval: Number of seconds between sending a packet to the server (to keep the connection alive).
# ClientAliveCountMax: Number of above ServerAlive packets before closing the connection. Autossh will create a new connection when this happens.

autossh -M 33306 -f -N -o "ServerAliveInterval 10" -o "ServerAliveCountMax 3"  -L 3306:localhost:3306 user@example.com
```

There you have it. Push your project up to Heroku and now you have a reliable SSH tunnel to a service of your choice hosted remotely.
