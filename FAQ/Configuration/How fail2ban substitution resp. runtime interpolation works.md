For original man-page of fail2ban configuration use `man jail.conf` or e. g. here - https://manned.org/jail.conf.5
How the default basic interpolation in python config files works you can read in the python documentation (e. g. see https://docs.python.org/3/library/configparser.html#supported-ini-file-structure)

Our implementation is more complex (allows includes, recursive interlaced substitution, etc.).

**[Q]** I'm trying to reference some variables in my configuration, but `cftoken = %(cftoken)s` doesn't work

**[A]**
This case `cftoken = %(cftoken)s` is not allowed, because it will produce endless cycle (always substitute itself, again and again).

As well as following is not allowed also (for the same reason):
```
A = %(B)s 
B = %(C)s
C = %(A)s
```
Since we support an extensions called "known" with syntax `%(known/option)s`, after some fix (introduced somewhere 0.9.3 or 0.9.4, but some features possible first in 0.10), it allows to use the same last known option...

So try your luck with `cftoken = %(known/cftoken)s`

If you will set resp. deliver some init parameter from jail to filter or action, you can currently use another interpolation extension of us with another syntax `<option>` (Note: this is a run-time substitution):
```
# action my-action.conf
[Definition]
cftoken = <cftoken>
[Init]
cftoken = default value
```
```
# jail.local
[my-jail]
banaction = my-action[cftoken="customized value"]
```

