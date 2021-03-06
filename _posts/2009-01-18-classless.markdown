---
layout: post
title: Classless in-addr.arpa. Delegation
---

Classless in-addr.arpa. delegation allows network administrators to provide
authoritative reverse DNS on subnets that don't fall on octet boundaries.
This is especially useful for subnets comprised of less than eight bits in the
host portion of the address (i.e. smaller than a class C).

There are two important things to remember: first, we're dealing with
*classless* subnets, meaning they don't align themselves neatly with IPv4's
octet boundaries (like a class A, B, C, D, or E network); and second, only one
name server can be the primary authoritative controller of a given class of IP
addresses.

In order for a non-octet subnet network administrator to properly provide
reverse DNS for his hosts, he will require the cooperation of his upstream
provider.  This goes back to the second rule above: the upstream provider
technically controls the complete class of addresses out of which the
downstream subnet receives its address space.

I found myself in this position years ago while working for a small web-based
startup.  We were allocated a block of 32 addresses as a /27 subnet (netmask:
255.255.255.224).  What follows is a brief summary (with examples) of how we
went about establishing our name servers as the authorities for reverse DNS in
our zone.

All given configurations were tested using [BIND 8][bind] under both OpenBSD
and FreeBSD circa 2000, but I expect they are still relevant today.

Our assigned subnet was 206.126.7.64/27 (in [CIDR][] notation).  Thus, our
network address was 206.126.7.64 and our broadcast address was 206.126.7.95,
giving us 30 usable host addresses in between. 

We begin by creating a standard zone file (`example-hosts`) for the
`example.com` domain (for "forward" DNS queries):

    @               IN      A       206.126.7.73
    ns              IN      A       206.126.7.65
    [...]
    gateway         IN      A       206.126.7.94

There really isn't anything special going on here.  Simply add this entry to
your `named.conf` file:

    zone "example.com" {
        type master;
        file "example-hosts";
    };

Next, we define the reverse lookup mappings for the 206.126.7.64/27 subnet in
a file named `64-27.7.126.206.rev`:

    65      IN      PTR     ns.example.com.
    [...]
    94      IN      PTR     gateway.example.com.

Now, for this file, we add a slightly different entry to `named.conf`:

    zone "64/27.7.126.206.in-addr.arpa" {
        type slave;     
        file "64-27.7.126.206.rev";
        masters {
                206.126.7.65;
        };
    };

Be sure to list your provider's name server(s) in the `masters {}` block.  As
far as the provider is concerned, they are still the authoritative "master" of
this IP address zone even though they delegate the mappings within our subnet
to us.  We must act as the slave and participate in zone transfers.

That's really all the subnet administrator has to worry about.  The rest of
the work needs to be done on the provider's side.  In order for them to
delegate the reverse lookups to the downstream name servers, we employ (or
perhaps abuse) the `CNAME` record. These entries go in a file named
`7.126.206.rev`.  Note that this file contains entries for the entire
206.126.7.0/24 class (C) of addresses, not just our 206.126.7.64/27 subnet.

    ;; example.com
    ;
    ; Name servers for 206.126.7.64/27
    ;
    64/27           NS      ns.example.com.     ; 206.126.7.65
    64/27           NS      raq.example.com.    ; 206.126.7.66
    ;
    ; Classless in-addr.arpa. delegation for 206.126.7.64/27
    ;
    65              CNAME   65.64/27.7.126.206.in-addr.arpa.
    [...]
    94              CNAME   94.64/27.7.126.206.in-addr.arpa.

The provider's `named.conf` should contain an entry like:

    zone "7.126.206.in-addr.arpa" {
        type master;
        file "7.126.206.rev";
    };

That's really all there is to it.  It took me a couple of hours to muddle my
way through all of the details (and making a few silly mistakes along the way
didn't help much either), but it's really not that complicated.  When a
reverse DNS query is issued, it's sent to the authoritative name server (the
provider's), who in turn delegates the request to the subnet's name servers,
who then get the final say.  This is transparent to the querying client, and
everyone is happy in the end. 

For more (reputable) information on this topic, see [RFC 2317][rfc2317].

You may also want to check out [Avoid RFC 2317 style delegation of
"in-addr.arpa."](http://homepages.tesco.net/J.deBoynePollard/FGA/avoid-rfc-2317-delegation.html), which describes an alternative to RFC 2317.  I offer no
opinion as to which method is the best solution for a given situation,
however.

[bind]: https://www.isc.org/software/bind
[cidr]: http://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing
[rfc2317]: http://tools.ietf.org/html/rfc2317
