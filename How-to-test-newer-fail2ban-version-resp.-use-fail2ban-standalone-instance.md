Sometimes you'll need to test a fix for some issue, new functionality or some features provided only in the newer version, another different fail2ban configuration, etc.

To test another version of fail2ban, you do not need to install it at all.

You can create a standalone or test instance parallel to your stock fail2ban (fail2ban works basically without installation).

Below you'll find an example, how to create such test instance without installation, e. g. 0.10-th version of fail2ban, regardless of original fail2ban version, that may be installed parallel on the same host:

- download or git checkout a 10th branch, e. g. to directory `/tests/f2b10`:
```bash
git clone --branch 0.10 https://github.com/fail2ban/fail2ban.git /tests/f2b10
```
- enter your new fail2ban directory:
```bash
cd /tests/f2b10
f2b10dir="$(pwd)"
```
- if you will use python3, first execute `./fail2ban-2to3` (don't do that if `python --version` returns 2.x)
- if you want, you can execute the test suite (works only on clean copy, as long as you do not made some customisation, resp. no *.local files was created):
```bash
PYTHONPATH=. ./bin/fail2ban-testcases --fast --no-network
```
- create separate directory `run` for your files this fail2ban instance uses:
```bash
mkdir run
```
- create `./config/fail2ban.local` and write there this another paths:
```
[Definition]
logtarget = /tests/f2b10/run/fail2ban.log
socket = /tests/f2b10/run/fail2ban.sock
pidfile = /tests/f2b10/run/fail2ban.pid
dbfile = /tests/f2b10/run/fail2ban.sqlite3
```
- create `./config/jail.local` and write there your own customizations only, e. g. your backend, enable there your jails, etc.:
```
[DEFAULT]
backend = pyinotify
default_backend = pyinotify
[sshd]
enabled = true
[pam-generic]
enabled = true
```
- copy your fail2ban database (contains last positions inside logs, current bans, etc.):
```bash
cp /var/lib/fail2ban/fail2ban.sqlite3 /tests/f2b10/run/fail2ban.sqlite3
```
- :warning: [important] stop original stock fail2ban `service fail2ban stop` or `fail2ban-client stop`<br/>
Note: If you want to leave your original stock fail2ban running, see warning at end of this article for more informations.

- define alias for your new test fail2ban instance:
```bash
alias f2b10="sudo PYTHONPATH=$f2b10dir $f2b10dir/bin/fail2ban-client -c $f2b10dir/config/"
```
- start your test fail2ban instance:
```bash
#sudo PYTHONPATH=. ./bin/fail2ban-client -c $(pwd)/config/ start
f2b10 start
```
- to stop this fail2ban instance:
```bash
#sudo PYTHONPATH=. ./bin/fail2ban-client -c $(pwd)/config/ stop
f2b10 stop
```
- path for configs `/tests/f2b10/config` (corresponds to `/etc/fail2ban`)
- path for log `/tests/f2b10/run/fail2ban.log`

Easy and without any installation...

**:warning: [Important] you should normally always shutdown your original version before you start this standalone instance and vice versa**
But you can leave your original stock fail2ban running, and try to run this test (standalone) version parallel side by side with original version of fail2ban. In this case you should never use the same jail names as configured in `jail.local` (`jail.conf`) from `/etc/fail2ban/`, because otherwise, through equal name convention, you may get conflict in tables, chains or rules in your system actions (like iptables, etc.)