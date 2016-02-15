# Debuggin SSL on Linux

To help anyone looking at the SSL code, here are a few tips I've found handy.

[TOC]

## Building your own NSS

In order to use a debugger with the NSS library, it helps to build NSS yourself.
Here's how I did it:

First, read
[Network Security Services](http://www.mozilla.org/projects/security/pki/nss/nss-3.11.4/nss-3.11.4-build.html)
and/or
[Build instructions](https://developer.mozilla.org/En/NSS_reference/Building_and_installing_NSS/Build_instructions).

Then, to build the most recent source tarball:

```shell
cd $HOME
wget ftp://ftp.mozilla.org/pub/mozilla.org/security/nss/releases/NSS_3_12_RTM/src/nss-3.12-with-nspr-4.7.tar.gz
tar -xzvf nss-3.12-with-nspr-4.7.tar.gz
cd nss-3.12/
cd mozilla/security/nss/
make nss_build_all
```

Sadly, the latest release, 3.12.2, isn't available as a tarball, so you have to
build it from cvs:

```shell
cd $HOME
mkdir nss-3.12.2
cd nss-3.12.2
export CVSROOT=:pserver:anonymous@cvs-mirror.mozilla.org:/cvsroot
cvs login
cvs co -r NSPR_4_7_RTM NSPR
cvs co -r NSS_3_12_2_RTM NSS
cd mozilla/security/nss/
make nss_build_all
```

## Linking against your own NSS

Sadly, I don't know of a nice way to do this; I always do

    hammer --verbose net > log 2>&1

then grab the line that links my app and put it into a shell script link.sh,
and edit it to include the line

    DIR=$HOME/nss-3.12.2/mozilla/dist/Linux2.6_x86_glibc_PTH_DBG.OBJ/lib

and insert a `-L$DIR` right before the `-lnss3`.

Note that hammer often builds the app in one, deeply buried, place, then copies
it into Hammer for ease of use. You'll probably want to make your `link.sh` do
the same thing.

Then, after a source code change, do the usual `hammer net` followed by
`sh link.sh`.

Then, to run the resulting app, use a script like

## Running against your own NSS

Create a script named `run.sh` like this:

```sh
#!/bin/sh
set -x
DIR=$HOME/nss-3.12.2/mozilla/dist/Linux2.6_x86_glibc_PTH_DBG.OBJ/lib
export LD_LIBRARY_PATH=$DIR
"$@"
```

Then run your app with

    sh run.sh Hammer/foo

Or, to debug it, do

    sh run.sh gdb Hammer/foo

## Logging

There are several flavors of logging you can turn on.

*   `SSLClientSocketNSS` can log its state transitions and function calls using
    `base/logging.cc`.  To enable this, edit `net/base/ssl_client_socket_nss.cc`
    and change `#if 1` to `#if 0`. See `base/logging.cc` for where the output
    goes (on Linux, it's usually stderr).

*   `HttpNetworkTransaction` and friends can log its state transitions using
    `base/trace_event.cc`. To enable this, arrange for your app to call
    `base::TraceLog::StartTracing()`. The output goes to a file named
    `trace...pid.log` in the same directory as the executable (e.g.
    `Hammer/trace_15323.log`).

*   `NSS` itself can log some events. To enable this, set the environment
    variables `SSLDEBUGFILE=foo.log SSLTRACE=99 SSLDEBUG=99` before running
    your app.

## Network Traces

http://wiki.wireshark.org/SSL describes how to decode SSL traffic. Chromium SSL
unit tests that use `net/base/ssl_test_util.cc` to set up their servers always
use port 9443 with `net/data/ssl/certificates/ok_cert.pem`, and port 9666 with
`net/data/ssl/certificates/expired_cert.pem` This makes it easy to configure
Wireshark to decode the traffic: do

Edit / Preferences / Protocols / SSL, and in the "RSA Keys List" box, enter

    127.0.0.1,9443,http,<path to ok_cert.pem>;127.0.0.1,9666,http,<path to expired_cert.pem>

e.g.

    127.0.0.1,9443,http,/home/dank/chromium/src/net/data/ssl/certificates/ok_cert.pem;127.0.0.1,9666,http,/home/dank/chromium/src/net/data/ssl/certificates/expired_cert.pem

Then capture all tcp traffic on interface lo, and run your test.

## Valgrinding NSS

Read https://developer.mozilla.org/en/NSS_Memory_allocation and do

    export NSS_DISABLE_ARENA_FREE_LIST=1

before valgrinding if you want to find where a block was originally allocated.

If you get unsymbolized entries in NSS backtraces, try setting:

    export NSS_DISABLE_UNLOAD=1

(Note that if you use the Chromium valgrind scripts like
`tools/valgrind/chrome_tests.sh` or `tools/valgrind/valgrind.sh` these will both
be set automatically.)

## Support forums

If you have nonconfidential questions about NSS, check
[the newsgroup](http://groups.google.com/group/mozilla.dev.tech.crypto).
The NSS maintainer monitors that group and gives good answers.
