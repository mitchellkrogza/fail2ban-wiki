**[Q]** Can I ban something other as host (IP-address), like user or e-mail, etc.<br/>
**[A]** Yes, it is theoretically possible with fail2ban, since no-host banning was implemented (v. 0.9.5 or 0.10). See [fail2ban/gh-1454](https://github.com/fail2ban/fail2ban/pull/1454) for more implementation details.

* You should create your own `action` and specify there which command(s) should be executed by ban/unban
* Version 0.10 allows you to define failure-ID in `failregex`:
  - use `<F-ID/>` for failure-ID as no-space tag (equivalent to `(?P<fid>\S+)`), example:
  ```
  failregex = ^authentication failure; login=<F-ID/>
  ```
  - use `<F-ID>...</F-ID>` for own regex contains failure-ID (equivalent to `(?P<fid>...)`), example:
  ```
  failregex = ^authentication failure; login=<F-ID>[^@]+@\S+</F-ID/>
  ```
* In version 0.9 you should use `(?P<host>...)` to define failure-ID and implicitly reset all host-related features (e. g. dns resolving) for this jail, so define `usedns = raw`, `ignoreip =`, `ignorecommand =`

Example for test jail to ban users, config `jail.local`:
```bash
[test]
# don't use dns, because host group is not hostname (and not resolvable ip):
usedns = raw
ignoreip =
ignorecommand =
# if used some filter:
#filter = no-host-filter[HOST=(?P<host>\S+)]
#
# if used failregex:
filter =
# v. 0.10:
failregex = ^\s*(?:\S+\s+)?(?:[^:]+:auth\[\d+\]:\s+)?pam_unix(?:\(\S+\))?:?\s+authentication failure; login=<F-ID/>
# action that bans users
banaction = test-ban-user[name=%(__name__)s]
#
logpath  = %(syslog_authpriv)s
enabled = true
```
For fail2ban version 0.9, you should define `failregex` like below:
```bash
[test]
...
# v. 0.9.5:
failregex = ^\s*(?:\S+\s+)?(?:[^:]+:auth\[\d+\]:\s+)?pam_unix(?:\(\S+\))?:?\s+authentication failure; login=(?P<host>\S+)
```

Action config file `action.d/test-ban-user.local`:
``` bash
[Definition]
actionstart = 
actionstop =
actioncheck =
actionban = echo 'ban f2b-<name> --user <ip>'
actionunban = echo 'unban f2b-<name> --user <ip>'
```

To test, the user "xxx" will be banned, just execute following commands (3 times if `maxretry = 3` for this jail):
``` bash
logger -t 'test:auth' -i -p auth.info "pam_unix(test:auth): authentication failure; login=xxx"
logger -t 'test:auth' -i -p auth.info "pam_unix(test:auth): authentication failure; login=xxx"
logger -t 'test:auth' -i -p auth.info "pam_unix(test:auth): authentication failure; login=xxx"
```
This should produce in `/var/log/auth.log` 3 entries like:
``` bash
Nov 14 20:07:35 srv test:auth[3141]: pam_unix(test:auth): authentication failure; login=xxx
```

To test regular expression, use new option `--raw` or `-r`, to prevent dns resolving errors:

``` bash
# v. 0.10:
fail2ban-regex --raw /var/log/auth.log '^\s*(?:\S+\s+)?(?:[^:]+:auth\[\d+\]:\s+)?pam_unix(?:\(\S+\))?:?\s+authentication failure; login=<F-ID/>'
# v. 0.9.5:
fail2ban-regex --raw /var/log/auth.log '^\s*(?:\S+\s+)?(?:[^:]+:auth\[\d+\]:\s+)?pam_unix(?:\(\S+\))?:?\s+authentication failure; login=(?P<host>\S+)'
```

**[Q]** I don't have any failure-ID in the log-entry, can I nevertheless configure the fail2ban, that it should simply execute some command if some message will be found in observed log-file<br/>
**[A]** Yes, if you've no failure-id at all (no user-id, e-mail or something other), but you'll that fail2ban execute some shell script after failure occurrence, you should additionally:
* set empty or something other as match for failure-id (still `<host>` in 0.9th-branch) in `failregex`, example:
``` bash
# DDOS resp. "too many IPs" will be used as failure-ID:
failregex = ^<F-ID>DDOS</F-ID> attack detected$
            ^IDS raises alarm: <F-ID>too many IPs</F-ID> in stack$
```
* set `maxretry = 1` and `findtime = 1` (ban after first occurrence in 1 seconds);
* set small `bantime` (e. g. 1 second) to this "jail" (otherwise no "ban" action will be executed in this time, because "already banned" occurs), e. g. `bantime = 1`
* you need to specify only `actionban` parameter in your custom action file:
```bash
actionban = /user/bin/ids-attack.sh '<fid>'
``` 
* `actionban` script will be executed as root (or with user, fail2ban running), so use `su` if other/restricted user needed;
- set `usedns`, `ignoreip`, `ignorecommand` as suggested above, otherwise you can get error by comparison with empty/illegal host (that will be found by "failure");
