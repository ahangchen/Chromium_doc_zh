A zygote process is one that listens for spawn requests from a master process
and forks itself in response. Generally they are used because forking a process
after some expensive setup has been performed can save time and share extra
memory pages.

On Linux, for Chromium, this is not the point, and measurements suggest that the
time and memory savings are minimal or negative.

We use it because it's the only reasonable way to keep a reference to a binary
and a set of shared libraries that can be exec'ed. In the model used on Windows
and Mac, renderers are exec'ed as needed from the chrome binary. However, if the
chrome binary, or any of its shared libraries are updated while Chrome is
running, we'll end up exec'ing the wrong version. A version _x_ browser might be
talking to a version _y_ renderer. Our IPC system does not support this (and
does not want to!).

So we would like to keep a reference to a binary and its shared libraries and
exec from these. However, unless we are going to write our own `ld.so`, there's
no way to do this.

Instead, we exec the prototypical renderer at the beginning of the browser
execution. When we need more renderers, we signal this prototypical process (the
zygote) to fork itself. The zygote is always the correct version and, by
exec'ing one, we make sure the renderers have a different address space
randomisation than the browser.

The zygote process is triggered by the `--type=zygote` command line flag, which
causes `ZygoteMain` (in `chrome/browser/zygote_main_linux.cc`) to be run. The
zygote is launched from `chrome/browser/zygote_host_linux.cc`.

Signaling the zygote for a new renderer happens in
`chrome/browser/child_process_launcher.cc`.

You can use the `--zygote-cmd-prefix` flag to debug the zygote process. If you
use `--renderer-cmd-prefix` then the zygote will be bypassed and renderers will
be exec'ed afresh every time.
