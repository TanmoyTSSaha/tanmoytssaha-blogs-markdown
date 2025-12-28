# The HTTP Mindmap
Date: 2025-12-27 <br>
*Written by: Tanmoy Saha*

HTTP has evolved from HTTP/1.1 to HTTP/2, and now HTTP/3, which uses the QUIC protocol over UDP for improved performance. Today, it’s the backbone of almost everything on the internet, from browsers and APIs to streaming, cloud, and AI systems.

At the foundation, we have underlying protocols. TCP/IP for IPv4 and IPv6 traffic. Unix domain sockets for local communication. HTTP/3 running over UDP instead of TCP. These handle the actual data transport before HTTP even comes into play.

Security wraps around everything. HTTPS isn’t optional anymore. WebSockets power real-time connections. Web servers manage workloads. CDNs distribute content globally. DNS resolves everything to IPs. Proxies (forward, reverse, and API gateways) route, filter, and secure traffic in between.

Web services exchange data in different formats, REST with JSON, SOAP for enterprise systems, RPC for direct calls, and GraphQL for flexible queries. Crawlers and bots index the web, guided by robots.txt files that set the boundaries.

The network world connects everything. LANs, WANs, and protocols like FTP for file transfers, IMAP/POP3 for email, and BitTorrent for peer-to-peer communication. For observability, packet capture tools like Wireshark, tcpdump, and OpenTelemetry let developers peek under the hood to understand performance, latency, and behavior across the stack.

Over to you: HTTP has been evolving for 30+ years, what do you think the next big shift will be?