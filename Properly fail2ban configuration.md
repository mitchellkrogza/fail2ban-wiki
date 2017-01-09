If not reconfigured, Fail2ban will load configuration files from directory `/etc/fail2ban`. You can find there many files called `*.conf`.<br/>
Before you start fail2ban service, you should do some configurations appropriate to your system. At least to enable jails that you want to protect with fail2ban.

**[Q]** Should I make my configuration directly in `jail.conf` and `fail2ban.conf`?<br/>
**[A]** No. You should avoid to change `.conf` files, created by fail2ban installation.

Since this files may be overwritten by package upgrades, or because your changes may be incompatible with some future versions, you shouldn't edit it in-place.<br/>
Instead, you'll write a new file having `.local` extension. For example any values defined in `jail.local` will override those in `jail.conf` in the same sections (e. g. `[DEFAULT]`).

So for example if original `.conf` file contains:
```
[DEFAULT]
log = /path/to/log

[section1]
log = /other/path
enabled = true

[section2]
enabled = true
```
And you'll create a `.local` file contains:
```
[DEFAULT]
log = /my-path/to/log
```
The value of parameter `log` in `section1` will be still `/other/path`.<br/>
But value of parameter `log` in `section2` will be changed to `/my-path/to/log` (because it was not specified in section self, and new default value will be used).

**[Q]** Which configurations are necessary to let fail2ban protect a service?<br/>
**[A]** You should create a `jail.local` file and at least enable there corresponding jails (all jails are disabled by default) resp. overwrite there all the settings you've different from normally stock installation, or even create your own jails (and/or) filters, that are not available in default configuration of the fail2ban distribution.

For example if you'll, that fail2ban should ban authorization failures occurred in sshd and nginx, but the `error.log` of your your nginx-instance is configured as `/var/log/my-nginx/error.log` you should set also parameter `logpath` additionally to `enabled` in section `[nginx]`.

So your `jail.local` looks like:
```
[nginx]
logpath = /var/log/my-nginx/error.log
enabled = true

[sshd]
enabled = true
```

If you use another version of fail2ban as provided from maintainers of your distribution, you should check another parameters (that may be normally specified in some distribution config files), like:
- several path-parameters of fail2ban service self (specified in `fail2ban.conf` or includes):
```
[Definition]
logtarget = /var/log/fail2ban.log
socket =    /var/run/fail2ban/fail2ban.sock
pidfile =   /var/run/fail2ban/fail2ban.pid
dbfile =    /var/run/fail2ban/fail2ban.sqlite3
```
- other jail parameters (`jail.conf` or includes) like `backend` (e. g. usage of systemd journals expected `systemd` backend), `action` resp. `banaction` (e. g. you can't use `iptables` if your system does not support it), `logpath`, etc.

You can also control resp. configure another optional configurations parameters, like `ignoreip`, etc.

**[Q]** How I can see the current (merged) configuration, that fail2ban will use by start<br/>
**[A]** You can dump your current configuration (all the parameters that fail2ban loads by start) with following commands:
```bash
# dump parameters:
fail2ban-client -d
# verbose: output config files will be loaded and dump parameters:
fail2ban-client -vd
fail2ban-client -vvd
```

**[Q]** How I can notify fail2ban, that the configuration was changed<br/>
**[A]** You should execute `fail2ban-client reload` (in previous versions before 0.10 `fail2ban-client restart`).<br/> 

You can also get and set corresponding parameter individually, using fail2ban client-server communication protocol. For example:
```bash
fail2ban-client set pam-generic logencoding UTF-8
fail2ban-client set nginx findtime 10m
```

**[Q]** How should I correctly modify log file locations other than in the jail settings or messing with master .conf files?<br/>
**[A]** To make a modification to the default log file locations you should create a .local file of paths-common.conf or paths-debian.com (whichever you are using in jail.local) and make changes only in your .local files which keeps it nicely structured for your jail(s) settings and avoids problems when Fail2Ban is updated<br/><br/>

To create your .local file<br/>
`sudo cp /etc/fail2ban/paths-common.conf /etc/fail2ban/paths-common.local`<br/><br/>

Now if you want for example an Nginx filter to read all your Nginx Access Logs for multiple web sites<br/>

Instead of using in your jail:<br/>
`logpath = /var/log/nginx/*access*.log`<br/><br/>

Edit the line in paths-common.local or paths-debian.local (whichever you are using) and add change the nginx_access_log line as follows<br/>
`nginx_access_log = /var/log/nginx/*access*.log`<br/><br/>

Then in your jail you would rather use<br/>
`logpath = %(nginx_access_log)s`<br/><br/>


**[Q]** I messed up Fail2Ban during Testing and blocked out my own IP address, how do I completely reset Fail2Ban to get it off to a clean start?<br/>
**[A]** To reset fail2ban completely and start off fresh<br/><br/>
Stop Fail2Ban<br/>
`sudo service fail2ban stop`<br/><br/>
Empty the Fail2Ban LogFile<br/>
`sudo truncate -s 0 /var/log/fail2ban.log`<br/><br/>
Delete the Fail2Ban SQLite Database File<br/>
`sudo rm /var/lib/fail2ban/fail2ban.sqlite3`<br/><br/>
Restart Fail2Ban<br/>
`sudo service fail2ban restart`<br/><br/>
Also consider deleting any of your Apache, Nginx or Auth log files or just the entries that may contain your own IP address used during testing, as once Fail2Ban starts again, depending on your jail settings, it will just block you again.<br/><br/>

**[Q]** Fail2Ban will not start and is giving me the following error message "Job for fail2ban.service failed. See 'systemctl status fail2ban.service' and 'journalctl -xn' for details." but checking those does not help me trace where my error is.<br/>
**[A]** <br/>
Stop the Failban Server by running<br/>
`sudo service fail2ban stop`<br/><br/>
Make sure the Fail2Ban client is also not running by running the following<br/>
`sudo fail2ban-client -vvv -x stop`<br/><br/>
Then start the Fail2Ban client in verbose mode as follows<br/>
`sudo fail2ban-client -vvv -x start`<br/><br/>
This will show you exactly in which jail, filter or action your error lies.
Once you can start the fail2ban-client successfully using `sudo fail2ban-client -vvv -x start`<br/><br/>
Then stop it again using <br/>
`sudo fail2ban-client -vvv -x stop`<br/><br/>
and then start the Fail2Ban Server<br/>
`sudo service fail2ban restart`<br/><br/>