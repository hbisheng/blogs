---
title: "Network: CDN"
date: "2023-01-03"
weight: 80
---

**TL;DR** CDN (content delivery network) speeds up the delivery of static content by putting it as close to the users as possible.

### CDN

Videos and images consist of lots of bits and can be slow to load. To speed up the delivery of those static contents, a CDN deploys a large number of geographically distributed servers and uses the server closet to the user to serve its request.

CDN servers deployed all over the world are called **point of presence (PoP)**. They cache the content. These edge servers can also terminate TLS for you, serving as a proxy between the client and the web server.

There could be two routing strategies for CDN:

1. DNS-based routing
   - Different IPs at different geographic locations.
2. Anycast
   - Same IP but is routed to a set of equivalent servers.
   - References
     - https://datatracker.ietf.org/doc/html/rfc1546
     - https://www.rfc-editor.org/rfc/rfc7094
     - https://www.rfc-editor.org/rfc/rfc4786

### Load balancing

CDN can be considered as a way of load balancing as it distributes the load to a large number of geographically distributed servers.

#### L4 load balancing

- You know the IP and port. But the data can be encrypted, compressed, etc.
- Use NAT (Network address translation). The client doesn't know the server and the server doesn't know which client. Only the proxy knows.
- Pros:
  - Just one TCP connection. TLS is not involved.
- Cons:
  - So little information. No smart load balancing.
- **Twemproxy** is L4. Just the IP and port.

#### L7 load balancing

- Can terminate TLS.
- It can do more smart things based on the rich information.
  - HTTP /GET/pictures
  - Backend servers dedicated to pictures and backends dedicated to texts.
  - Two TCP connections.
