---
layout: post
title: Twisted Python and Bonjour
category: python
---

[Bonjour][] (formerly Rendezvous) is Apple's [service discovery][dns-sd]
protocol.  It operates over local networks via [multicast DNS][mdns].  Server
processes announce their availability by broadcasting service records and
their associated ports.  Clients browse the network in search of specific
service types, potentially connecting to the service on the advertised port
using the appropriate network protocol for that service.

A common example of Bonjour in action is iTunes' music library sharing
feature.  iTunes sharing uses [DAAP][] (Digital Audio Access Protocol).
iTunes uses Bonjour to announce its local shared libraries as well as to
browse the network for remote DAAP servers.

[Twisted Python][twisted], while supporting a wide range of network protocols
by default, currently lacks an official Bonjour or multicast DNS
implementation.  The start of a multicast DNS implementation exists in
[Itamar's sandbox](http://twistedmatrix.com/trac/browser/sandbox/itamar/mdns),
but it hasn't been updated since 2004.

Given that, applications that want to use Bonjour-based service discovery must
provide their own implementation.  Unfortunately, there can only be one
Bonjour "responder" running on a system at one time.  If multiple applications
attempted to advertise services by standing up competing responders, a port
conflict would arise.  Therefore, all of the Bonjour-aware applications
running on a system must coordinate.

On Mac OS 10.2 and later, applications can simply communicate with the
operating system's built-in Bonjour service.  Most other operation systems
don't provide native Bonjour functionality, but support is generally available
via third-party packages.  Apple provides [Bonjour for Windows][bonjour-win],
and the LGPL-licensed [Avahi][] runs on most other platforms.

Supporting multiple potential Bonjour interfaces can be a burden for
application developers.  Fortunately, for Python-based projects, [pybonjour][]
exists to provide a very nice [ctypes][]-based abstraction layer to all of the
Bonjour-compatible libraries mentioned above.

pybonjour's public API is based on "service descriptors".  Each operation
returns a service descriptor reference and signals the caller via a callback
when the operation completes.  Service descriptors can generally be treated
like read-only file descriptors, but all read operations must be done using
pybonjour's `DNSServiceProcessResult()` function.

We can easily wrap a pybonjour service descriptor in an object that implements
Twisted's `IReadDescriptor` interface:

{% highlight python %}

    import pybonjour
    from twisted.internet.interfaces import IReadDescriptor
    from zope import interface

    class ServiceDescriptor(object):

        interface.implements(IReadDescriptor)

        def __init__(self, sdref):
            self.sdref = sdref

        def doRead(self):
            pybonjour.DNSServiceProcessResult(self.sdref)

        def fileno(self):
            return self.sdref.fileno()

        def logPrefix(self):
            return "bonjour"

        def connectionLost(self, reason):
            self.sdref.close()

{% endhighlight %}

Then, it's simply a matter of writing some operation-specific functions that
join the pybonjour interface with Twisted's event-driven concepts.  These
functions initiate the pybonjour request, handle the pybonjour callback, and
dispatch the result using a Twisted `Deferred`.

The following example broadcasts a new service record.  Note that the local
callback function handles both the success and error results.

{% highlight python %}

    from twisted.internet.defer import Deferred

    def broadcast(reactor, regtype, port, name=None):
        def _callback(sdref, flags, errorCode, name, regtype, domain):
            if errorCode == pybonjour.kDNSServiceErr_NoError:
                d.callback((sdref, name, regtype, domain))
            else:
                d.errback(errorCode)

        d = Deferred()
        sdref = pybonjour.DNSServiceRegister(name = name,
                                            regtype = regtype,
                                            port = port,
                                            callBack = _callback)

        reactor.addReader(ServiceDescriptor(sdref))
        return d

{% endhighlight %}

The caller can then provide callback and errback functions that will be
invoked using Twisted's event-based reactor machinery.

{% highlight python %}

    from twisted.internet import reactor
    from twisted.python import log

    sdref = None

    def broadcasting(args):
        global sdref
        sdref  = args[0]
        log.msg('Broadcasting %s.%s%s' % args[1:])

    def failed(errorCode):
        log.err(errorCode)

    d = broadcast(reactor, "_daap._tcp", 3689, "DAAP Server")
    d.addCallback(broadcasting)
    d.addErrback(failed)

{% endhighlight %}

To stop broadcasting, simply close the service descriptor (`sdref.close()`).

[twisted]: http://twistedmatrix.com/
[bonjour]: http://en.wikipedia.org/wiki/Bonjour_(software)
[bonjour-win]: http://apple.com/support/downloads/bonjourforwindows.html
[avahi]: http://avahi.org/
[dns-sd]: http://www.dns-sd.org/
[mdns]: http://www.multicastdns.org/
[daap]: http://en.wikipedia.org/wiki/Digital_Audio_Access_Protocol
[pybonjour]: http://code.google.com/p/pybonjour/
[ctypes]: http://docs.python.org/library/ctypes.html
