# Mojo in Chromium

This document is intended to serve as a Mojo primer for Chromium developers. No
prior knowledge of Mojo is assumed.

[TOC]

## Should I Bother Reading This?

If you're planning to build a Chromium feature that needs IPC and you aren't
already using Mojo, YES! **Legacy IPC is deprecated.**

## Why Mojo?

TL;DR: The long-term intent is to refactor Chromium into a large set of smaller
services.

We can be smarter about:

  * Which services we bring up or don't
  * How we isolate these services to improve security and stability
  * Which binary features we ship to one user or another

A more robust messaging layer opens the door for a number of interesting
possibilities; in particular it allows us to integrate a large number of
components without link-time interdependencies, and it breaks down the growing
number of interesting cross-language boundaries across the codebase.

Much has been learned from using Chromium IPC and maintaining Chromium
dependencies in anger over the past several years and we feel there's now a
significant opportunity to make life easier for developers, and to help them
build more and better features, faster, and with much less cost to users.

## Mojo Overview

The Mojo system API provides a small suite of low-level IPC primitives:
**message pipes**, **data pipes**, and **shared buffers**. On top of this API
we've built higher-level bindings APIs to simplify messaging for consumers
writing C++, Java, or JavaScript code.

This document focuses primarily on using C++ bindings with message pipes, which
is likely to be the most common usage encountered by Chromium developers.

### Message Pipes

A message pipe is a lightweight primitive for reliable bidirectional transfer of
relatively small packets of data. Unsurprisingly a pipe has two endpoints, and
either endpoint may be transferred over another message pipe.

Because we bootstrap a primordial message pipe between the browser process and
each child process, this in turn means that you can create a new pipe and
ultimately send either end to any process, and the two ends will still be
able to talk to each other seamlessly and exclusively. Goodbye, routing IDs!

While message pipes can carry arbitrary packets of unstructured data we
generally use them in conjunction with generated bindings to ensure a
consistent, well-defined, versioned message structure on all endpoints.

### Mojom

Mojom is the IDL for Mojo interfaces. Given a `.mojom` file, the bindings
generator outputs bindings for all three of the currently supported languages.

For example:

```
// src/components/frob/public/interfaces/frobinator.mojom
module frob.mojom;

interface Frobinator {
  Frobinate();
};
```

would generate the following outputs:

```
out/Debug/gen/components/frob/public/interfaces/frobinator.mojom.cc
out/Debug/gen/components/frob/public/interfaces/frobinator.mojom.h
out/Debug/gen/components/frob/public/interfaces/frobinator.mojom.js
out/Debug/gen/components/frob/public/interfaces/frobinator.mojom.srcjar
...
```

The generated code hides away all the details of serializing and deserializing
messages on either end of a pipe.

The C++ header (`frobinator.mojom.h`) defines an abstract class for each
mojom interface specified. Namespaces are derived from the `module` name.

**NOTE:** Chromium convention for component `foo`'s module name is `foo.mojom`.
This means all mojom-generated C++ typenames for component `foo` will live in
the `foo::mojom` namespace to avoid collisions with non-generated typenames.

In this example the generated `frob::mojom::Frobinator` has a single
pure virtual function:

```
namespace frob {

class Frobinator {
 public:
  virtual void Frobinate() = 0;
};

}  // namespace frob
```

To create a `Frobinator` service, one simply implements `foo::Frobinator` and
provides a means of binding pipes to it.

### Binding to Pipes

Let's look at some sample code:

```
// src/components/frob/frobinator_impl.cc

#include "components/frob/public/interfaces/frobinator.mojom.h"
#include "mojo/public/cpp/bindings/binding.h"
#include "mojo/public/cpp/bindings/interface_request.h"

namespace frob {

class FrobinatorImpl : public mojom::Frobinator {
 public:
  FrobinatorImpl(mojo::InterfaceRequest<mojom::Frobinator> request)
      : binding_(this, std::move(request)) {}
  ~FrobinatorImpl() override {}

  // mojom::Frobinator:
  void Frobinate() override { DLOG(INFO) << "I can't stop frobinating!"; }

 private:
  mojo::Binding<mojom::Frobinator> binding_;
};

}  // namespace frob
```

The first thing to note is that `mojo::Binding<T>` *binds* one end of a message
pipe to an implementation of a service. This means it watches that end of the
pipe for incoming messages; it knows how to decode messages for interface `T`,
and it dispatches them to methods on the bound `T` implementation.

`mojo::InterfaceRequest<T>` is essentially semantic sugar for a strongly-typed
message pipe endpoint. A common way to create new message pipes is via the
`GetProxy` call defined in `interface_request.h`:

```
mojom::FrobinatorPtr proxy;
mojo::InterfaceRequest<mojom::Frobinator> request = mojo::GetProxy(&proxy);
```

This creates a new message pipe with one end owned by `proxy` and the other end
owned by `request`. It has the nice property of attaching common type
information to each end of the pipe.

Note that `InterfaceRequest<T>` doesn't actually **do** anything. It just scopes
a pipe endpoint and associates it with an interface type at compile time. As
such, other typed service binding primitives such as `mojo::Binding<T>` take
these objects as input when they need an endpoint to bind to.

`mojom::FrobinatorPtr` is a generated type alias for
`mojo::InterfacePtr<mojom::Frobinator>`. An `InterfacePtr<T>` scopes a message
pipe endpoint as well, but it also internally implements every method on `T` by
serializing a corresponding message and writing it to the pipe.

Hence we can put this together to talk to a `FrobinatorImpl` over a pipe:

```
frob:mojom::FrobinatorPtr frobinator;
frob::FrobinatorImpl impl(GetProxy(&frobinator));

// Tada!
frobinator->Frobinate();
```

Behind the scenes this serializes a message corresponding to the `Frobinate`
request and writes it to one end of the pipe. Eventually (and incidentally,
very soon after), `impl`'s internal `mojo::Binding` will decode this message and
dispatch a call to `impl.Frobinate()`.

**NOTE:** In this example the service and client are in the same process, and
this works just fine. If they were in different processes (see the example below
in [Exposing Services in Chromium](#Exposing-Services-in-Chromium)), the call
to `Frobinate()` would look exactly the same!

### Responding to Requests

A common idiom in Chromium IPC is to keep track of IPC requests with some kind
of opaque identifier (i.e. an integer *request ID*) so that you can later
respond to a specific request using some nominally related message in the other
direction.

This is baked into mojom interface definitions. We can extend our `Frobinator`
service like so:

```
module frob.mojom;

interface Frobinator {
  Frobinate();
  GetFrobinationLevels() => (int min, int max);
};
```

and update our implementation:

```
class FrobinatorImpl : public mojom::Frobinator {
 public:
  // ...

  // mojom::Frobinator:
  void Frobinate() override { /* ... */ }
  void GetFrobinationLevels(const GetFrobinationLevelsCallback& callback) {
    callback.Run(1, 42);
  }
};
```

When the service implementation runs `callback`, the response arguments are
serialized and sent back over the pipe. The proxy on the other end knows how to
read this response and will in turn dispatch it to a callback on that end:

```
void ShowLevels(int min, int max) {
  DLOG(INFO) << "Frobinator min=" << min << " max=" << max;
}

// ...

  mojom::FrobinatorPtr frobinator;
  FrobinatorImpl impl(GetProxy(&frobinator));

  frobinator->GetFrobinatorLevels(base::Bind(&ShowLevels));
```

This does what you'd expect.

## Exposing Services in Chromium

There are a number of ways one might expose services across various surfaces of
the browser. One common approach now is to use a
[`content::ServiceRegistry` (link)](https://goo.gl/uEhx06). These come in
pairs generally spanning a process boundary, and they provide primitive service
registration and connection interfaces. For one example, [every
`RenderFrameHost` has a `ServiceRegistry`](https://goo.gl/4YR3j5), as does
[every corresponding `RenderFrame`](https://goo.gl/YhrgXa). These registries are
intertwined.

The gist is that you can add a service to the local side of the registry -- it's
just a mapping from interface name to factory function -- or you can connect by
name to services registered on the remote side.

**NOTE:** In this context the "factory function" is simply a callback which
takes a pipe endpoint and does something with it. It's expected that you'll
either bind it to a service implementation of some kind or you will close it, effectively rejecting the connection request.

We can build a simple browser-side `FrobinatorImpl` service that has access to a
`BrowserContext` for any frame which connects to it:

```
#include "base/macros.h"
#include "components/frob/public/interfaces/frobinator.mojom.h"
#include "content/public/browser/browser_context.h"
#inlcude "mojo/public/cpp/system/interface_request.h"
#inlcude "mojo/public/cpp/system/message_pipe.h"
#inlcude "mojo/public/cpp/system/strong_binding.h"

namespace frob {

class FrobinatorImpl : public mojom::Frobinator {
 public:
  FrobinatorImpl(content::BrowserContext* context,
                 mojo::InterfaceRequest<mojom::Frobinator> request)
      : context_(context), binding_(this, std::move(request)) {}
  ~FrobinatorImpl() override {}

  // A factory function to use in conjunction with ServiceRegistry.
  static void Create(content::BrowserContext* context,
                     mojo::InterfaceRequest<mojom::Frobinator> request) {
    // See comment below for why this doesn't leak.
    new FrobinatorImpl(context,
                       mojo::MakeRequest<mojom::Frobinator>(std::move(pipe)));
  }

 private:
  // mojom::Frobinator:
  void Frobinate() override { /* ... */ }

  content::BrowserContext* context_;

  // A StrongBinding is just like a Binding, except that it takes ownership of
  // its bound implementation and deletes itself (and the impl) if and when the
  // bound pipe encounters an error or is closed on the other end.
  mojo::StrongBinding<mojom::Frobinator> binding_;

  DISALLOW_COPY_AND_ASSIGN(FrobinatorImpl);
};

}  // namespace frob
```

Now somewhere in the browser we register the Frobinator service with each
`RenderFrameHost` ([this](https://goo.gl/HEFn63) is a popular spot):

```
frame_host->GetServiceRegistry()->AddService<frob::mojom::Frobinator>(
    base::Bind(
        &frob::FrobinatorImpl::Create,
        base::Unretained(frame_host->GetProcess()->GetBrowserContext())));
```

And in the render process we can now do something like:

```
mojom::FrobinatorPtr frobinator;
render_frame->GetServiceRegistry()->ConnectToRemoteService(
    mojo::GetProxy(&frobinator));

// It's IPC!
frobinator->Frobinate();
```

There are now plenty of concrete examples of Mojo usage in the Chromium tree.
Poke around at existing mojom files and see how their implementions are built
and connected.

## Mojo in Blink

*TODO*

This is a work in progress. TL;DR: We'll also soon begin using Mojo services
from Blink so that the platform layer can consume browser services
directly via Mojo. The long-term goal there is to eliminate `content/renderer`.

## Questions, Discussion, etc.

A good place to find highly concentrated doses of people who know and care
about Mojo in Chromium would be the [chromium-mojo](https://goo.gl/A4ebWB)
mailing list[.](https://goo.gl/L70ihQ)
