# tarsnap-stats

[![License: CC0](https://img.shields.io/badge/license-CC0_1.0-lightgrey.svg?style=plastic)](https://creativecommons.org/publicdomain/zero/1.0/)
![Built With Love](https://img.shields.io/badge/built%20with-%E2%9D%A4%20by%20DeuZa-red?style=plastic)
![Hack The Planet](https://img.shields.io/badge/hack-the--planet-black?style=flat-square&logo=gnu&logoColor=white)

A small POSIX `sh` wrapper that turns `tarsnap --print-stats` into a readable
summary: snapshot count, sizes, deduplication ratios and storage cost.

No dependency beyond a POSIX shell, the usual base utilities (`awk`, `grep`,
`sed`, `tail`, `date`, `mktemp`...) and `tarsnap` itself; `stdbuf` is optional.
It needs tarsnap 1.0.37 or later, for `--verify-config`, and 1.0.40 or later
for `-p`; it is tested with tarsnap 1.0.41.

## Why

`tarsnap --print-stats` gives you raw byte counts, unless `tarsnap.conf` sets
`humanize-numbers`. `--humanize-numbers` gives you sizes truncated to two or
three significant digits. Neither gives you the deduplication ratios, the
snapshot count, or what the whole thing actually costs you per month.

This script adds the missing arithmetic, and nothing else. Every figure it
prints can be recomputed by hand from tarsnap's own output (see
[Verification](#verification)).

## Output

```text
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

```text
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

Either way, the script works from raw byte counts: it passes
`--no-humanize-numbers` to every tarsnap call, so a `humanize-numbers` line in
`tarsnap.conf` changes nothing, and it refuses a table whose sizes are not
plain byte counts (exit code 6).

## Verification

Do not take the numbers on trust. Here is the same account, straight from
tarsnap, at the same moment.

Raw byte counts, with the `--no-humanize-numbers` that the script passes too,
so that a `humanize-numbers` line in `tarsnap.conf` cannot get in the way:

```text
root@foobar:~# tarsnap --print-stats --no-humanize-numbers
                                       Total size  Compressed size
All archives                         647226578282     619831444714
  (unique data)                        9209227991       8566901824
```

And tarsnap's own human-readable rendering:

```text
root@foobar:~# tarsnap --print-stats --humanize-numbers
                                       Total size  Compressed size
All archives                               647 GB           619 GB
  (unique data)                            9.2 GB           8.5 GB
```

And the snapshot count, which the script takes as the number of non-empty
lines of the archive list:

```text
root@foobar:~# tarsnap --list-archives | grep -c .
46
```

Every figure follows from that count and those four numbers:

| Figure | Computation | Result |
| --- | --- | --- |
| Snapshots | non-empty lines of `tarsnap --list-archives` | 46 |
| Total size | 647226578282 / 10^9 | 647.23 GB |
| Compressed size | 619831444714 / 10^9 | 619.83 GB |
| Unique total | 9209227991 / 10^9 | 9.21 GB |
| Encoded size | 8566901824 / 10^9 | 8.57 GB |
| All archives ratio | 619831444714 / 647226578282 | 95.8% |
| Unique data ratio | 8566901824 / 9209227991 | 93.0% |
| Monthly cost | 8566901824 x 250e-12 | $2.1417 |
| Daily cost | monthly / number of days in the current month (30 in September) | $0.071391 |

In `-b` mode the same bytes are divided by 2^30 instead, which is why 647.23 GB
and 602.78 GiB are the same amount of data.

Note that tarsnap truncates rather than rounds: 619.83 is displayed as 619, and
8.566 as 8.5. This script rounds to two decimals, so the last digit can differ
from `--humanize-numbers` by design.

## Install

Download the script directly using `curl`:

```sh
curl -fsSLO https://raw.githubusercontent.com/deuza/tarsnap-stats/main/tarsnap-stats.sh
chmod +x tarsnap-stats.sh
```

or `wget`:

```sh
wget https://raw.githubusercontent.com/deuza/tarsnap-stats/main/tarsnap-stats.sh
chmod +x tarsnap-stats.sh
```

or clone the entire repository using Git, which keeps the script executable:

```sh
git clone https://github.com/deuza/tarsnap-stats.git
cd tarsnap-stats
```

There is nothing to configure. The script reads your existing `tarsnap.conf`,
so if `tarsnap --list-archives` and `tarsnap --print-stats` work, this works.

## Usage

```text
usage: ./tarsnap-stats.sh [-b] [-k keyfile] [-p method]
  -b           use binary prefixes (KiB/MiB/GiB) instead of SI ones
  -k keyfile   key file of this machine, when it is not where tarsnap.conf
               says it is (default there is usually /root/tarsnap.key)
  -p method    how tarsnap reads the passphrase of a protected key file:
               env:VAR, file:PATH, dev:tty-stdin or dev:tty-once
               (tarsnap 1.0.40 or later)
```

`-h` prints that, and so does `--help` given as the first argument. Four
environment variables are honoured:

| Variable | Effect |
| --- | --- |
| `TARSNAP_KEYFILE` | same as `-k`, which wins over it |
| `TARSNAP_SPINNER` | set to 0 to disable the progress display |
| `TARSNAP` | path to the tarsnap binary (a path only, not a command with arguments) |
| `TMPDIR` | directory of the temporary file, `/tmp` by default |

None of them stands for `-p`, on purpose: a variable named after the
passphrase would invite exporting the passphrase itself, which would then end
up on tarsnap's command line.

`-k` is meant for a key of **this** machine sitting somewhere unusual. It is
not a way to report on another machine's backups: `--print-stats` never queries
the server, it reads the local cache directory, so a foreign key would list
that machine's archives while printing this machine's sizes.

### Passphrase-protected key

Tarsnap loads the key file on each of the two calls that follow the
configuration check, `--list-archives` and `--print-stats`, so a protected key
asks for its passphrase twice by default. `-p METHOD` is passed to every
tarsnap call as `--passphrase METHOD`, which needs tarsnap 1.0.40 or later:

- `env:VAR` reads the passphrase from the environment variable `VAR`, and
  `file:PATH` from the file `PATH`: nothing is asked at all.
- `dev:tty-stdin`, tarsnap's default, and `dev:tty-once` keep the prompt, once
  per call.
- `dev:stdin-once` is refused: the second call would find standard input
  already read.

Any other value is refused with exit code 2, before tarsnap runs, and the
message does not repeat it, in case a passphrase was typed instead of a method.

A `passphrase` line in `tarsnap.conf` does the same job, and `-p` wins over it:
tarsnap processes its command line before its configuration files. As
tarsnap(1) warns, keeping a passphrase in an environment variable or in a file
may be a security risk.

## Progress display

While the archive list is being fetched, the count and the name of the last
archive received are shown on stderr:

```text
\ 41 archives, last: foobar-2026-09-04_03-00-00
```

It only appears when stderr is a terminal, so nothing pollutes your logs, and
stdout stays clean and pipeable in every case. With the live count, nothing
at all is drawn before the first archive arrives, so the progress line can
never land on top of a passphrase prompt. Any byte of an archive name outside
printable ASCII is shown as `?`, so that a name cannot drive your terminal, and
the line stops by itself if the script is killed, even with `kill -9`.

The live count needs `stdbuf(1)`, shipped with GNU coreutils and with the
FreeBSD base system, otherwise tarsnap fills a 4 kB stdio buffer before
anything reaches the file being watched. Without it, the script falls back to
a plain spinner, which starts drawing at once and may therefore overlap a
passphrase prompt; set `TARSNAP_SPINNER=0` to disable it.

## Exit codes

Every failure prints its exact reason on stderr, except a stop by a signal
(128 + the signal number), which cleans up and exits silently.

| Code | Meaning |
| --- | --- |
| 0 | success |
| 2 | command line usage error |
| 3 | configuration rejected by `tarsnap --verify-config` |
| 4 | `tarsnap --list-archives` failed |
| 5 | `tarsnap --print-stats` failed |
| 6 | unrecognised `--print-stats` output, or sizes that are not plain byte counts |
| 7 | could not create a temporary file |
| 8 | `-k` or `TARSNAP_KEYFILE` points at an unreadable key file |
| 9 | the tarsnap binary was not found or is not executable |
| 129 | stopped by HUP (terminal closed), after cleaning up |
| 130 | stopped by INT (`Ctrl-C`), after cleaning up |
| 131 | stopped by QUIT (`Ctrl-\`), after cleaning up |
| 143 | stopped by TERM, after cleaning up |

The checks run in a fixed order: the command line and the `-p` value (code 2),
the `-k` key file (8), the temporary file (7), then the tarsnap binary and the
configuration, with `tarsnap --verify-config` (9 and 3), which reads no key and
makes no network call. Only then come the two calls that load the key,
`--list-archives` (4) and `--print-stats` (5), and the reading of the table
(6). A broken `tarsnap.conf` therefore fails at once, and a passphrase is never
asked for a run that was going to stop anyway.

## Do not put this in a crontab

Run it by hand, when you actually want to look at the numbers.

- `--list-archives` downloads and decrypts the whole archive metadata on every
  run. Tarsnap bills bandwidth per byte transmitted, and these figures only
  move when an archive is created or deleted.
- `--print-stats` reads the cache directory, which a scheduled backup may be
  writing to at the same time.
- Without an `env:` or `file:` passphrase method, given with `-p` or in
  `tarsnap.conf` (see [Passphrase-protected key](#passphrase-protected-key)),
  a protected key cannot be unlocked without a terminal, so the script simply
  fails under cron.
- With `-k` pointing at another machine's key, the snapshot count would come
  from that machine and the sizes from this one, and nobody reads the figures
  of an unattended run to notice the mix.

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
- With a protected key and no passphrase method, you are prompted twice, once
  per key-using call: an `env:` or `file:` method avoids it, see
  [Passphrase-protected key](#passphrase-protected-key).
- `shellcheck -s sh` clean, no bashisms, tested under `dash`, `bash` and
  `busybox sh` by the regression suite of tarsnap-tools (see
  [Development](#development)).

## References

- [Tarsnap pricing](https://www.tarsnap.com/) - 250 picodollars per byte-month
  of encoded data
- [tarsnap(1)](https://www.tarsnap.com/man-tarsnap.1.html) - `--humanize-numbers`
  uses SI prefixes and `--no-humanize-numbers` overrides it in a configuration
  file, global statistics come from the cache directory, and `--passphrase`
  takes the methods listed above
- [Tarsnap FAQ](https://www.tarsnap.com/faq.html#daily-cost-change) - why the
  daily storage cost changes from month to month
- [Tarsnap NEWS](https://github.com/Tarsnap/tarsnap/blob/1.0.41/NEWS.md) -
  `--verify-config` appeared in 1.0.37, `--passphrase` in 1.0.40

## Development

`tarsnap-stats.sh` is developed in
[deuza/tarsnap-tools](https://github.com/deuza/tarsnap-tools), alongside the
backup scripts it completes: it is audited and regression-tested there, and
every fix is made there before the file is copied here unchanged.

## License

`CC0 1.0 Universal` -- Public Domain.  
[![CC0](https://mirrors.creativecommons.org/presskit/buttons/88x31/svg/cc-zero.svg)](https://creativecommons.org/publicdomain/zero/1.0/)

<p align="center">With ❤️ by <a href="https://github.com/deuza">DeuZa</a></p>
