# zrep-expire v2
Zrep snapshot expiration tool (v2)

This is a python script to help with the retention and expiration of snapshots that were created with [zrep](http://www.bolthole.com/solaris/zrep/).

    usage: zrep-expire [-h] [-c CONFIG] [-k KEEP] [-v] [-d] [-n] [-a] [--zrep-ignore] [filesystem [filesystem ...]]

    Zrep snapshot expiration tool

    positional arguments:
      filesystem                  the filesystem(s) to act on (default: None)

    optional arguments:
      -h, --help                  show this help message and exit
      -c CONFIG, --config CONFIG  path to configuration file (default: None)
      -k KEEP, --keep KEEP        keep the last n snapshots (default: 5)
      -v, --verbose               output verbose information (default: False)
      -d, --debug                 output even more verbose information (default: False)
      -n, --noop                  list snapshots and results but do not act (default: False)
      -a, --all                   act on all ZFS filesystems (default: False)
      --zrep-ignore               act on all snapshots, not just zrep (default: False)

The script is developed and tested with Python 2.7. It has one external dependency: [*parsedatetime*](https://github.com/bear/parsedatetime). On Debian systems, you can install it with `apt-get install python-parsedatetime` ([package in Testing](https://packages.debian.org/testing/python-parsedatetime)).

The program can take a [configuration file](zrep-expire.conf), in which the retention schedule can be specified.

The default retention configuration is this:

    # MAX_AGE : MIN_INTERVAL
     1 hour   : 1 minute
     12 hours : 5 minutes
     8 days   : 1 hour
     32 days  : 1 day
     1 year   : 1 week
     5 years  : 1 month

The script is meant to be run from cron, like this:

    55 * * * * /usr/local/bin/zrep-expire -c /etc/zfs/zrep-expire.conf -v -a

Its workings are simple:

1. For each ZFS filesystem specified as command line arguments, or for each ZFS filesystem found with `-a`, it gets a list of all existing snapshots. It removes the last `KEEP` snapshots from the list, so they will never be removed.
2. If a snapshot is not created by `zrep`, it will be skipped unless the `--zrep-ignore` option is specified.
3. It matches the creation time of the snapshot against the first column (`MAX_AGE`) of the expiration rule, so see which retention interval should be applied.
4. If the time delta between the current snapshot and the previous one is smaller than `MIN_INTERVAL`, the snapshot is removed.

If one set of expiration rules is sufficient for all of your filesystems, run `zrep-expire` with the `-a` option. If you need different expiration rules for different file systems, run a separate `zrep-expire` for each one, using different configuration files and specifying the filesystem as a command line argument.

The order of the rules in the configuration file is not important. The rules will be sorted and applied in the correct order.

The values in the configuration file will be parsed with [parsedatetime](https://github.com/bear/parsedatetime), where the first column will be used as a negative offset to the current time (to mean 'for the last hour' or 'for the last 8 days'), and the second column will be used as a timedelta to compare a snapshot to the previous one. `parsedatetime` is quite liberal in what it will parse to some meaningful value, so '2h' will work just like '2 hours'. If you really want to get creative (but why would you? we're talking backups here...) you could even put something like `last saturday 20:00` in the left column, and the rule will apply to all snapshots created since that time.

Consider the second rule from the default set: `12 hours : 5 minutes`. The objective of this rule is to keep one restore point per 5 minutes for the current working day. Except that 'the last 12 hours' isn't *exactly* the current working day. You could replace `12 hours` with `7am` and it would be what you expect. Just keep in mind that snaphots will pile up until the next occurrence of '7am', and after that, the whole bunch will fall under the next rule, all at once. So if the next rule is `8 days : 1 hour` like in the default set, the first `zrep-expire` after 7:00 AM will reduce the entire set of the day before from one snapshot per 5 minutes to one snapshot per hour. This means that up to 24 * 11 = 264 snapshots will be removed in one run (for each applicable filesystem!), which could take a long time. This may be perfectly fine, but using a moving window of 12 hours might be just as effective and more predictable. Your call.

# Differences with version 1

The current version of the script (let's call it v2) is written from scratch, because the cron-like configuration of v1 didn't fit my current use case anymore. The cron-like retention scheme required you to know the exact times when snapshots are created, or you had to use a time range for each rule, but that didn't work well when multiple/many snapshots exist in a small time frame. In other words, there was no way to say that out of 3 snapshots made in a certain time frame (which could be 1 minute!) you only want to keep one.

The new scheme is much simpler. You just specify what the minimum time between snapshots should be for a specific time frame, and `zrep-expire` removes all snapshots that are redundant. For example, if you create 3 snapshots per minute and you tell `zrep-expire` you want a minimum interval of 1 minute, it will keep the first snapshot for every minute and remove the other two. It also means, that if your snapshot creation times are somewhat irregular, you don't have to worry about that, because the script will only check if the elapsed time since the last snapshot has been *long enough*. It's great.

