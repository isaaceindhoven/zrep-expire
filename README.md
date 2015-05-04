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
