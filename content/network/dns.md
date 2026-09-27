---
title: "Network: DNS"
date: "2023-01-03"
weight: 70
---

**TL;DR** DNS converts domain names into IP addresses.

---

Let's say the Internet is out there. There are websites of different kinds waiting for you to visit. And there are so many of them. Theoretically, each website can just be identified by an IP address, a 32-bit number. But human minds are not so good at remembering such numbers. You don't remember all the phone numbers of your friends. Your phone maintains a phonebook (a contact list) so you can just look up the phone number by a certain person's name.

DNS (Domain Name System) serves as your phonebook for the Internet. It converts the names of the website (domain names) into IP addresses. One caveat with the phone book metaphor is that in reality, IP addresses of websites are much more dynamic; they can change frequently.

### DNS lookup process

So you have a domain name you want to resolve.

Although you don't need to remember every IP address for every website, there's one IP address you or your computer needs to remember—the DNS server/resolver address. The IP address of a DNS server is usually well-known, e.g. 8.8.8.8 (Google Public DNS server), 75.75.75.75 (Comcast).

- Your application asks the DNS server to resolve a domain name.
- The DNS server may have cached the result for the domain name your application asks about. In that case, it will just return it. Otherwise, it goes through a recursive DNS look-up process.
- The DNS resolver first reaches out to the **root server**. The root server knows about different domains, such as “com”.
- The root server returns the IP addresses of the **TLD (Top Level Domain)** server that's responsible for the target top-level domain. The DNS resolver will query the TLD server. This way the original requests are delegated to the TLD servers.
- The TLD server will further delegate the requests to other nameservers that own the domain name, all the way down to the **authoritative nameservers** that hold actual records.

### DNS hierarchy

**Root servers:**

- There are 13 root servers and 13 corresponding IP addresses. The number of root servers was due to historical reasons and related to the DNS response size. Each root server has an IPv4 and IPv6 address. Root server IPs can change. Root priming is used to discover the latest list but a static list must be configured first.
- There are about 1000 root server instances in the world.
  - They have the same 13 IPs but are located at different locations.
  - Anycast is used to talk to the root server.
  - The root server mirrors merely copy information from the main root servers.
- **Root zone**: A global list of TLD. TTL is 2 days.
  - The root zone is not signed in its entirety
  - Root zone file: <https://www.internic.net/domain/root.zone.gz>

**TLD server (**.com, .net, …**)**

- Responsible for top level domain.
- There are over 1500 TLDs now.

**(Intermediate?) Nameserver** (facebook.com, google.com)

- Name servers that are between TLD servers and the authoritative nameserver.

**Authoritative nameserver** (www.facebook.com, mail.google.com)

- Source of truth for certain DNS domain records.

Note: DNS servers are reached by IP addresses. They also need to broadcast their IP addresses through BGP to allow for routing.

### DNS change propagation

If you have control over your authoritative nameserver, changing the IP address of a domain name should be pretty quick. Usually, the record cached at the resolver has a short TTL (a few minutes). Once the record expires, the resolver will resolve the domain again recursively and finally reach the authoritative nameserver to get the latest record.

It will take longer to change the address of a TLD server, which usually has a longer TTL. It can be slow to get changes propagated because we're waiting for old records to expire.

Some ISPs configure their DNS servers to ignore the TTL value. You can't do anything about that.

### DNS-based load balancing

For a popular domain, DNS can resolve to different IP addresses depending on the location and achieve geo-based load balancing.

Another way to do load balancing is with anycast: even with the same IP, there could be multiple servers announcing that same IP and each server is equivalent in serving requests. DNS is not so relevant though.

### DNSSEC

Zone signing keys are used to sign groups of DNS records.

Root Key Signing Key (KSK) is used to sign the zone signing keys.

Root Key Signing Key is taken offline and guarded securely in two locations: El Segundo, CA and Culpeper, VA.

Reference:

- https://www.cloudflare.com/dns/dnssec/how-dnssec-works/
- https://www.cloudflare.com/dns/dnssec/root-signing-ceremony/

### UDP

DNS uses UDP. Compared with TCP, UDP is connectionless, lower latency, less reliable.
