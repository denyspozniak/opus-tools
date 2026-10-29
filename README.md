# Opus Audio Tools

[![upstream pipeline](https://gitlab.xiph.org/xiph/opus-tools/badges/master/pipeline.svg)](https://gitlab.xiph.org/xiph/opus-tools/commits/master)
[![upstream](https://img.shields.io/badge/upstream-gitlab.xiph.org-e24329)](https://gitlab.xiph.org/xiph/opus-tools)
[![license](https://img.shields.io/badge/license-BSD--2--Clause%20%2B%20GPL--2.0%20(opusinfo)-blue)](COPYING)
![stand with ukraine](.github/assets/stand-with-ukraine.svg)

**The reference command-line tools for Ogg Opus.** `opus-tools` is the Xiph.Org
Foundation's own encoder, decoder and stream inspector for the Opus codec —
the codec behind WebRTC, most modern SIP trunks and every Ogg `.opus` file.

> [!IMPORTANT]
> **This is a fork. The code is not mine.** opus-tools is written and maintained
> by the Xiph.Org Foundation and its contributors. The canonical repository is
> **[gitlab.xiph.org/xiph/opus-tools](https://gitlab.xiph.org/xiph/opus-tools)**,
> mirrored read-only at [github.com/xiph/opus-tools](https://github.com/xiph/opus-tools).
> Report bugs and send patches there, not here.

Four programs live here:

| Program | What it does |
| --- | --- |
| `opusenc` | encodes WAV, AIFF, FLAC, Ogg/FLAC or raw PCM into Ogg Opus |
| `opusdec` | decodes Ogg Opus to WAV or raw PCM, or plays it straight to the sound card |
| `opusinfo` | inspects an Ogg Opus file and validates its stream structure |
| `opusrtp` | sends an Ogg Opus file as RTP, or pulls Opus RTP out of a live interface or a `.pcap` back into a playable file |

```console
$ opusenc tone.wav tone.opus
$ opusinfo tone.opus | grep -E 'Playback length|bitrate'
	Playback length: 0m:05.000s
	Average bitrate: 118.2 kbit/s, w/o overhead: 115.8 kbit/s
$ opusdec tone.opus back.wav
```

For a VoIP engineer, `opusrtp` is the one worth knowing about — it turns the
Opus leg of a captured call into a file you can listen to:

```console
$ opusrtp --extract call.pcap --port 30000 --output caller.opus
$ opusdec caller.opus caller.wav
```

---

## Table of contents

- [Why this fork exists](#why-this-fork-exists)
- [Install](#install)
  - [From this fork's releases](#from-this-forks-releases)
  - [From your distribution](#from-your-distribution)
- [Build from source](#build-from-source)
- [Usage](#usage)
  - [opusenc](#opusenc)
  - [opusdec](#opusdec)
  - [opusinfo](#opusinfo)
  - [opusrtp](#opusrtp)
- [opusrtp and real captures](#opusrtp-and-real-captures)
- [Tests](#tests)
- [Releases](#releases)
- [Keeping in sync with upstream](#keeping-in-sync-with-upstream)
- [Credits](#credits)
- [Maintenance and AI assistance](#maintenance-and-ai-assistance)
- [License](#license)

---

## Why this fork exists

opus-tools is not abandoned, but its releases are. The last tag, **v0.2, was cut
on 2018-09-18**. Master has moved on by 52 commits since, most of them in a
burst of maintenance in early 2026 — and every Linux distribution still ships
0.2:

| Distribution | Packaged opus-tools | Packaged libopus |
| --- | --- | --- |
| Debian 12 (bookworm) | 0.2 | 1.3.1 |
| Debian 13 (trixie) | 0.2 | 1.5.2 |
| Ubuntu 24.04 (noble) | 0.2 | 1.4 |

So anything merged after 2018 — the QEXT controls, the larger frame sizes, the
C23 and autoconf fixes — only exists if you build from master. Those packages
also leave out `opusrtp` entirely (see [below](#opusrtp)).

This fork builds upstream master into installable packages and publishes them
on every push. **Its code is identical to upstream.** The fork adds only:

- `debian/` — packaging, which also installs `opusrtp`;
- `.github/workflows/release.yml` — the build and release pipeline;
- this README, replacing upstream's one-paragraph one.

Everything under `src/`, `man/` and the build system is untouched.

Lineage:

```
gitlab.xiph.org/xiph/opus-tools  (canonical, Xiph.Org Foundation)
  ├── github.com/xiph/opus-tools          read-only mirror
  └── github.com/denyspozniak/opus-tools  this fork — tracks upstream master
```

## Install

### From this fork's releases

Every push to master that changes code produces a
[release](https://github.com/denyspozniak/opus-tools/releases) with `.deb`
packages built from that commit:

| File | Target |
| --- | --- |
| `opus-tools_<ver>.debian12_amd64.deb` | Debian 12 (bookworm) |
| `opus-tools_<ver>.debian13_amd64.deb` | Debian 13 (trixie) |
| `opus-tools_<ver>.ubuntu2404_amd64.deb` | Ubuntu 24.04 LTS |
| `opus-tools_<ver>.ubuntu2604_amd64.deb` | Ubuntu 26.04 LTS |
| `opus-tools-<describe>.tar.gz` | source tarball from `make distcheck` |

```bash
# pick the file matching your distribution from the latest release
sudo apt install ./opus-tools_0.2.git54.g1234567.ubuntu2404_amd64.deb
opusenc --version
sha256sum -c SHA256SUMS     # optional, with SHA256SUMS from the same release
```

The package replaces the distribution's `opus-tools` and adds `opusrtp`. To go
back: `sudo apt remove opus-tools && sudo apt install opus-tools`.

Ubuntu 22.04 is not built: it has no `libopusenc` package. All builds use the
distribution's own libopus, and none of those is compiled with QEXT, so the
QEXT control is accepted by `opusenc` but rejected by the library.

### From your distribution

Distribution packages give you 0.2 — on Debian and Ubuntu without `opusrtp`.
That is fine for plain encoding and decoding:

```bash
sudo apt install opus-tools      # Debian / Ubuntu
sudo dnf install opus-tools      # Fedora / RHEL-likes
brew install opus-tools          # macOS
```

## Build from source

Dependencies, with the minimum versions `configure.ac` asks for:

| Library | Minimum | Needed for |
| --- | --- | --- |
| libogg | 1.3 | everything |
| libopus | 1.1 | everything |
| libopusfile + libopusurl | 0.5 | `opusdec` |
| libopusenc | 0.2 | `opusenc` |
| libFLAC | 1.1.3 | FLAC input to `opusenc` (optional) |
| libpcap | — | `opusrtp --sniff` / `--extract` (optional) |

```bash
# Debian / Ubuntu
sudo apt install autoconf automake libtool make gcc pkg-config \
  libogg-dev libopus-dev libopusenc-dev libopusfile-dev libflac-dev libpcap-dev

./autogen.sh
./configure
make
make check
sudo make install
```

`./autogen.sh` is only needed for a git checkout; release tarballs come with
`configure` already generated.

Things that are easy to trip over:

- **`make install` does not install `opusrtp`** — neither the binary
  (`noinst_PROGRAMS`) nor its man page (only shipped in the tarball). Both stay
  in the source tree: `./opusrtp` and `man ./man/opusrtp.1`. Copy them yourself
  if you want them on `PATH`.
- **FLAC is on by default.** Turn it off with `./configure --without-flac`. The
  error message you get when FLAC is missing suggests `--disable-flac`, which
  is not the option's name.
- **QEXT depends on the libopus you build against**, not on opus-tools.
  `opusenc --set-ctl-int` only accepts the QEXT control if the libopus headers
  define `OPUS_SET_QEXT_REQUEST`; against an older libopus the build silently
  produces an `opusenc` without it.
- **Windows:** a Visual Studio 2015 solution lives in `win32/VS2015/`.
  `Makefile.unix` is a bare-bones alternative to autotools for quick local
  builds.

Verified on Debian 13 against libopus 1.5.2, libopusenc 0.2.1, libopusfile
0.12, libogg 1.3.5 and libFLAC 1.5.0: `./configure && make && make check`
passes at upstream commit `5f894e9`.

## Usage

Every tool takes `-h` and `-V`, and all but `opusinfo` the long forms
`--help`/`--version`. Full references are in `man 1 opusenc`, `opusdec` and
`opusinfo`, and in `man/opusrtp.1`. The tables below cover
the options you will actually reach for.

### opusenc

```
opusenc [options] input_file output_file.opus
```

Either file can be `-` for stdin/stdout.

| Option | Meaning |
| --- | --- |
| `--bitrate <kbit/s>` | target bitrate. `--help` still says 6–256 per channel; master accepts up to 750 for QEXT |
| `--vbr` / `--cvbr` / `--hard-cbr` | variable (default), constrained variable, or hard constant bitrate |
| `--music` / `--speech` | override the automatic signal detection at low bitrates |
| `--comp <0-10>` | encoder complexity; 10 (slowest, best) is the default |
| `--framesize <ms>` | 2.5, 5, 10, **20** (default), 40, 60 — and on master also 80, 100, 120 |
| `--expect-loss <%>` | tune the encoder for a lossy network |
| `--downmix-mono` / `--downmix-stereo` | reduce channel count before encoding |
| `--max-delay <ms>` | maximum Ogg container delay, 0–1000 |
| `--title`, `--artist`, `--album`, `--date`, `--comment tag=val`, `--picture` | metadata |
| `--raw`, `--raw-rate`, `--raw-chan`, `--raw-bits`, `--raw-float` | headerless PCM input (defaults: 48 kHz, stereo, 16-bit) |
| `--set-ctl-int x=y` | pass a raw encoder control — this is how QEXT is switched on |

```bash
# speech for a voicemail system: mono, 24 kbit/s
opusenc --downmix-mono --speech --bitrate 24 greeting.wav greeting.opus

# straight from a pipe
sox input.mp3 -t wav - | opusenc - output.opus
```

### opusdec

```
opusdec [options] input [output]
```

With no output file it plays the audio (OSS or sndio). An output ending in
`.wav` gets a WAV header; any other name, or `-`, gets raw PCM.

| Option | Meaning |
| --- | --- |
| `--rate <Hz>` | decode at a different sample rate (Opus is always 48 kHz internally) |
| `--force-stereo` | always output two channels |
| `--gain <dB>` | adjust the output volume |
| `--float` | 32-bit float output instead of 16-bit |
| `--no-dither` | skip dithering on 16-bit output |
| `--force-wav` | write a WAV header even to stdout |
| `--packet-loss <%>` | simulate random packet loss — handy for hearing what PLC does |

```bash
# 8 kHz mono for a narrowband pipeline
opusdec --rate 8000 call.opus call-8k.wav
```

### opusinfo

```
opusinfo [-q] [-v] file1.opus [file2.opus ...]
```

Prints the header, tags, channel mapping, playback length and bitrate of each
stream, and flags structural errors in the Ogg container — holes in the data,
bad granule positions, broken headers. `-v` adds detail, `-q` (once or twice)
strips it down to errors. Long options are not supported here; `--help` is
rejected.

### opusrtp

```
opusrtp [transmit-options] in.opus ...
opusrtp --sniff <device> | --extract <in.pcap> [receive-options]
```

| Option | Meaning |
| --- | --- |
| `-d`, `--destination <addr>` | transmit: destination address (default `127.0.0.1`) |
| `-p`, `--port <n>` | transmit: destination port (default 1234). Receive: only take packets **to** this port |
| `-t`, `--type <n>` | RTP payload type to send (default 120) or to accept |
| `--sniff <device>` | receive from a live interface |
| `-e`, `--extract <file>` | receive from a `.pcap` |
| `-o`, `--output <file>` | Ogg Opus file to write |
| `-r`, `--rate <Hz>` / `-c`, `--channels <n>` | values written into the output header (defaults 48000, 2) |

Transmit paces packets in real time, so it doubles as a simple Opus RTP source
for testing a media server:

```bash
opusrtp --destination 192.0.2.10 --port 30000 --type 111 prompt.opus
```

## opusrtp and real captures

`opusrtp --extract` was written to round-trip its own output, and it shows on
real call captures. Each point below was checked against upstream commit
`5f894e9` with synthetic and live pcaps, not just read from the code:

| Situation | What happens | Workaround |
| --- | --- | --- |
| Capture taken with `tcpdump -i any` | **Nothing is extracted.** Current libpcap writes `-i any` captures as `LINUX_SLL2` (link type 276); opusrtp only knows Ethernet, BSD loopback and the older `LINUX_SLL`, and skips every packet | capture with `tcpdump -i any -y LINUX_SLL`, or on a named interface (`-i eth0`) |
| RTP sequence number wraps past 65535 | Every packet after the wrap is dropped as "out-of-sequence". The comparison is a plain `<` with no 16-bit wrap handling; in the test, a 10 s stream that wrapped 2 s in came out **2 s long** | none in opus-tools; RTP starts at a random sequence number, so on average a call wraps after about 11 minutes at 20 ms packets |
| No `--port` given, two-way call | It locks onto the first packet that parses as Opus with a dynamic payload type and keeps only that direction | always pass `--port` with the RTP port of the leg you want |
| Packet loss in the capture | Lost packets are not padded: the granule position only counts what arrived, so the timeline closes up instead of leaving a gap | fine for listening, wrong for measuring timing |
| 802.1Q / QinQ VLAN tags | handled | — |
| IPv6 | handled | — |

The extraction writes one direction of one stream. For a whole call, run it
once per leg.

## Tests

```bash
make check       # what upstream CI runs
make distcheck   # also checks the release tarball builds
```

Upstream CI ([`.gitlab-ci.yml`](.gitlab-ci.yml)) builds on Debian 12 with
autotools and runs both targets. On GitHub, the release pipeline below runs
`make check` inside every package build and `make distcheck` for the tarball.

## Releases

There is no manual release step. [`release.yml`](.github/workflows/release.yml)
runs on every push to master — documentation-only pushes excepted — and can be
started by hand from the Actions tab:

1. **Version** from `git describe`, the same lookup upstream's
   `update_version` does: `0.2-54-g1234567` becomes `0.2+git54.g1234567`. The
   commit count keeps successive builds in order for `apt`, and `+git` sorts
   above the distributions' plain `0.2`. Each `.deb` gets a `~<dist>` suffix.
2. **Packages** for Debian 12/13 and Ubuntu 24.04/26.04, each built in its
   own container with `dpkg-buildpackage`, then installed and smoke-tested:
   all four tools present, `opusenc --version` matching the commit, and a round
   trip through `opusenc` → `opusinfo` → `opusdec`.
3. **Source tarball** from `make distcheck`.
4. **Release** tagged with the version (no `v` prefix, so it can never be
   picked up by `update_version`), with install and usage notes, the upstream
   commit it is based on, and `SHA256SUMS`. Re-running the same commit
   refreshes that release instead of failing.

If the repository was created with GitHub's *Fork* button, Actions are off by
default: enable them once under the repository's **Actions** tab.

## Keeping in sync with upstream

```bash
git remote add upstream https://gitlab.xiph.org/xiph/opus-tools.git
git fetch upstream
git merge upstream/master
git push origin master
```

A plain merge, not `--ff-only`: the fork carries its own commits (packaging,
workflow, README), so it cannot fast-forward. Upstream has no `debian/` or
`.github/workflows/`, so the only conflict to expect is in `README.md` — keep
this fork's version. The push then builds and releases the new upstream code.

## Credits

opus-tools is entirely the work of the Xiph.Org Foundation and its
contributors:

- **[Xiph.Org Foundation](https://www.xiph.org/)**, which develops Opus,
  opus-tools, libopusenc, opusfile and libogg.
- **Gregory Maxwell**, **Mark Harris**, **Ralph Giles**, **Jean-Marc Valin**
  and **Timothy B. Terriberry** — the largest contributors by commit count —
  among many others; see [`AUTHORS`](AUTHORS) and the full history in
  `git log`. Mark Harris authored nearly all of the 2026 maintenance on master.
- **Jean-Marc Valin** and **Thorvald Natvig**, for the original `opusdec` and
  the resampler; **John Edwards**, for Windows audio output.
- **Michael Smith**, whose `ogginfo` from vorbis-tools is the origin of
  `opusinfo`.

Canonical source: [gitlab.xiph.org/xiph/opus-tools](https://gitlab.xiph.org/xiph/opus-tools).
Project home: [opus-codec.org](https://opus-codec.org/).

## Maintenance and AI assistance

This README was written with an AI coding assistant (Claude Code) and reviewed
by a human. The claims in it were checked rather than assumed: the build and
`make check` were run on Debian 13, `--help` output was captured from the
built binaries, distribution versions were read from the package archives, and
every row in the [opusrtp table](#opusrtp-and-real-captures) was reproduced
with a pcap.

The Debian packaging and the release workflow were written the same way, and
the package build was run in containers for all four target distributions,
with the install test, before it was committed.

No upstream code in this fork has been changed. If that ever changes, the
change will be listed here and offered upstream first.

## License

Unchanged from upstream, see [COPYING](COPYING):

- everything except `opusinfo` — two-clause BSD-style license;
- `opusinfo.c`/`opusinfo.h` — GPL-2.0, inherited from `ogginfo` in vorbis-tools.

A binary distribution of `opusinfo` therefore carries the GPL's obligations;
the other tools do not.

---

*Generated with Claude AI — please review before distribution.*
