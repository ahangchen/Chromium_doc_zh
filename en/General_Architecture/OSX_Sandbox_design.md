# OSX Sandboxing Design
This document describes the process sandboxing mechanism used on Mac OS X.
## Background

Sandboxing treats a process as a hostile environment which at any time can be compromised by a malicious attacker via buffer overruns or other such attack vectors. Once compromised, the goal is to allow the process in question access to as few resources of the user's machine as possible, above and beyond the standard file-system access control and user/group process controls enforced by the kernel.

See the overview document for goals and general architectural diagrams.
## Implementation

On Mac OS X versions starting from Leopard, individual processes can have their privileges restricted using the sandbox(7) facility of BSD, also referred to in some Apple documentation as "Seatbelt". This is made up of a single API call, sandbox_init(), which sets the access restrictions of a process from that point on. This means that previously opened file descriptors continue working even if the new privileges would deny access to newly created descriptors. We can use this to our advantage by setting up everything correctly at the start of the process then cutting off all access before we expose the renderer to any 3rd party input (html, etc).

Seatbelt does not place restrictions on memory allocation, threading, or access to previously opened OS facilities. As a result, this shouldn't impose any additional requirements or drastically alter our IPC designs. 

OS X provides additional protection against buffer overflows. In Leopard, the stack is marked as non-executable memory and thus less susceptible as an attack vector for executing malicious code. This doesn't prevent against buffer overruns in the heap, but for 64-bit apps, Leopard disallows any attempts to execute code unless that portion of memory is explicitly marked as executable. As we move towards 64-bit render processes in the future, this will be another attractive security feature. 

sandbox_init() has supports for both predefined sandbox access restrictions and sandbox profile scripts which provide finer grained control.

Chromium uses custom sandbox profiles defined in .sb files in the source tree.

The following profiles are defined (paths relative to root of source directory):
* content/common/common.sb - used for common setup for all sandboxes.
* content/renderer/renderer.sb - used for the extension & renderer processes. Enables access to the font server.
* chrome/browser/utility.sb - used by the utility process. Allows access to a single configurable directory.
* content/browser/worker.sb - used by the worker process.  Most restrictive - no file system access apart from loading system libraries.
* chrome/browser/nacl_loader.sb - used for running Native Client untrusted (i.e., "user") code.

One sticky point we run into is that the sandboxed process calls through to OS X system APIs. There is no documentation available about which privileges each API needs, such as whether they need access to on-disk files, or call other APIs to which the sandbox restricts access. Our approach to date has been to "warm up" any problematic API calls before turning the sandbox on. This means that we call through to the API, to allow it to cache whatever resource it needs. For example, color profiles and shared libraries can be loaded from disk before we "lock down" the process.

SandboxInitWrapper::InitializeSandbox() is the main entry point for initializing the Sandbox, it performs the following steps:
* Determines if the current process type needs to be sandboxed and if so, which sandbox configuration to use.
* "Warm up" relevant system APIs by calling through to  sandbox::SandboxWarmup() .
* Enable the Sandbox by calling through to  sandbox::EnableSandbox() .

### Diagnostics

The OS X sandbox implementation supports the following flags:
* --no-sandbox - Disable the sandbox entirely.
* --enable-sandbox-logging - Verbose information about which system calls are blocked is logged to syslog.

[Debugging Chrome on OS X](https://www.chromium.org/developers/how-tos/debugging-on-os-x) contains more documentation on debugging and diagnostic tools provided by the Mac OS X sandbox API.

## Additional Reading

* http://reverse.put.as/wp-content/uploads/2011/09/Apple-Sandbox-Guide-v1.0.pdf
* http://www.318.com/techjournal/?p=107
* sandbox man page (man 7 sandbox)
* System sandbox files can be found under one of the following paths (depending on the OS Version):

  * /Library/Sandbox/Profiles
  * /System/Library/Sandbox/Profiles
  * /usr/share/sandbox