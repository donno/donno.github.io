Internet
========

Place holder / reminder to perform a write-up on Internet.

Topics to cover
* Border Gateway Protocol (BGP)
* Autonomous System (AS) Number.
* Domain Name System (DNS)
* Internet Exchange Points
* Peering
* Internet Protocol
    * V4
    * V6
* Protocols
    * TCP
    * UDP
    * HTTP
    * WHOIS
    * RDAP
* Software
    * CoreDNS - DNS Server written in Go.
    * BIND - DNS
* References

## Border Gateway Protocol

A starting point would be to visit [https://bgp.tools/][bgp-tools], which
gives you an idea on your current connection to the Internet. It provides the
assigned number (AS) that you are connecting.

It provides way to look-up a Autonomous System Number (ASN), IP Prefix,
domain name or a MAC address.

It typically runs on TCP port 179.

### States

* Idle
* Connect
* Active
* Open-Sent
* Open-Confirm
* Established

### Implementations

* BIRD Internet Routing Daemon ([BIRD][bird])
    * Started at  Faculty of Mathematics and Physics, Charles University, Prague.
    * Licened via GNU General Public License.
    * Widely used
* OpenBSD Border Gateway Protocol Daemon ([OpenBGPD][openbgpd])
    * Started as part of OpenBSD project
    * Licened via the ISC license.
    * A recent effort saw it ported to other platforms (i.e. Linux).

## Autonomous system (AS) Number

The [Internet Assigned Numbers Authority][iana] (IANA) assigns these numbers to
regional Internet registries (RIRs) in blogs.

The starting point for this is the [AS numbers]([as-numbers) page from
[IANA][iana]. This lists the RIRs which at the time of writing there are 5 of
them. See the next section for details.


### Textual Representation of Autonomous System
This is covered by [Request for Comment 5396][rfc5396].

Three formats:
* asplain
    * Applies to all AS numbers
    * Format is a decimal integer notation.
    * IANA Regstiries should use this representation for AS numbers.
    * Example: AS number of value 65526 would be represented as string "65526".
* asdot+
    * Applies to all AS numbers.
    * Format is a two integer values joined by a period character.
    * Format: [high-order 16-bit value in decimal].[low order 16-bit value in decimal].
    * Examples
        * AS number 65526 becomes "0.65526"
        * AS number 65546  becomes "1.10"
* asdot
    * Represent AS numbers less than 65536 the same as `asplain` and equal or greater
      using `asdot+`.
    * Example: AS number of value 65546 would be represented as string "1.1065526".

## Regional Internet address registry

* AFRINIC - Africa
* APNIC - Asia-Pacific region
    * whois.apnic.net
    * https://rdap.apnic.net/
* ARIN - American Registry for Internet Number. This covers Canada, United
  States, some Caribbean nations.
    * whois.arin.net
    * https://rdap.arin.net/registry
* LACNIC - Latin American and Caribbean Internet Addresses Registry. This
  covers Latin America, some Caribbean nations
    * whois.lacnic.net
    * https://rdap.lacnic.net/rdap/
* RIPE NCC - Réseaux IP Européens Network Coordination Centre. This covers
  Europe, Russia, Middle East, Central Asia.
    * whois.ripe.net
    * https://rdap.db.ripe.net/

### APNIC
The Asia-Pacific region provides daily records of their assignments at:
https://ftp.apnic.net/stats/apnic/


### RIR statistics exchange format
The [documentation][apnic-rir-format] for the RIR statistics exchange format
by APNIC is [available][apnic-rir-format].

TODO: Summarise the file format here.

## Software

### CoreDNS

* DNS Server written in Go.
* Powered by plugins - selected set of some of note.
    * file plugin
        * Web page: https://coredns.io/plugins/file/
        * Serves data from disk in the RFC 1035 style data.
    * hosts plugin
        * Web page: https://coredns.io/plugins/hosts/
        * Serves files via a `/etc/hosts` style file.
    * prometheus
        * https://coredns.io/plugins/metrics/
        * Store metrics for the service in

TODO: Explore the file one.
#### Basic Example - Forward
This example forwards the DNS request to another server.

Create a file named `Corefile` and run `coredns`.
```
. {
    forward . 8.8.8.8 9.9.9.9
    log
}
```

Client output
```
$ nslookup duckduckgo.com 127.0.0.1
Server:  localhost
Address:  127.0.0.1

Non-authoritative answer:
Name:    duckduckgo.com
Address:  20.43.111.112
```

Server output
```
$ coredns
maxprocs: Leaving GOMAXPROCS=8: CPU quota undefined
.:53
CoreDNS-1.13.2
windows/amd64, go1.25.5, 0233f3e
[INFO] 127.0.0.1:60803 - 1 "PTR IN 1.0.0.127.in-addr.arpa. udp 40 false 512" NOERROR qr,rd,ra 85 0.0120281s
[INFO] 127.0.0.1:60805 - 2 "A IN duckduckgo.com. udp 32 false 512" NOERROR qr,rd,ra 62 0.0110022s
[INFO] 127.0.0.1:60806 - 3 "AAAA IN duckduckgo.com. udp 32 false 512" NOERROR qr,rd,ra 120 0.018047s
[INFO] SIGINT: Shutting down
```

## References

### How to become your own ISP
A talk by Nick Bouwhuis given at WHY (What Hackers Yearn) 2025.

Recording: https://media.ccc.de/v/why2025-9-how-to-become-your-own-isp

Overall this has a do-it-yourself angle and covers the kind of hardware you
need, getting rack space in a right data center, etc, however it provides a
good overview of everything else.

[iana]: https://www.iana.org/
[as-numbers]: https://www.iana.org/assignments/as-numbers/as-numbers.xhtml
[bgp-tools]: https://bgp.tools/
[coredns]: https://coredns.io
[bind]: https://www.isc.org/bind/
[apnic-rir-format]: https://www.apnic.net//about-apnic/corporate-documents/documents/resource-guidelines/rir-statistics-exchange-format/
[bird]: https://bird.network.cz/
[openbgpd]: https://www.openbgpd.org/
[rfc5396]: https://www.rfc-editor.org/rfc/rfc5396