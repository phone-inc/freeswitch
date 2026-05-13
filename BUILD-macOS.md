# macOS Build Notes

These are the steps that successfully compiled this FreeSWITCH checkout on Apple Silicon with Homebrew installed under `/opt/homebrew`.

## Prerequisites

Install the documented Homebrew dependencies:

```sh
brew install autoconf automake curl ffmpeg@5 jpeg ldns libpq@16 libsndfile libtiff libtool lua openssl opus pcre pkgconf sofia-sip speex speexdsp sqlite yasm signalwire/homebrew-signalwire/libks2 signalwire/homebrew-signalwire/signalwire-c2 signalwire/homebrew-signalwire/spandsp
```

If Homebrew reports link conflicts from older SignalWire formulas, unlink the older packages and link the newer documented ones:

```sh
brew unlink libks
brew link libks2
brew unlink signalwire-client-c
brew link signalwire-c2
```

## Configure

Generate autotools files, then configure for the Apple Silicon install prefix:

```sh
./bootstrap.sh
PKG_CONFIG_PATH="/opt/homebrew/opt/openssl@3/lib/pkgconfig:/opt/homebrew/opt/curl/lib/pkgconfig:/opt/homebrew/opt/libpq@16/lib/pkgconfig:/opt/homebrew/opt/ffmpeg@5/lib/pkgconfig:/opt/homebrew/opt/jpeg/lib/pkgconfig:${PKG_CONFIG_PATH}" ./configure --prefix=/opt/freeswitch
```

## Patch

For a clean checkout, apply the local FFmpeg 5+ `mod_av` compatibility patch before building:

```sh
git apply freeswitch-macos-ffmpeg5-mod-av.patch
```

Skip this step if the patch has already been applied. To check whether the current tree already has the patch, run:

```sh
git apply --reverse --check freeswitch-macos-ffmpeg5-mod-av.patch
```

If that command exits successfully, the patch is already present.

## Build

Enable `mod_xml_curl` in `modules.conf` before building:

```text
xml_int/mod_xml_curl
```

Use these include paths so the build picks up `ffmpeg@5`, OpenSSL, TIFF, and JPEG headers instead of conflicting global or keg-only headers:

```sh
make CPPFLAGS="-I/opt/homebrew/opt/ffmpeg@5/include -I/opt/homebrew/opt/openssl@3/include -I/opt/homebrew/opt/libtiff/include -I/opt/homebrew/opt/jpeg/include" CFLAGS="-g -O2 -Wno-error=attribute-warning"
```

The `-Wno-error=attribute-warning` flag keeps newer curl typed-option warnings from failing the build while FreeSWITCH is configured with `-Werror`.

## Verify

After a successful build, check the binary:

```sh
./freeswitch -version
```

Verify that `mod_xml_curl` was built:

```sh
test -f src/mod/xml_int/mod_xml_curl/.libs/mod_xml_curl.so && echo "mod_xml_curl built"
```

Expected result from this checkout:

```text
FreeSWITCH version: 1.11.0-release+git~20260507T215558Z~aae20f9fcd~64bit
```

## Install

Create the install prefix before running `make install`. `/opt` is owned by root on macOS, so creating `/opt/freeswitch` requires `sudo`. Change ownership afterward so the normal, non-root build user can install files into it:

```sh
sudo mkdir -p /opt/freeswitch
sudo chown -R `id -u`:`id -g` /opt/freeswitch
```

Install FreeSWITCH into `/opt/freeswitch`:

```sh
make install
```

Install the sample configuration and htdocs explicitly after `make install`. This safely adds missing files such as `freeswitch.xml` without overwriting existing config files:

```sh
make samples-conf samples-htdocs
```

Fix runtime library lookup for `mod_verto` and `mod_signalwire`, which link to Homebrew's SignalWire libraries via `@rpath`:

```sh
install_name_tool -add_rpath /opt/homebrew/lib /opt/freeswitch/lib/freeswitch/mod/mod_verto.so
install_name_tool -add_rpath /opt/homebrew/lib /opt/freeswitch/lib/freeswitch/mod/mod_signalwire.so
```

Verify the installed runtime starts:

```sh
/opt/freeswitch/bin/freeswitch -c -nonat -nosql
```

On macOS, `ERROR: Could not set nice level` can appear when running without elevated priority permissions. It is non-fatal if FreeSWITCH continues starting.

Optional sound prompts and music on hold:

```sh
make cd-sounds-install cd-moh-install
```

## Local Source Patch

This checkout needed a small `mod_av` compatibility patch so FFmpeg 5+ builds do not link against the removed `avcodec_close()` symbol:

- `freeswitch-macos-ffmpeg5-mod-av.patch`
- `src/mod/applications/mod_av/avcodec.c`
- `src/mod/applications/mod_av/avformat.c`
