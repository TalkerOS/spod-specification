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


### Directory

A _directory_ exists to index and assist the discovery of _services_ and inform
clients of the necessary details to successfully connect to such services.  A
directory that is compliant with this specification is referred to as a _SPOD directory_.

A directory is not assumed to itself run or host any of the services it lists, though
a directory operator MAY opt to do so.


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


#### Directory Domain record

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


#### Directory Domain Hint record

A SPOD TXT record where the `d` key is present but it's FQDN value _differs_ from
the record's own FQDN is considered to be a _directory domain hint_.  For example,
if the second-level domain (SLD) `example.com` wanted to advertise it hosts a SPOD
directory domain (but doesn't use the SLD itself as the root of the directory) then
it MAY host a domain hint record that "points to" the canonical directory domain:

    example.com.  TXT  "v=spod1; d=_spod.example.com"

A client that is configured to use `example.com` as its SPOD directory MAY use this
record to discover the _canonical_ directory (sub)domain.  After retrieving this
record, a client SHOULD query for a SPOD TXT record at the FQDN indicated by the
`d` key.  If no valid record is found then this is a "dead" hint and MUST be
ignored as if no such directory or hint exists.

If a valid SPOD TXT record exists, and it is a valid _directory domain record_ (see
prior section), then the _canonical_ directory domain has been found and it can be
used for subsequent operations.  If instead another _directory domain hint_ record
is found then the process of looking for a _directory domain record_ repeats using
the new `d` key value as the starting domain.  Clients MUST implement safeguards
to prevent cycles (loops) between one or more hints and SHOULD limit the maximum
number of followed hints to a reasonable number before giving up without finding
a canonical directory domain record.


#### Directory Service record

A SPOD directory domain contains zero or more records for individual services.
Each service within a directory SHOULD be identified by a single DNS name component
(no `.` separators) and follow the other conventions and restrictions regarding DNS
host names.  A service name component MUST NOT begin with an underscore(`_`).  Each
service's FQDN within the SPOD directory domain is the concatenation of its service
name component -- or moniker -- as a prefix of the directory's FQDN.

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
in this example, indicates that SSH is preferred because it has the lower value.

This directory specification is protocol agnostic.  Compliant directories MUST include
the `alpn` value (application layer protocol name) in SVCB records; there is no default or
implied protocol.  Directories MUST include a `port` value in SVCB records when a service
is offered on a port that is _different than the default port_ for the specified protocol
name.  Directories MAY elect to always provide the port number information.  The transport
protocol (i.e. TCP vs UDP) SHALL be determined based on the `alpn` value; if an application
protocol supports _both_ TCP and UDP -- such as is the case with DNS -- then this
specification is not opinionated.  Clients SHOULD defer to other conventions or standards
associated with the relevant application protocols to make a choice or ordered preference.

Some protocols, such as `https` and `h2` or `h3`, can be served from the same port.
Directories MUST combine such protocols into a single SVCB record for a given port number
by setting the `alpn` value to a comma-separated list of the protocol names.  Clients
MUST accept SVCB records that list multiple protocol names; however, clients MUST ignore
SVCB records where _none_ of the protocol names given by the `alpn` value are supported
by that client.

> NOTE:  
> Directories SHOULD NOT publish SVCB records containing the (insecure) `http`
> protocol in the `alpn` field.  Use of insecure protocols SHOULD be discouraged,
> though the `telnet` protocol (also insecure) MAY be published while services
> SHOULD consider migrating to more secure alternatives.

Directories MUST NOT publish new or updated SVCB records that target FQDNs that do not
resolve to _publicly routable_ address records.  Directories MAY target FQDNs that are
CNAME records as long as such records can be resolved to one or more `A` and/or `AAAA`
record(s) that otherwise meet the requirements.  Directories SHOULD periodically
revisit their SVCB records and remove those whose target FQDNs no longer resolve to
valid addresses.

Directory Domains SHOULD NOT offer `ipv4hint` and/or `ipv6hint` values within their
published SVCB records as there may be security and usability implications to end-users
if a directory publishes and a client relies upon such hints.  (Consider that the SSH
protocol strongly ties host keys to FQDNs and/or IP addresses and will complain *loudly*
if something changes in an unexpected manner.)

Clients MAY use IP address hints from a SVCB record to avoid additional `A` or `AAAA`
lookups when provided, but clients SHOULD consider the same user-facing security and
usability impact described above, and clients MUST NOT require hints to be present.


#### Directory Service Information records

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
  This represents the _minimum age_ expected or required of participants connecting
  to or using the service.  The special value zero(`0`) designates that all ages are
  welcome.  The _absence_ of this key means that the age requirement is _unknown_.

- `k` (keywords)  
  Directories MAY provide a comma-separated list of keywords applicable to the
  service.  The set of allowed keywords is not defined by this specification.

- `l` (location)  
  A short name or description of the geographic area where the service is hosted,
  if relevant.  The name MAY consist of multiple words separated by spaces and
  the entire value MUST be surrounded with quotes(`"`), MUST NOT contain inner
  quotes (there is no escaping mechanism), and SHOULD NOT contain semicolon(`;`)
  characters to avoid errors with naive TXT record parsers.

- `n` (descriptive name)  
  When present on a SPOD TXT record hosted _within the directory domain_ itself, the
  `n` (name) value is the human-readable display name for the service, which MAY
  consist of multiple words separated by spaces and MAY include characters that are
  not valid as a service moniker.  Display names MUST be surrounded with quotes(`"`),
  MUST NOT contain inner quotes (there is no escaping mechanism), and SHOULD NOT
  contain semicolon(`;`) characters to avoid errors with naive TXT record parsers.

- `p` (service login URL path)  
  The value MUST be an absolute path (beginning with `/`) that points to the service
  entry point for new connections and MUST be encoded to be URL-safe.  The path SHALL
  be used with any relevant `alpn` protocol(s) and used relative to the host authority
  resolved from the service's SVCB target value.  A path value MAY contain a query
  component (`?` followed by one or more URL key-value pairs) and/or a fragment (`#`
  followed by one or more characters), but it MUST NOT contain the `;` symbol.
  The entire path value MAY contain up to ONE(1) occurence of the (non-encoded) literal
  character sequence `%u` as a placeholder to hint the user's login name (handle).
  Before using the path value, clients MUST look for the `%u` placeholder and substitute
  the URL-safe encoding of the user's login name (if known) or else remove the
  placeholder by substituting it with an empty string.

- `s` (software)  
  A short name or description of the software that provides the service.  For Talkers,
  examples are "NUTS", "TalkerOS", "EW-Too", etc.  The name MAY consist of multiple
  words separated by spaces and the entire value MUST be surrounded with quotes(`"`),
  MUST NOT contain inner quotes (there is no escaping mechanism), and SHOULD NOT
  contain semicolon(`;`) characters to avoid errors with naive TXT record parsers.

- `t` (topic or theme)  
  A short sentence or phrase describing the theme, topic, or focus of the service.
  The topic MAY consist of multiple words separated by spaces and the entire value
  MUST be surrounded with quotes(`"`), MUST NOT contain inner quotes (there is no
  escaping mechanism), and SHOULD NOT contain semicolon(`;`) characters to avoid
  errors with naive TXT record parsers.

- `w` (website)  
  Specifies a Web address that provides more information about the service.  Directories
  MAY point to a service-specific page hosted by the directory itself _or_ point to an
  arbitrary URL provided by the service owner, such as the service's own Web page.
  The value MUST be a well-formed URL including a _scheme_, host _authority_, optional
  _port_, and absolute _path_ (which MAY be the root path `/`).  The value MUST use a
  _scheme_ that uses a secure protocol, such as `https`.  This URL is for informational
  purposes only and clients MUST NOT use this value when attempting to establish a
  connection to the service itself (see: `p` above).

> IMPORTANT:  
> A directory MUST publish _service information_ record(s) when one of the service's
> SVCB records specifies an `alpn` protocol name that relies upon or could potentially
> need a URI path -- as in the case of protocols `https`, `h2`, `h3`, etc.  In that
> scenario, at least one of the SPOD TXT records MUST include the `p` (path) key with
> an appropriate value.
>
> Clients that select an `alpn` protocol name matching the above conditions MUST query
> the directory domain for _service information_ record(s) to discover the appropriate
> `p` (path) for constructing the connection URL.  Otherwise, a client MAY ignore
> _service information_ records.

Directories MUST NOT publish _service information_ records containing any key-value
pairs not listed above in this section.  Clients MUST accept and ignore key-value
pairs they do not recognize.


#### Directory Service List records

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

A FQDN, whether a CNAME or an address record, MAY point to a host that offers
multiple services on that same host.  The `n` key allows for a comma-separated
list of moniker names.  All such names MUST be resolved relative to the same
SPOD Directory Domain pointed to by the `d` key in the same SPOD TXT record.
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
