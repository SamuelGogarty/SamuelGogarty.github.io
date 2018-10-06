Here I will be writing my notes for creating a service script in FreeBSD from scratch, generally used for server applications.

Usually you don't need to write service scripts for server applications, as any mainstream server applications such as NGINX, Apache httpd and any other daemon such as an IRCd or SSHd that are installed through the operating systems package manager would have this script automatically installed.

Although for server applications that are not distributed through the operating system's package manager usually, due to license issues or just not being submitted in the first place, this service script will likely not be present, but it could easily be sought after as a easy way to check on the applications status as well as an off/on switch for the service among other things that we will visit later on. The example I will be using here is going to be murmur. Murmur is an open source voice-over-ip server, made as the server component for mumble.

All of FreeBSD's service scripts are stored in the /etc/rc.d/ directory. All service scripts must be written in sh.

On FreeBSD, murmur can be installed via the binary package manager pkg, or through ports.

here we will be installing with pkg as shown below:

```markdown
root@freebsdvm: pkg install murmur
```

Simply running murmur with:
```markdown
root@freebsdvm: murmurd
```
We will start the murmur server with the default config as a background process. But it is often desirable that we have service control. Let's start with our script, let's name a script "mumbled" inside /etc/rc.d/.
```markdown
root@freebsdvm: ee /etc/rc.d/mumbled
```
The bare bones of the script for basic functionality looks like this:
```markdown
#!/bin/sh

. /etc/rc.subr

name=murmurd
rcvar=murmurd_enable

command="/usr/local/sbin/${name}"

load_rc_config $name
run_rc_command "$1"
```
With this script, the daemon will automatically be started on boot up, and you will be able to issue the "start" "stop" and "status" commands. Each has a self explanatory name, the syntax looks like this:
```markdown
root@freebsdvm:/etc/rc.d$ service mumbled stop #stops the daemon
Stopping murmurd.
Waiting for PIDS: 662.
```
```markdown
root@freebsdvm:/etc/rc.d$ service mumbled start #starts the daemon
Starting murmurd.
```
```markdown
root@freebsdvm:/etc/rc.d$ service mumbled status #shows the process ID
murmurd is running as pid 803.
```markdown
Although this is script is bare-bones when it comes to the applicable uses for a service script. Since it is just sh with imported functions, you can get pretty creative on what this script will run. For example a lot of people like to start their server applications that have a command line interpreter to be inside screen. Screen is a terminal multiplexer often used to contain the command line interpreter inside a re-attachable session, so to not be lost when the ssh pipe is broken.

In this VM, I have screen installed, let's edit the script so when the service starts on boot up, it is started in screen.

We will have to start murmurd with the foreground argument -fg, as seen here:
```markdown
root@freebsdvm:~$ murmurd -fg #opens murmurd as a foreground proc, doesnt fork.
<W>2018-09-16 06:52:59.535 Initializing settings from /root/.murmurd/murmur.ini (basepath /root/.murmurd)
<W>2018-09-16 06:52:59.536 Meta: TLS cipher preference is "ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:AES256-SHA:AES128-SHA"
<W>2018-09-16 06:52:59.536 OpenSSL: OpenSSL 1.0.2o-freebsd  27 Mar 2018
<C>2018-09-16 06:52:59.536 WARNING: You are running murmurd as root, without setting a uname in the ini file. This might be a security risk.
<W>2018-09-16 06:52:59.549 ServerDB: Opened SQLite database /root/murmur.sqlite
<W>2018-09-16 06:52:59.550 Murmur 1.2.19 (Compiled Sep  2 2018 04:08:26) running on X11: FreeBSD 11.2-RELEASE: Booting servers
<W>2018-09-16 06:52:59.555 1 => Server listening on https://www.linkedin.com/redir/invalid-link-page?url=0%2e0%2e0%2e0:64738
<W>2018-09-16 06:52:59.558 1 => Announcing server via bonjour
<W>2018-09-16 06:53:02.720 1 => Not registering server as public
```
Let's edit the service script so it is started in foreground by default, inside of screen.

```markdown
#!/bin/sh

. /etc/rc.subr

name=murmurd
rcvar=murmurd_enable

screen_start()
{
        /usr/local/bin/screen -dmS mumbled #starts a blank screen session
        /usr/local/bin/screen -S mumbled -X stuff 'murmurd -fg\n' #runs murmurd in foreground inside the screen session
}

daemon_stop()
{
        /usr/bin/killall murmurd #kills processes and sub processes ran by murmurd
        /usr/local/bin/screen -X -S mumbled quit #closes the screensession created for murmurd
}

command="/usr/local/sbin/murmurd"
start_cmd=screen_start
stop_cmd=daemon_stop

load_rc_config $name
run_rc_command "$1"
```

Here we can see that I override the default stop and start commands imported from rc.subr with my own functions, since we need to do more then just kill and start murmurd itself. In the screen_start() function we are starting a screen session in detached mode and then naming that session "mumbled". The next line in the function passes a command to that session without attaching, it passes the command "murmurd -fg" into the session, which runs murmurd in the foreground rather than forking it into the background.

In the daemon_stop() function, we replace the default stop command. The first line kills murmurd with the killall command, the second line kills the screen session so it is not lingering after the daemon has been killed.

The practical use of this for this specific server application is for logging, although obviously there are superior ways to log the murmurd process, like simply just writing all daemon output to a log file, this is to just show you that /etc/rc.d/ scripting is flexible and its limits are only the limits that reside in sh itself.

An example where running screen around your application is more useful is for an application which has its own command interpreter in runtime, something like a quake server comes to mind where you could run commands from the dedicated server's console.

Anyway that will conclude my notes for writing a service script in FreeBSD from scratch.




