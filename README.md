[![License: CC0](https://img.shields.io/badge/license-CC0_1.0-lightgrey.svg?style=plastic)](https://creativecommons.org/publicdomain/zero/1.0/)
![Built With Love](https://img.shields.io/badge/built%20with-%E2%9D%A4%20by%20DeuZa-red?style=plastic)
![Hack The Planet](https://img.shields.io/badge/hack-the--planet-black?style=flat-square&logo=gnu&logoColor=white)

# tarsnap-stats

A small POSIX `sh` wrapper that turns `tarsnap --print-stats` into a readable
summary: snapshot count, sizes, deduplication ratios and storage cost.

No dependency beyond a shell, `awk` and `tarsnap` itself.

## Why

`tarsnap --print-stats` gives you raw byte counts. `--humanize-numbers` gives
you sizes truncated to three significant digits. Neither gives you the
deduplication ratios, the snapshot count, or what the whole thing actually
costs you per month.

This script adds the missing arithmetic, and nothing else. Every figure it
prints can be recomputed by hand from tarsnap's own output (see
[Verification](#verification)).

## Output

```
root@foobar:~# ./tarsnap-stats.sh
All archives:
  Snapshots:        46
  Total size:       647.23 GB
  Compressed size:  619.83 GB
  Ratio:            95.8%

Unique data:
  Total size:       9.21 GB
  Encoded size:     8.57 GB
  Ratio:            93.0%

Storage cost:
  Monthly:          $2.1417
  Daily:            $0.071391 (30-day month)
  (250 pUSD/byte/month of encoded data)
```

46 snapshots totalling 647 GB, deduplicated and compressed down to the 8.57 GB
that are actually stored on the Tarsnap servers and billed. That is the whole
point of the tool: the number you pay for is not the number you back up.

Sizes use SI prefixes by default, the same as `tarsnap --humanize-numbers`.
Pass `-b` for binary prefixes, correctly labelled:

```
root@foobar:~# ./tarsnap-stats.sh -b
All archives:
  Snapshots:        46
  Total size:       602.78 GiB
  Compressed size:  577.26 GiB
  Ratio:            95.8%

Unique data:
  Total size:       8.58 GiB
  Encoded size:     7.98 GiB
  Ratio:            93.0%

Storage cost:
  Monthly:          $2.1417
  Daily:            $0.071391 (30-day month)
  (250 pUSD/byte/month of encoded data)
```

## Verification

Do not take the numbers on trust. Here is the same account, straight from
tarsnap, at the same moment.

Raw byte counts:

```
root@foobar:~# tarsnap --print-stats
                                       Total size  Compressed size
All archives                         647226578282     619831444714
  (unique data)                        9209227991       8566901824
```

And tarsnap's own human-readable rendering:

```
root@foobar:~# tarsnap --print-stats --humanize-numbers
                                       Total size  Compressed size
All archives                               647 GB           619 GB
  (unique data)                            9.2 GB           8.5 GB
```

Every figure follows from those four numbers:

| Figure | Computation | Result |
| --- | --- | --- |
| Total size | 647226578282 / 10^9 | 647.23 GB |
| Compressed size | 619831444714 / 10^9 | 619.83 GB |
| Unique total | 9209227991 / 10^9 | 9.21 GB |
| Encoded size | 8566901824 / 10^9 | 8.57 GB |
| All archives ratio | 619831444714 / 647226578282 | 95.8% |
| Unique data ratio | 8566901824 / 9209227991 | 93.0% |
| Monthly cost | 8566901824 x 250e-12 | $2.1417 |
| Daily cost | monthly / 30 days in September | $0.071391 |

In `-b` mode the same bytes are divided by 2^30 instead, which is why 647.23 GB
and 602.78 GiB are the same amount of data.

Note that tarsnap truncates rather than rounds: 619.83 is displayed as 619, and
8.566 as 8.5. This script rounds to two decimals, so the last digit can differ
from `--humanize-numbers` by design.

## Install

Download the script using `curl` or `wget` :


 

```
git clone https://github.com/deuza/tarsnap-stats.git
cd tarsnap-stats
chmod +x tarsnap-stats.sh
./tarsnap-stats.sh
```

There is nothing to configure. The script reads your existing `tarsnap.conf`,
so if `tarsnap --print-stats` works, this works.

## Usage

```
usage: ./tarsnap-stats.sh [-b] [-k keyfile]
  -b           use binary prefixes (KiB/MiB/GiB) instead of SI ones
  -k keyfile   key file of this machine, when it is not where tarsnap.conf
               says it is (default there is usually /root/tarsnap.key)
```

`-h` and `--help` print that. Two environment variables are honoured:

| Variable | Effect |
| --- | --- |
| `TARSNAP_KEYFILE` | same as `-k` |
| `TARSNAP_SPINNER=0` | disable the progress display |
| `TARSNAP` | path to the tarsnap binary (a path only, not a command with arguments) |

`-k` is meant for a key of **this** machine sitting somewhere unusual. It is
not a way to report on another machine's backups: `--print-stats` never queries
the server, it reads the local cache directory, so a foreign key would list
that machine's archives while printing this machine's sizes.

## Progress display

While the archive list is being fetched, the count and the name of the last
archive received are shown on stderr:

```
\ 41 archives, last: backup-2026-09-04-0300
```

It only appears when stderr is a terminal, so nothing pollutes your logs, and
stdout stays clean and pipeable in every case. Nothing at all is drawn before
the first archive arrives, so the progress line can never land on top of a
passphrase prompt.

The live count needs `stdbuf(1)` from GNU coreutils, otherwise tarsnap fills a
4 kB stdio buffer before anything reaches the file being watched. Without it
(BSD, macOS) the script falls back to a plain spinner.

## Exit codes

Every failure prints its exact reason on stderr.

| Code | Meaning |
| --- | --- |
| 0 | success |
| 2 | command line usage error |
| 3 | configuration rejected by `tarsnap --verify-config` |
| 4 | `tarsnap --list-archives` failed |
| 5 | `tarsnap --print-stats` failed |
| 6 | unrecognised `--print-stats` output |
| 7 | could not create a temporary file |
| 8 | `-k` points at an unreadable key file |

The configuration is checked first, before any key is read and before any
network call, so a broken `tarsnap.conf` fails immediately instead of producing
three confusing errors in a row.

## Do not put this in a crontab

Run it by hand, when you actually want to look at the numbers.

- `--list-archives` downloads and decrypts the whole archive metadata on every
  run. Tarsnap bills bandwidth per byte transmitted, and these figures only
  move when an archive is created or deleted.
- `--print-stats` reads the cache directory, which a scheduled backup may be
  writing to at the same time.
- A passphrase-protected key cannot be unlocked without a terminal, so the
  script simply fails under cron.

If you want the numbers automatically, add `--print-stats` to your own backup
invocation instead. It costs nothing extra there.

## Notes

- Only one network call is made, `--list-archives`, and only to count the
  snapshots. `--print-stats` is computed locally from the cache directory.
  `--no-aggressive-networking` is passed to every invocation so the script
  never opens multiple connections to the Tarsnap servers.
- The daily cost divides by the real length of the current month, not by a flat
  30, because tarsnap bills per byte-day. In February the difference is about
  7%.
- With a passphrase-protected key you will be prompted twice, once per
  key-using call. That is how tarsnap works, not something the script can
  avoid.
- `shellcheck -s sh` clean, no bashisms, tested under `dash`.

## References

- [Tarsnap pricing](https://www.tarsnap.com/) - 250 picodollars per byte-month
  of encoded data
- [tarsnap(1)](https://www.tarsnap.com/man-tarsnap.1.html) - `--humanize-numbers`
  uses SI prefixes, and global statistics come from the cache directory
- [Tarsnap FAQ](https://www.tarsnap.com/faq.html) - why the daily storage cost
  changes from month to month

## License
`CC0 1.0 Universal` -- Public Domain.  
[![CC0](https://mirrors.creativecommons.org/presskit/buttons/88x31/svg/cc-zero.svg)](https://creativecommons.org/publicdomain/zero/1.0/)    

## Author
<a href="https://github.com/deuza">DeuZa</a>

<p align="center">With ❤️ by <a href="https://github.com/deuza">DeuZa</a></p></sup></sub>
