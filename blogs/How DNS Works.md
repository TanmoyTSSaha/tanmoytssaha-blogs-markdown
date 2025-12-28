# How DNS Works
Date: 2025-12-27 <br>
*Written by: Tanmoy Saha*

You type a domain name and hit enter, but what actually happens before that webpage loads is more complex than most people realize. DNS is the phonebook of the internet, and every request you make triggers a chain of lookups across multiple servers.

Step 1: Someone types bytebytego. com into their browser and presses enter.

Step 2: Before doing anything, the browser looks for a cached IP address. Operating system cache gets checked too.

Step 3: Cache miss triggers a DNS query. The browser sends a query to the configured DNS resolver, usually provided by your ISP or a service like Google DNS or Cloudflare.

Step 4: Resolver checks its own cache.

Step 5-6: If the resolver doesn’t have the answer cached, it asks the root servers, “Where can I find .com?” For bytebytego. com, the root server responds with the address of the .com TLD name server.

Step 7-8: Resolver queries the .com TLD server. TLD server returns the authoritative server address.

Step 9-10: This server has the actual A/AAAA record mapping the domain to an IP address. The resolver finally gets the answer: 172. 67. 21. 11 for bytebytego. com.

Step 11-12: The IP gets cached at the resolver level for future lookups, and returned to the browser.

Step 13-14: The browser stores this for its own future use, and uses the IP to make the actual HTTP request.

Step 15: The web server returns the requested content.

All this happens in milliseconds, before your first page even starts loading.

Over to you: Which DNS tools or commands do you rely on most, dig, nslookup, or something else?