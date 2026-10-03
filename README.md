# SPOD Specification version 1 (draft-1)

_SPOD_ is a recursive backronym for **Spod Protocol and Option Discovery**, where the
original meaning of "spod" is [given by Wiktionary.org][wiktionary-spod] as:

> (1) One who uses talkers, an early form of chat room.  
> (2) One who wastes time on nonproductive activities online.

[wiktionary-spod]: https://en.wiktionary.org/wiki/spod

This specification defines a convention that uses DNS records to inform compatible
clients how to connect to a Talker, MUD, BBS, or similar text-based interactive
program starting from minimal information such as a fully-qualified domain name or
a commonly known moniker.


## Background

The peak era of Internet-connected BBSes, MUDs, and Talkers occurred in the 1990s
and connections were commonly made using the TCP-based *Telnet* protocol on nonstandard
port numbers.  Unencrypted Telnet was, then, a common way to remotely connect to systems,
just as unencrypted HTTP was the way to retrieve and view Hypertext documents.
Both Telnet and HTTP have IANA-assigned well-known port numbers, 23 and 80, respectively.
The HTTP protocol uses URI _paths_ to segment different offerings or content from perhaps
different users on the same host, but the Telnet protocol only offers a socket-based
connection with no built-in way to request or route to different content or providers
within the same host.

Port 23 was usually reserved for accessing the host's operating system _shell_.
Connecting to other services, like Talkers, required knowledge of an alternative port
number in addition to the host's IP address or DNS name.  This additional information
could be encoded into URLs on Web pages (e.g., `telnet://example.com:1234`) but this
placed undue burden on end-users, namely:

1. It forces end-users to learn about the existence of TCP port numbers, whereas most
   other protocols like HTTP/S can rely on standardized port numbers by default that
   "just work" after the user provides a fully-qualified domain name.

2. It requires users to dissect URLs into their constituent parts; they must recognize
   that the _scheme_ dictates the protocol and that the _FQDN or host_ is separated from
   the _port number_ by a colon(:).

3. It requires translation into a usable command line.  While graphical terminal
   emulator programs were growing in number, the default and often only available
   client on end-user systems was the `telnet` program run from the local system's shell.
   While some Web browsers _could_ launch a registered program for `telnet://` URLs, in
   many cases what the user needed was the command-line syntax (e.g.,
   `telnet example.com 1234`).

In the modern day, current best practices actively discourage the use of Telnet as
a protocol due to its unencrypted nature.  Operating systems switched to the Secure
Shell (SSH) protocol, usually on the IANA assigned TCP port 22, and many no longer
provide a pre-installed `telnet` client by default.

MUDs and Talkers still exist, though in fewer numbers.  These interactive services
would better serve their users' privacy by also switching to use SSH.  Though SSH
has a concept of _channels_, which could be used by a client to connect to something
_other than the OS shell_ on the same IP/port endpoint, in practice the operating
system's `sshd` daemon on port 22 is *too sensitive* to be shared with lower-privileged
services -- so the original issues above are not solved merely by switching protocols.

Furthermore, a service planning to switch from one protocol to another may wish
to offer both protocols in unison during a transition period.  In that case it will
use two distinct TCP port numbers, at minimum, and still none of them aligning with
Telnet nor SSH defaults.  Clients have no way to automatically discover when the
service begins offering a new protocol, and on what port, or when the old protocol
becomes the less-preferred one or ceases to be offered.


## Objective

This specification intends to define a lightweight means of using existing DNS
record types to solve the challenges above, while being client, server, and protocol
agnostic.  A successful solution requires the user to use a client that facilitates
making the necessary DNS queries and interpreting the responses according to this
specification, and one that natively supports one or more common protocols (like
Telnet, SSH, or HTTPS) -- or is at least able to delegate to another program that does.

Additionally, to designate the target service for a connection, the user will need
one of the following:

1. A non-qualified, "friendly" name, e.g. "coolchat",
   _and_ a (possibly pre-configured) SPOD directory domain, e.g. `_spod.example.com`
2. Or a fully-qualified domain name, e.g. `coolchat.other.example.org`

A hypothetical command-line client, `spod`, might support initiating a connection
in any of three ways:

    $ spod coolchat                                # using a default directory domain
    $ spod coolchat --directory _spod.example.com  # search within a given domain
    $ spod coolchat.other.example.org              # direct FQDN

The client would then use DNS to determine the protocol and port number required to
connect.  Clients SHOULD automatically prefer more secure protocols over any legacy
protocols such as Telnet; clients MAY allow users to override this preference.  A
client, after determining these details, MAY execute or delegate to another client
program (e.g. `telnet` or `ssh`) to broker the actual connection.

Clients MAY support optional syntax to pass a desired _handle_ (username) as a
convenience to end-users, e.g:

    $ spod myuser@coolchat



## Components


### Service

This specification assumes that a _service_ is a text-based interactive program,
such as a <abbr title="Multi-User Dungeon">MUD</abbr>, Talker, or
<abbr title="Bulletin Board Service">BBS</abbr>; but this is not a strict requirement.
This specification can support discovery of any arbitrary program that offers IP-based
access to the public over the Internet using well-known application layer protocols,
usually over TCP or UDP with at least one associated port number.


### Service names (monikers)

Practically, there are multiple identifiers for a service; it has a human-readable name
it displays to users, it most likely has a FQDN used to locate its host(s) on the Internet,
and it will need yet another identifier to distingush it from other entries in an external
index such as a third-party directory described by this specification.

Within a single directory instance, each service SHALL have a single name, or moniker,
that is unique within that directory.  Since there can be multiple, independent
directory implementations a given service MAY be known by more than one name; also, a
given name that appears in different directory instances MAY point to different target
services in each.  That is, the service moniker "A" in directory "Foo" need not point to
the same actual service implementation as the service moniker "A" in directory "Bar".

Directories that are compliant with this specification MUST use service name monikers that
are representable as a single _label_ component of a DNS hostname; this implies that service
names are also case-insensitive.  Service names SHOULD be limited to printable US-ASCII
characters as allowed in DNS labels and SHOULD NOT use punycode or other extended character
set mechanisms.  Additionally, SPOD service name monikers MUST NOT begin with an underscore(`_`)
character as DNS labels beginning with that character are _reserved_ by this specification.


### Client

Within this specification, a _client_ is a program running on an end-user device that
facilitates establishing connections to named services.  Establishing a connection often
requires finding a fully-qualified domain name (FQDN) and ultimately an Internet protocol
(IP) address, a designated port number, transport protocol (TCP or UDP), and an
appropriate application layer protocl (Telnet, SSH, HTTPS, etc.).  Clients that are
compliant with this specification MAY use a compliant _directory_ (see below) to ease
the burden placed upon end users trying to connect to services.

This specification is only concerned with discovering and communicating the necessary
information to make a connection attempt possible.  The actual establishment of a
connection through socket APIs, protocol negotiation, and subsequent interfacing with
a remote service is not in scope.


### Directory

A _directory_ exists to index and assist the discovery of _services_ by _clients_
and to inform clients of the necessary details to successfully connect to such services.
A directory that is compliant with this specification is referred to as a _SPOD directory_.

A directory is not assumed to itself run or host any of the services it lists, though
a directory operator MAY opt to do so.  A directory MAY contain entries for services
without the knowledge or consent of the operators of those services, similar to how any
Web page may link to any other.  Well-behaved directories SHOULD take steps to prevent
malicious or abusive listings, though how to accomplish this is outside the scope of
this specification.

Clients SHOULD NOT use information discovered from a directory for security-impacting
decisions or functions without consent or acknowledgement from its end-user.  When a
client has previously resolved a service from a particular directory and later attempts
a new resolution, the client SHOULD alert the user if the current FQDN reported by
the directory does not match the last known FQDN for the same service from that directory;
when a change is detected, clients SHOULD NOT proceed with any connection attempts
prior to obtaining positive consent from the end-user.  When the FQDN changes it could
be due to a benign reason, such as a hosting provider move, or it could be indicative of
a more serious change (or loss) of control of the entry in the directory provider; such
reasons cannot be distinguished.


### DNS zones

The directory lookup mechanism described by this specification relies heavily upon the
public, distributed, and accessible nature of the Domain Name System.  DNS is a robust,
highly-available, decentralized, and (perhaps most importantly) _cheap_ infrastructure
service.  Databases and APIs MAY be used to manage and maintain a directory offering,
but such implementation details MUST be hidden from clients.  A client compatible with
this specification needs only to use and understand DNS to search and make use of the
directory.

This specification uses well-defined, standard DNS record types, though it prescribes a
specific way of using those types.

For maximum compatibility with various DNS servers and providers, only common record types
such as `A`, `AAAA`, `CNAME`, ([RFC 1035][rfc1035]) and `TXT` ([RFC 1464][rfc1464]) SHALL
be used whenever possible.  A standard but newer record type, `SVCB` ([RFC 9460][rfc9460]),
is also required by this specification; however, only domains acting as a _directory domain_
require a DNS server capable of publishing this type; support for `SVCB` is _not required_
on the part of the DNS provider used by a service in order to be listed in a SPOD directory.

It is a purposeful design decision that a service can be listed and discovered within a
directory _without requiring_ any additions or modifications to the service's own DNS records.
Standard DNS records such as A/AAA/CNAME that point to a service's host(s) are usually required
for reasons beyond this specification.  This specification defines certain DNS records that
a service MAY opt-in to use for its domain or zone.

It is assumed that recursive DNS resolvers servicing end-users are able to, at a minimum,
resolve and pass through all record types to their clients.

[rfc1035]: https://datatracker.ietf.org/doc/html/rfc1035
[rfc1464]: https://datatracker.ietf.org/doc/html/rfc1464
[rfc9460]: https://datatracker.ietf.org/doc/html/rfc9460


### SPOD Directory Domain

A _directory domain_ serves as an indirection that allows for simple name lookups
without requiring the user to know a service's fully-qualified domain name (FQDN).
It does require the user to know the directory domain FQDN or, more likely, a domain
pre-configured by default in a given client implementation.

A DNS domain or zone MAY opt-in to participating as a SPOD _directory domain_
by self-publishing a _directory domain record_ (see below).  Clients MUST
query for the _directory domain_ record prior to issuing any other SPOD-related
DNS requests, and clients MUST NOT issue further DNS requests for SPOD records
against that domain if a valid _directory domain record_ is not present.


### SPOD DNS records

The following subsections describe the use of various standard DNS record types.
This specification heavily uses a record type it calls `SPOD TXT`, which is
implemented using the standard `TXT` record type.  Because TXT records are
a general-purpose record type, and commonly used by other standards such as SPF
and DKIM, any such record must be inspected to determine if it is applicable to
this specification.

TXT records consist of key-value pairs, separated by a semicolon(;) and optional
whitespace.  Each key-value pair consists of an alphanumeric key, followed by the
equals(=) symbol, followed by the value.

TXT records acting as _SPOD version 1_ records under this specification MUST begin
with the version key-pair `v=spod1`.  Any TXT record encountered that does not meet
this requirement MUST be ignored as if it did not exist.

#### SPOD TXT record grammar

SPOD TXT records are encoded as a sequence of key=value pairs. The record MUST
begin with the version pair `v=spod1`. Each additional pair is separated by a
single semicolon(`;`) followed by optional whitespace.  Keys are unquoted
alphanumeric tokens.  Values that contain whitespace or characters outside of
US-ASCII MUST be enclosed in double quotes(`"`) and encoded as UTF-8; otherwise,
they are treated as unquoted values.  Quoted values SHOULD NOT contain semicolon
to avoid issues with naive parsing implementations.

The following ABNF defines the required syntax:

```abnf
SPOD-TXT       = VERSION-PAIR *( ";" [SPACE] KEY-VALUE ) *(SPACE)
VERSION-PAIR   = "v" "=" "spod1"
KEY-VALUE      = KEY "=" VALUE
KEY            = ALPHA *( ALPHA / DIGIT )

VALUE          = QUOTED-VALUE / UNQUOTED-VALUE
QUOTED-VALUE   = DQUOTE *( UTF8-CHAR ) DQUOTE
UNQUOTED-VALUE = 1*( %x21-7E ) ; unquoted values must not contain SPACE or ";"

; UTF8-CHAR is any valid UTF-8 character except DQUOTE and control characters.
; See RFC 3629.
UTF8-CHAR      = %x20-21 / %x23-7E / UTF8-2 / UTF8-3 / UTF8-4
UTF8-2         = %xC0-DF %x80-BF
UTF8-3         = %xE0-EF %x80-BF %x80-BF
UTF8-4         = %xF0-F7 %x80-BF %x80-BF %x80-BF
SPACE          = SP / TAB
SP             = %x20
TAB            = %x09
ALPHA          = %x41-5A / %x61-7A
DIGIT          = %x30-39
DQUOTE         = %x22
```

Examples:

```Text
"v=spod1; d=_spod.example.com; n=coolchat"
```

```text
"v=spod1; t=\"Châtroom Français\"; l=\"Paris, FR\"; s=\"Un espace de discussion en français\""
```

When a value contains whitespace or international characters, the value MUST be quoted.
A value like `d=_spod.example.com` is a valid unquoted value because it contains no
whitespace, no semicolon, and no characters outside US-ASCII.



## Record types and purposes


### Directory Domain record

A _directory domain record_ is a SPOD TXT record that designates a DNS (sub)domain
as a directory provider and authorizes clients to make additional DNS queries against
its nameserver(s) in adherence with this specification.

For an example SPOD directory domain of `_spod.example.com`,
its _directory domain record_ SHOULD appear as:

    _spod.example.com.  TXT  "v=spod1; d=_spod.example.com"

The `d` key specifies the FQDN of the directory domain.  A SPOD TXT record whose `d`
value "points to itself" (its own FQDN), as in the above example, indicates that the
QNAME/FQDN is opting-in to acting as the root of a SPOD-compliant directory.

If the `d` key's value does not reference the same FQDN as the record itself, then
that SPOD TXT record IS NOT a _domain directory record_ and clients MUST NOT treat
it as such.  (See the next section instead.)


### Directory Domain Hint record

A SPOD TXT record where the `d` key is present and also the _only_ key other than the
`v=spod1` version pair, and where the `d` key value _differs_ from the record's own
FQDN is considered to be a _directory domain hint_ that points to a _canonical_ directory
domain.

A single fully-qualified domain name MAY NOT direct a hint to more than one directory.
If multiple _directory domain hints_ are present at the same FQDN then clients MUST
discard ALL hints as if no records had been found and the client MUST NOT perform more
SPOD-related DNS queries against this domain address.

For example, if the second-level domain (SLD) `example.com` wanted to advertise that
it hosts a SPOD directory domain (but doesn't use the SLD itself as the root of the
directory) then it MAY publish a hint that "points to" the canonical directory domain:

    example.com.  TXT  "v=spod1; d=_spod.example.com"

A client that is configured to use `example.com` as its SPOD directory MAY use this
record to discover a _canonical_ directory (sub)domain.  After retrieving this
record, a client SHOULD query for a SPOD TXT record at the FQDN indicated by the
`d` key.  If no valid record is found then this is a "dead" hint and MUST be
ignored as if no such directory or hint exists.

If a valid SPOD TXT record exists, and it is a valid _directory domain record_ (see
prior section), then the _canonical_ directory domain has been found and it can be
used for subsequent operations.  If instead another _directory domain hint_ record
is found then the process of looking for a _directory domain record_ repeats using
the new `d` key value as the starting domain.  Clients MUST implement safeguards
to prevent cycles (loops) between one or more hints by remembering all previously
queried FQDNs for a given directory-finding operation and stop if any visited FQDN
is repeated.  Clients SHOULD limit the maximum number of followed hints to a
reasonable number (recommended: five(5) or fewer) before giving up without finding
a canonical directory domain record.


### Directory Service record

A SPOD directory domain contains zero or more records for individual services.
Each service within a directory SHOULD be identified by a single DNS _label_ component
and follow the other conventions and restrictions regarding DNS host names.  A service
name component MUST NOT begin with an underscore(`_`).  Each service's FQDN within the
SPOD directory domain is the concatenation of its service name label -- or moniker --
as a prefix of the directory's FQDN.

Example:  
The `_spod.example.com` directory domain represents the service monikers "coolchat"
and "mudpie" with DNS records at `coolchat._spod.example.com` and
`mudpie._spod.example.com`, respectively.

Service entries in the directory are _primarily_ represented by one or more `SVCB`
(Service Binding) records.  SPOD directory SVCB records MUST NOT use the "Alias"
mode of SVCB records; this means that the `priority` field of such records MUST
be positive and nonzero (i.e., >= 1).

The SVCB record(s), for the example "coolchat" directory entry, may look something like:

    coolchat._spod.example.com.  SVCB 1 coolchat.other.example.org. alpn="ssh" port=2222
    coolchat._spod.example.com.  SVCB 2 coolchat.other.example.org. alpn="telnet" port=2323

The above records indicate that "coolchat" supports _both_ the secure SSH protocol
_and_ the legacy Telnet protocol.  Each record also provides the host FQDN and TCP port
number on which the service offers the respective protocols.  In most cases the target
FQDN for all supported protocols will be the same for a given service, but a service MAY
use different target FQDNs for different protocol offerings.  The service priority field,
in this example, indicates that the SSH is preferred because it has the lower value.

This directory specification is application-layer protocol agnostic.  Compliant directories
MUST include the `alpn` (application layer protocol name) value in its SVCB records; there
is no default or implied protocol.  A given protocol name MUST NOT appear more than once
within the set of SVCB records published for a given service within a single directory.
Some protocols, such as `https` and `h2` or `h3`, can be served from the same port:

    coolchat._spod.example.com.  SVCB 3 coolchat.other.example.org. alpn="h2,https" port=443
    coolchat._spod.example.com.  SVCB 4 coolchat.other.example.org. alpn="https" ; illegal, repeated protocol

Directories MUST combine such protocols into a single SVCB record for a given port number by
setting the `alpn` value to a comma-separated list of the protocol names in the order
preferred by the service (leftmost protocol first).  Clients MUST accept SVCB records that
list multiple protocol names.  A client MUST ignore protocol names that it does not support
or understand; if _none_ of the protocol names in a SVCB record are supported by a client
then it MUST discard the entire record as if did not exist.  When multiple protocols names
are present in a single SVCB record, a client SHOULD attempt those protocols it supports in
the order listed within the SVCB record.

> NOTE:  
> Directories SHOULD NOT publish SVCB records containing the (insecure) `http`
> protocol in the `alpn` field.  Use of insecure protocols SHOULD be discouraged,
> though the `telnet` protocol (also insecure) MAY be published while services
> SHOULD consider migrating to more secure alternatives.

Directories MUST include a `port` value in SVCB records when a service is offered on a
port that is _different than the default port_ for the specified application protocol name.
Separate SVCB records must be used for each distinct port number.  Directories MAY elect
to always provide the port number information, and MUST provide the port number information
when it is unable to determine the default port for the specified `alpn` protocol(s).

The transport protocol (TCP, UDP) SHALL be determined based on the client's knowledge
of its supported `alpn` value(s).  For example, a client that supports making SSH
connections SHOULD include the knowledge that the SSH protocol uses TCP as its transport;
therefore a SVCB record specifying `alpn="ssh"` implies TCP and not UDP.  In the rare
case that an `alpn` protocol is capable of using both TCP and UDP -- such as is the case
with DNS -- the transport protocol selection is out of scope of this specification.  It is
expected that such application layer protocols already provide clients with conventions or
guidance to determine the appropriate transport (e.g., RFC 1123 for DNS required UDP first
with TCP as a fallback, and this was later revised by RFC 7766).  Clients SHOULD NOT be
concerned with determining the transport protocol for `alpn` values they do not support.

Directories MAY target FQDNs that are CNAME records; however, they MUST NOT publish new
or updated SVCB records that target FQDNs that do not ultimately resolve to `A` and/or
`AAAA` address records.  A directory that intends to serve queries to the public Internet
MAY decline to list a service whose FQDN resolves to non-public, non-routable IP address(es).

Directories SHOULD periodically revisit their SVCB records and remove those whose target
FQDNs no longer resolve to valid addresses.  Directories SHOULD periodically revisit their
SVCB records and remove or correct entries where the target FQDN does not respond to the
designated application protocol(s) on the designated port number; this requires the
directory to have some knowledge of commonly supported protocols, and a directory SHOULD
decline to list protocols that it does not understand or cannot validate.

The DNS TTL for SVCB service records served by a directory domain SHOULD be limited
to a reasonable value determined by the directory implementation.  Directories SHOULD
consider the expected frequence of FQDN, protocol, and/or port number updates that a
service may need; for example, a service running on a residential host behind CGNAT may
need to update its inbound port number(s) more frequently than a traditionally-hosted
service.  But TTLs should also be set long enough to survive brief, transient outages.
Directories SHOULD set a TTL short enough that cached records will expire at most within
one or two intervals between the directory's internal reviews or periodic re-scans.

Directory Domains SHOULD NOT offer `ipv4hint` and/or `ipv6hint` values within their
published SVCB records as there may be security and usability implications to end-users
if a directory publishes and a client relies upon such hints.  (Consider that the SSH
protocol strongly ties host keys to FQDNs and/or IP addresses and will complain *loudly*
if something changes in an unexpected manner.)

Clients MAY use IP address hints from a SVCB record to avoid additional `A` or `AAAA`
lookups when provided, but clients SHOULD consider the same user-facing security and
usability impact described above, and clients MUST NOT require hints to be present.


### Directory Service Information records

_Service information_ records are hosted by a directory domain to provide _supplemental
information_ about a service.  These records are SPOD TXT records located at the same
FQDN as the primary _Service record_ (see prior section).  Unlike most other SPOD TXT
records, information records MUST NOT include the `d` key since the directory domain
has already been discovered at the point that such records are queried.

A _service information_ record may contain one or more of the following keys and
associated data.  If the key-value pairs are too large for a single SPOD TXT record,
a directory MAY use multiple SPOD TXT records but it MUST NOT repeat the same key in
more than one record.  Clients MUST accept zero or more _service information_ records
and treat them as if they were a combined record consisting of the union of the key-value
pairs found.

Informational keys:

- `a` (minimum age)  
  If defined, the value MUST be a decimal integer number between 0 and 100, inclusive.
  This represents the _minimum age_ expected or desired of participants connecting
  to or using the service.  The special value zero(`0`) designates that all ages are
  welcome.  The _absence_ of this key means that the age requirement is _unknown_.
  Clients SHOULD refuse to connect to a service when the end-user's age is known and is
  non-compliant.  Services MUST NOT rely on directories and/or clients for enforcement.
  Directories MAY provide this data as a suggestion or for purely informational purposes
  and its presence SHOULD NOT be interpreted under any particular regulatory statute(s).

- `i` (software info/implementation)  
  A short name or description of the software that implements the service.  For Talkers,
  examples are "NUTS", "TalkerOS", "EW-Too", etc.  The value MUST be a quoted value and
  MAY consist of multiple words and/or international characters represented as UTF-8;
  values MUST NOT include double-quote(`"`) and SHOULD NOT include semicolon(`;`).

- `k` (keywords)  
  Directories MAY provide a comma-separated list of keywords applicable to the
  service.  The set of allowed keywords is not defined by this specification.

- `l` (location)  
  A short name or description of the geographic area where the service is hosted,
  if relevant.  The location value MUST be a quoted value and MAY consist of multiple
  words and/or international characters represented as UTF-8; values MUST NOT include
  double-quote(`"`) and SHOULD NOT include semicolon(`;`).

- `p` (service login HTTPS URL path)
  The path value MUST be a quoted value encoded to be URL-safe, and the path MUST be
  absolute (begins with `/`) and point to the service entry point for new connections.
  The path value MUST NOT include double-quote(`"`) and SHOULD NOT include semicolon(`;`).
  It SHALL be used with any HTTPS-compatible `alpn` protocol(s) (`h2`, etc.) and used
  only with the host authority resolved from the service's SVCB target value.  A path
  value MAY contain a URL query component (`?` after all path components followed by
  one or more URL key-value pairs) and/or a URL fragment (`#` after all path and any
  query components, followed by one or more characters).  The entire path value MAY
  contain up to ONE(1) occurrence of the (non-encoded) literal character sequence `%u`
  as a placeholder to hint the user's login name (handle).  Before using the path value,
  clients MUST look for the presence of the `%u` placeholder and, if found, substitute
  the URL-safe encoding of the user's login name (if known) or else remove the
  placeholder by substituting it with an empty string.

- `s` (subject, topic, or theme)  
  A short sentence or phrase describing the theme, topic, or focus of the service.
  The subject value MUST be a quoted value and MAY consist of multiple words and/or
  international characters represented as UTF-8; values MUST NOT include
  double-quote(`"`) and SHOULD NOT include semicolon(`;`).

- `t` (descriptive title)  
  Gives a human-readable display name for the service, which MAY
  The title value MUST be a quoted value and MAY consist of multiple words and/or
  international characters represented as UTF-8; values MUST NOT include
  double-quote(`"`) and SHOULD NOT include semicolon(`;`).

- `w` (website)  
  Specifies a Web address that provides more information about the service.  Directories
  MAY point to a service-specific page hosted by the directory itself _or_ point to an
  arbitrary URL provided by the service owner, such as the service's own Web page.
  The website value MUST be a quoted value, MUST NOT include double-quote(`"`), SHOULD NOT
  include semicolon(;), and MUST be a well-formed URL including a _scheme_, host
  _authority_, optional _port_, and absolute _path_ (which MAY be the root path `/`).
  The value MUST use a _scheme_ that uses a secure protocol, such as `https`.  This URL is
  for informational purposes only and clients MUST NOT use this value when attempting to
  establish a connection to the service itself (see: `p` above).

> IMPORTANT:  
> A directory MUST publish _service information_ record(s) when one of the service's
> SVCB records specifies an `alpn` protocol name that includes `https`, `h2`, `h3`,
> or any future HTTPS-compatible protocol.  In that scenario, at least one of the
> SPOD TXT records MUST include the `p` (login URL path) key with an appropriate value.
>
> Clients that select an `alpn` protocol name matching the above conditions MUST query
> the directory domain for _service information_ record(s) to discover the appropriate
> path to use when constructing the connection URL.  Otherwise, a client MAY ignore
> _service information_ records.

Directories MUST NOT publish _service information_ records containing any key-value
pairs not listed above in this section.  Clients MUST accept and ignore key-value
pairs they do not recognize.


### Directory Service List records

The above record types allow a client to discover all the information it needs about
a service once it has identified the directory domain and the service's name (moniker)
within the directory.  _Directory service list_ records allow for the discovery of
previously unknown services through enumeration once a directory has been located.

This is an optional specification feature.  A directory MAY choose to exclude this
record type.  Clients SHOULD NOT consider it an error if a directory does not provide
_directory service list_ records.

A _directory service list_ is made up of one or more SPOD TXT record(s) located at
the special domain prefix `_svc`.  For example, the `_spod.example.com` directory
will publish its _directory service list_ at the FQDN `_svc._spod.example.com`.
These SPOD TXT record(s) normally only contain a single `n` (name) key containing the
comma-separated service names (monikers) of those services indexed by the directory.
Since this record exists within the directory domain itself, no `d` key is required
and the `d` key MUST be absent.

A directory MAY present its service list using multiple SPOD TXT record(s), each
containing a subset of the list under the `n` key.  Clients MUST accept multiple
records and SHOULD treat them as if they were a single record containing the union
of all the names found under the `n` keys of any of the records.

A very large directory MAY choose to split the names across additional FQDNs due
to DNS packet size limitations.  If so, at least one of the SPOD TXT records at
the special `_svc` prefix MUST include the key-value `next=_xxx` where `_xxx` is
replaced by another prefix of the directory's choosing.  The directory SHOULD choose
a prefix that would not conflict or be confused with an actual service entry; for
example, `_svc2`.  Clients that encounter the `next` key SHOULD query also for
SPOD TXT records under the designated prefix and process them as if they were
originally resolved under the `_svc` special prefix; the record format MUST match
the requirements of this section.  A directory using a `next` prefix MAY also use
the `next` mechanism for additional values, if necessary.  Clients SHOULD continue
to follow the `next` sequences to build a complete list of the service names;
however, clients also SHOULD implement safeguards to prevent following too many
`next` sequences.


#### Service Address records

The SVCB _directory service records_ point to the FQDN(s) of the host that provides
the actual service.  These FQDNs MAY be within the same domain root as the SPOD
directory, but it is expected in most cases that the service's host domain and the
directory domains will be separate.

Unless the SPOD directory domain offers additional features outside of this
specification, such as _dynamic DNS_, the listed services are expected to have
their respective address records(s) already available and published in the
public DNS system.

Each service MUST have a FQDN that resolves to one or more `A` and/or `AAAA`
records.  A service MAY use a `CNAME` record as its FQDN as long as the CNAME
ultimately resolves to `A` and/or `AAAA` records:

    coolchat.other.example.org.  CNAME server2.example.org.
    server2.example.org.         A     127.0.0.2
    server2.example.org.         AAAA  ::1

Services SHOULD avoid relying on long chains (multiple hops) of CNAME record
resolutions.  Clients SHOULD give up after following a reasonable number of
CNAME records without finding a publicly routable A or AAAA address record.
(Note: the address records above are _not publicly routable_ only because this
is a documentation example.)


#### Service Hint records

The above record types supply sufficient information to connect to "coolchat" if
the user has a client that knows to use the `_spod.example.com` directory domain.
But consider a user who discovered `coolchat.other.example.org` on a Web page or
some other way, and has a client that is _unaware_ of the `_spod.example.com`
directory.

The service can enable discovery of its directory information by adding a SPOD TXT
record to its own host's FQDN record(s) outside of the directory:

    server2.example.org    TXT "v=spod1; d=_spod.example.com; n=coolchat"

A client MAY resolve `coolchat.other.example.org` looking for SPOD TXT records and
SHOULD follow the CNAME to `server2.example.org`.  The SPOD TXT record there points
to the SPOD directory domain using the `d` key (as a _directory domain hint_);
additionally, it adds the key `n` (for _name_) with its value equal to the service's
moniker within the referenced directory.  This adds enough context for the client
to do a normal SPOD directory lookup for `coolchat._spod.example.com` to find
the appropriate protocol, port, and other information.

Clients that support the resolution and discovery process described in the preceding
paragraph MUST, after locating the relevant directory entry, verify that at least one
of the _directory service record(s)_ specifies a SVCB target FQDN that matches exactly
the FQDN initially requested by the end-user.  If an exact match is not found then the
client MUST behave as if the directory entry was not found; a client SHOULD NOT
proceed with the connection attempt unless it already has sufficient information (i.e.
protocol and port) provided through configuration or input directly from its user.

A FQDN, whether a CNAME or an address record, MAY point to a host that offers
multiple services on that same host.  In this use case the `n` key allows for a
comma-separated list of moniker names.  All such names MUST be resolved relative to
the same SPOD Directory Domain pointed to by the `d` key in the same SPOD TXT record.
If there are too many services to list in a single record, multiple SPOD TXT
records MAY be used and clients SHOULD union the distinct names across all such
records sharing the same `d` key value.

Services SHOULD publish a SPOD TXT record from their host FQDN to facilitate
the discovery of directory data.  Services MAY publish multiple SPOD TXT records
hinting to more than one SPOD directory domain when they are listed in more than
one SPOD directory.  When a service is listed in multiple directories, its
service moniker MAY be different in each directory or it MAY reuse the same
moniker in multiple directories (availability permitting).

Clients MUST be prepared to handle multiple SPOD TXT records on a host FQDN,
for the same or possibly different `d` values, and MUST be prepared to handle
SPOD TXT records whose `n`-key values specify more than one service moniker.

In the presence of multiple service names in the same service hint record, or
multiple directory domains, the client SHOULD, upon being told to connect to
a plain host FQDN, present the user with the available directory-and-service
name pairs and provide a means to indicate which service entry is wanted (if any).



## Security considerations

### DNSSEC

This specification creates an indirection that relies upon information in one
DNS domain (belonging to the directory) to influence the behavior of clients
acting on behalf of end-users when attempting to connect to services hosted
on other domains (which may or may not belong to the service's operator).

A DNS response that is compromised by an attacker-in-the-middle (AitM) prior
to receipt by the end-user's client may result in adverse behavior by the
client without being noticed by the end-user.  For example, the attacker could
alter a SVCB response to direct the client to connect to an insecure protocol
such as Telnet on a FQDN that resolves to an IP address and port under the
attacker's control.  A sufficiently similar facsimile to the legitimate service
could then capture and replace login credentials, or proxy and record an entire
interactive session.

DNSSEC is the general solution to this problem for the service's domain, but
implementing DNSSEC for the service's host(s) records does not protect the
information received through the directory's indirection.  Therefore, DNSSEC
record signing SHOULD be implemented for any DNS zone serving a canonical
_directory domain_.

Clients SHOULD use validating DNS resolvers and/or implement DNSSEC signature
validation within the client.  DNS records with an invalid signature MUST be
rejected.  If directory records are not signed at all, clients SHOULD warn
end-users of the security implications of trusting such records from an
unsigned directory.  Clients MAY use such records only after positive user
consent and MAY persist a record of that consent for future resolutions. If
a client persists consent, the consent SHALL be removed upon detection of
the subject directory later publishing DNSSEC-signed records such that any
reversion to an unsigned state causes the user to be alerted and prompted
again for consent.

Implementing DNSSEC as described in this section will protect the integrity
of directory data in transit between the authoritative zone server(s) hosting
the directory and end-user clients.  However, DNSSEC does not attest to the
accuracy or trustworthiness of the records relative to anywhere else on the
Internet outside of the signed zone itself.  For example, a DNSSEC-signed
zone may publish `A`, `AAAA`, `CNAME`, or other records that point to domains
and IP addresses that do not belong to, and may have no legitimate
relationship with, the zone's owner/operator.

This specification relies on that ability and does not protect against it.


### Rogue or Malicious Directory entries

A directory entry could be submitted or managed by an entity other than
the rightful operator of a service.  This case may be completely benign,
in the case of a helpful user wanting a favorite service to be listed in
a directory used by that user, while the service's operator may not be
aware of that directory or SPOD directories in general.  Such a directory
entry is said to be "rogue" because it is not managed by the service target's
legitimate owner/operator.

Depending on how submissions are accepted and vetted by a directory instance,
if at all, the same actions could be used by a malicious entity to point to
services or infrastructure that do not wish to be listed and do not desire
receiving DNS queries to their DNS zones from well-intentioned but misled
clients browsing the directory.  Such directory entries are said to be
"malicious" because they intend to deceive or cause unwanted side effects.

However, nothing in the way this specification uses DNS introduces any new
capability that a malicious actor could not do with any DNS zone they could
create for themselves.  The only new component is the presumed existence
of clients that, when following this specification, will make such queries
while attempting to perform what they perceive to be legitimate actions.

This specification attempts to minimize the number of DNS requests that
are issued toward a listed service's DNS zones to reduce the chance of
unintended attacks such as DNS amplification.  Most of the DNS queries
described in the specification resolve against the directory's DNS zone.
A directory operator SHOULD consider these concerns and the potential DNS
traffic load before opting to host a public directory on the Internet.

The DNS queries required for resolving CNAMEs (if any) and A/AAAA records
to determine the service's host IP address prior to attempting a connection
are the same DNS queries that would be required by any client attempting
to connect directly (without the aid of any directory described by this
specification), so this specification does not consider these queries to
be any additional risk that publicly reachable services have not already
potentially allowed already.

Similarly a service that is already publicly reachable on the Internet
should not be surprised if its FQDN(s), IP(s), protocol(s), and port(s)
are somehow published -- whether in print, on a Web page or, in this case,
in a particular format within DNS.  This specification makes those details
more discoverable, but a service relying on their secrecy is in fact
practicing _security by obscurity_ and will be commonly defeated by other
threats such as IPv4 address space and port scans with server fingerprinting.


### Loss of Control of a Service's Directory entry

This specification does not prescribe how a directory implementation
ingests new directory entries nor how existing entries are managed and
updated.  However, directory operators SHOULD account for how their
practices may impact end-users that may use their directory and the
service operators that want to be listed within the directory.

A service owner that experiences a loss of credentials to update their
directory entry may be inconvenienced at best, or the directory entry
could become inaccessible to updates and become unusable should the
target service change its host FQDN, protocols, ports, etc.

The credential types offered to authenticate persons authorized to update
a directory entry is also a concern.  Using phishable credential types,
and particularly without the use of robust MFA, may put record maintainers
at risk of account takeover (ATO).  A malicious actor that achieves ATO
can deface a directory entry or, worse, change it so that the service's
trusted name (moniker) causes end-users to attempt to connect to rogue
instances.  (See concerns above in the section regarding DNSSEC, because
in this scenario ATO bypasses any protections afforded by DNSSEC.)

A somewhat related threat is also service name "squatting" where an
entity reserves in advance certain monikers within a directory to act as
a denial of service against the legitimate listing of real services that
may commonly be known by (and want to be listed under) the taken names.


### Directory enumeration

This specification specifically allows for and enables the enumeration of
directory domain records, which will lead to the discovery of service
names, metadata, FQDNs, IP addresses, ports, and protocol information of
listed services.

The point of a directory is to make this information available and it is
a non-goal to prevent enumeration or disclosure to unknown third-parties.
Directories servicing private infrastructure SHOULD publish directory
records to trusted DNS servers that control or limit access to authorized
users or network orgins -- through the use of firewalls or access control
lists (ACLs).


### User privacy and tracking

This specification assumes that any user tracking or privacy concerns
that would be present without a directory are still present when using
a directory to help facilitate the discovery of or connection to those
services.  For instance, if an end-user's DNS queries are logged or if
their outbound network activity is monitored using a firewall, their
connection attempts to the target services' infrastructure will be
observed when resolving a FQDN or making a TCP or UDP connection to
a particular IP address or port.  This specification does not attempt
to mitigate this privacy risk.

This specification acknowledges that it may make determining the purpose
of such communication attempts easier to discover, since a directory may
tie FQDNs/IPs and port/protocol combinations to metadata that provides
or implies the nature of the target service.

Additionally, there is added risk to end-users because a new DNS zone
recieves additional queries that would not have been issued if not for
this specification (and a client that operates using it).  That means
that the operator of the directory's DNS zone, or other relays or caching
resolves between the end user and the zone's authority, may observe and
associate queries to specific clients.
