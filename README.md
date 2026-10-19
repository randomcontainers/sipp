# SIPp

Container images with [SIPp](https://github.com/SIPp/sipp), a SIP traffic generator for testing VoIP servers, proxies and phones. SIPp places and answers SIP calls as described in XML scenario files, at a set call rate, and reports response times and failed calls. SIPp is compiled from the release tarball on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when SIPp publishes a release and when the base image changes.

This is an unofficial build, not affiliated with or endorsed by the SIPp project. Report problems with the image in this repository and SIPp bugs at [SIPp/sipp](https://github.com/SIPp/sipp/issues).

## Quick start

Answer calls on port 5060 with the built-in UAS scenario:

```sh
docker run --rm --network host ghcr.io/randomcontainers/sipp -sn uas -p 5060
```

On another host, place 100 calls against it at 10 calls per second, with the server's name or IP address in place of `<server>`:

```sh
docker run --rm --network host ghcr.io/randomcontainers/sipp -sn uac -r 10 -m 100 <server>:5060
```

The entrypoint runs `sipp` under `tini` in `/work`, and without arguments it prints the help. `-sn` runs one of the scenarios built into SIPp: `uac` and `uas`, `regexp`, `branchc` and `branchs`, the four `3pcc` scenarios, and `uac_pcap`. `-sd <name>` prints one as XML, a starting point for your own:

```sh
docker run --rm ghcr.io/randomcontainers/sipp -sd uac > uac.xml
```

Without a terminal, SIPp prints its statistics when it exits. With `-it` it shows its live screens instead, where `+` and `-` change the call rate and `q` stops the test:

```sh
docker run --rm -it --network host ghcr.io/randomcontainers/sipp -sn uac -r 10 <server>:5060
```

The [SIPp documentation](https://sipp3.readthedocs.io/en/latest/) describes the options and the scenario format. SIPp 3.8 also asks for a scenario step by step when it starts in a terminal without arguments; the image passes `-h` by default, so start it with `docker run --rm -it --network host --entrypoint sipp ghcr.io/randomcontainers/sipp`.

## Scenario files

Mount the directory with your scenario and injection files. SIPp writes its log files there too, such as the SIP messages with `-trace_msg` and the statistics with `-trace_stat`, named after the scenario and the process ID unless you set the names with `-message_file` and `-stf`:

```sh
docker run --rm --network host --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/sipp -sf scenario.xml -inf users.csv -m 50 -trace_msg <server>:5060
```

`-inf` reads values from a CSV file into `[field0]`, `[field1]` and so on, for example a user name and a password for `[authentication]`. `-lua_file` loads Lua functions that a scenario calls with `<exec lua="function arg ..."/>`, and these read and set the call's variables with `sipp.get` and `sipp.set`.

SIPp exits with 0 when every call succeeded and 1 when at least one failed, so a test run can fail a CI job. A run that ends at its `-timeout` exits with 0 if at least one call succeeded and none failed, and with 97 if no call succeeded or failed, for example a UAS that received no calls; add `-timeout_error` to make the timeout a fatal error. After `-h`, `-v` or `-sd` SIPp exits with 99. 253 means that an RTP check failed, 254 that a port SIPp needed was already in use, and 255 is any other fatal error, such as an unknown option.

## Networking

With `--network host` on Linux, SIPp uses the host's addresses and ports. SIPp writes its own address into the Via and Contact headers and the SDP of its messages, and on Docker's default bridge network that is the container's address, which a peer outside the host cannot reach. Two containers on one user-defined network can reach each other by name:

```sh
docker network create sip
docker run -d --rm --name uas --network sip ghcr.io/randomcontainers/sipp -sn uas -p 5060
docker run --rm --network sip ghcr.io/randomcontainers/sipp -sn uac -r 5 -m 20 uas:5060
```

SIPp 3.8 looks up a host name given without a port in DNS NAPTR and SRV records (RFC 3263). Without such records it uses the name's address and port 5060; give the port to skip these lookups or to use a port other than the one in the SRV record. The image runs as UID 1000, which can bind any port on Docker's bridge and user-defined networks. With `--network host`, and with Podman, a port below 1024 needs `--user 0:0`. `-i <address>` picks the local address when the host has several.

The transport is UDP by default. `-t t1` uses TCP, `-t l1` TLS, `-t w1` and `-t x1` SIP over WebSocket and secure WebSocket, and `-t s1` SCTP, which needs the `sctp` module in the host kernel. The `n` forms (`-t un`, `-t tn` and so on) use one socket per call. For TLS, a server needs `-tls_cert` and `-tls_key` (by default `cacert.pem` and `cakey.pem` in `/work`), and `-tls_ca` makes SIPp verify the peer's certificate.

To watch the SIP messages on the wire, run the [tcpdump](https://github.com/randomcontainers/tcpdump) or [TShark](https://github.com/randomcontainers/tshark) image in the same network namespace, here the `uas` container from above:

```sh
docker run --rm --user 0 --cap-add NET_RAW --network container:uas \
  ghcr.io/randomcontainers/tcpdump -i eth0 -n -A port 5060
```

## Playing RTP from capture files

The `play_pcap_audio`, `play_pcap_video` and other `play_pcap_*` actions send the packets of a capture file through a raw socket, which needs the `NET_RAW` capability. A container process only has that capability when it runs as root, so these scenarios need `--user 0`. Docker gives root in a container `NET_RAW` by default and Podman does not; `--cap-add NET_RAW` works with both. The binary has no file capabilities: with `cap_net_raw+ep` set it would not start at all under `--cap-drop ALL`, and `no-new-privileges` ignores file capabilities.

The capture files from SIPp's source, a G.711 A-law recording and the RFC 2833 DTMF digits, are in `/usr/local/share/sipp/pcap`. The built-in `uac_pcap` scenario plays `pcap/g711a.pcap` and `pcap/dtmf_2833_1.pcap` relative to the working directory, so run it in `/usr/local/share/sipp`:

```sh
docker run --rm --user 0 --cap-add NET_RAW --network host -w /usr/local/share/sipp \
  ghcr.io/randomcontainers/sipp -sn uac_pcap -m 1 <server>:5060
```

Log files that SIPp writes to a mounted directory while it runs as root are owned by root on the host.

## What is in the image

- `sipp` in `/usr/local/bin`, with TLS (OpenSSL), SIP over WebSocket and secure WebSocket, SCTP (lksctp-tools), capture file playback (libpcap) and Lua 5.4 scripting. `sipp -v` lists all of them except WebSocket after the version: `TLS-SCTP-PCAP-LUA`.
- The GNU Scientific Library, for the `normal`, `lognormal`, `exponential`, `weibull`, `pareto`, `gpareto`, `gamma` and `negbin` distributions of `<pause distribution="...">`.
- The capture files described in [Playing RTP from capture files](#playing-rtp-from-capture-files).

OpenSSL, libpcap, lksctp-tools, Lua, GSL, pugixml and ncurses are the Ubuntu or Alpine packages. Not included: `sipp-multi.py` and the other Python tools in SIPp's source, the man page, and TLS key logging to `SSLKEYLOGFILE`, which SIPp leaves out by default. The CMake options are in `/usr/local/share/randomcontainers/sipp/buildinfo`.

## Default or slim

SIPp's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is a SIPp release such as `3.8.0`. `<minor>` and `<major>` are its shorter forms, `3.8` and `3`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<minor>`, `<minor>-slim`, `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<minor>-ubuntu`, `<minor>-slim-ubuntu`, `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<minor>-alpine`, `<minor>-slim-alpine`, `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current SIPp version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new SIPp release and are rebuilt when the base image changes. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/sipp:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends iproute2 \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

`iproute2` adds `ip` and `ss`, for looking at addresses and open sockets next to a test. On Alpine, start from `slim-alpine` and use `apk add --no-cache iproute2`. The entrypoint is `["tini", "--", "sipp"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new SIPp releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/sipp/` holds the version, the source URLs, the build options, the license texts and `runtime-deps`, the list of distro packages SIPp needs at run time.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/sipp:latest \
  --repo randomcontainers/sipp --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/sipp:latest --format '{{ json .SBOM }}'
```

SIPp is compiled here, so it is not in the SBOM and image scanners do not check it. Its version is in `/usr/local/share/randomcontainers/sipp/version`.

SIPp publishes no signatures or checksum files. The SHA-256 in `package.yml` is the digest GitHub records for the `sipp-<version>.tar.gz` release asset, and the build checks the archive against it before compiling.

## Updates

The project checks the releases of [SIPp/sipp](https://github.com/SIPp/sipp/releases) every 15 minutes. A release is picked up once it is 72 hours old, since SIPp's fix releases have come within days of a release. Its source tarball is checked against the digest GitHub records for the release asset, the new version and the tarball's SHA-256 are committed to `package.yml`, and the images are rebuilt. Only the newest release by version number is built, so a later fix release of an older series, such as 3.7, is skipped; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes, including those for OpenSSL and libpcap, reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t sipp:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

SIPp's `LICENSE.txt` and most of its source files put it under the GNU General Public License, version 2 or later (GPL-2.0-or-later), while its README and `include/auth.hpp`, which SIPp compiles in, say version 3 or later. `send_packets.c` and `send_packets.h`, from tcpreplay, are under the BSD-3-Clause license. SIPp also links the GNU Scientific Library from the distro, licensed under the GNU General Public License, version 3 or later. For both reasons the `sipp` binary in the image is distributed under GPL-3.0-or-later. The image's license label is `GPL-2.0-or-later AND BSD-3-Clause AND GPL-3.0-or-later`. `/usr/local/share/randomcontainers/sipp/licenses/` holds SIPp's `LICENSE.txt`, with the GPL version 2 and the BSD license, and `GPL-3`, the text of the GPL version 3. The other libraries from Ubuntu or Alpine keep their own licenses, for example Apache-2.0 for OpenSSL and MIT for Lua and pugixml. The SBOM lists them.

The corresponding source for each image:

- SIPp: every version has a GitHub release in this repository, named `v<version>`, with the exact `sipp-<version>.tar.gz` that was compiled. The build applies no patches. `/usr/local/share/randomcontainers/sipp/source` lists that release and the download URL of the archive.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label. The Dockerfiles hold every CMake option.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

The files in this repository are available under the MIT license, see [LICENSE](LICENSE), except the license texts in `licenses/`, which keep their own terms.

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
