# zrep-expire
Zrep snapshot expiration tool

This is a python script to help with the retention and expiration of snapshots that were created with [zrep](http://www.bolthole.com/solaris/zrep/).

It can take a [configuration file](zrep-expire.conf), in which the retention schedule can be specified in a cron-like table.

The script is meant to be run from cron, like this:

    55 6 * * * /usr/local/bin/zrep-expire -c /etc/zfs/zrep-expire.conf

Its workings are simple:

1. It gets a list of all existing ZFS snapshots and removes the most recent one from the list, so it will never expire the last remaining snapshot.
2. It matches the time of snapshot creation against the expiration rules in the configuration.
3. If the snapshot should be expired according to the rules, it tries to destroy the snapshot, unless the custom local property 'zrep:sent' does not contain an integer value, which means that the snapshot was not created by zrep or something else went wrong.

At this time, zrep-expire does not consider diffent filesystems, so it operates on all snapshots of all filesystems at once. This means you cannot have different expiration rules for different filesystems. It also means that the fact that 'it will never expire the last remaining snapshot' should be taken quite literally. If at some point there is a demand for filesystem awareness in zrep-expire, please create an issue with a feature request, or implement it yourself and send a me a pull request :-)

## Configuration

The configuration file should contain a list of expiration rule, each consisting of 6 fields, resembling a crontab entry:

    # MIN HR         DOM  MON  DOW  EXPIRATION

meaning minutes, hours, day of month, month, day of week and expiration rule.

The first 5 fields define the time to match the snapshot creation time with. Each field can take an integer value, a range (x-y), a list of values and/or ranges (a,x,y-z) or a literal '*', which means 'any'.

For day-of-week, '1' means 'Sunday', '2' Monday, etc.

Expiration rules should specify a number and a unit, where unit should be one of 'years', 'months', 'weeks', 'days', 'hours', 'minutes', 'seconds', 'microseconds'.

Lines starting with a '#' are considered comments and will be ignored.

Please see the [example configuration](zrep-expire.conf) for some example rules. They should be self-explanatory.

Please note that rules are matched from bottom to top, so put the most specific ones at the bottom and the default at the top.
