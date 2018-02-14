# Structured service addressing
The purpose of this document is to explore the use of client side loadbalancing in new ways using IPv6.

"Structured Service Addressing" using IPv6 unicast addresses to enable controlled client side load sharing and content/direct
addressable storage in a “client – cluster communication paradigm”.

# Problem statement
Today we have a proliferation of scaled out clustered services hosted in lage datacenters. The traditional client-server communication pattern with a very simple (naïve?) client
implementation and small number of VIP's to represent the service, causes a need for state heavy or synchronized coordination mechanisms on the "server/cluster side". Such might
include sticky load balancers, monitoring nodes with cluster topology maps and load balancers with consistent hashing tables to map client-access to a
clustered resource. This approach imposes limitations to scalability and efficiency by requiring a lot of cluster-side compute and memory resources to
maintain, and in some cases creates aggregation points in the infrastructure which eventually becomes bottlenecks. The current approach also discourages the use
of statefull front-end services in modern scale out architectures in order to enable loadbalancing without sticky connections. Statefull front-end services can however have significant efficiency and latency advantages over stateless services with
statefull backends, so this is a hidden cost of todays non-sticky loadbalancer configurations.

# Proposition
“Structured Service Addressing” uses a large number of IPv6 addresses to front a service. Actual service endpoints are mapped to one or many addresses in an address range that could be described in a DNS APL RR. Optionally a destination address mapping pattern and “hard to tamper” client state, like the network part of the clients source
address, is used to communicate with the service. The client is responsible for calculating the destination address to reach the service. Depending on the rules demanded by the service, this could be as simple as picking
destination address(es) at random to get naïve load sharing or computing the destination address based on the source address, predefined rules and the
service IP-range to allow for stateless L4 DDoS protection at the cluster side, or computing destination address based on the content the client needs form
the service to directly address a scaled content service, or a combination of these three methods.
Caveats: NAT and IP spoofing will at least complicate implementation and operation.

# Resources
https://tools.ietf.org/html/rfc5375 "IPv6 Unicast Address Assignment Considerations“

https://tools.ietf.org/html/rfc3123 "A DNS RR Type for Lists of Address Prefixes (APL RR)“

https://tools.ietf.org/html/rfc3587 "IPv6 Global Unicast Address Format"

# Use case 1: Client-cluster communication – Internal known client, naïve load sharing

![Use case 1](https://github.com/acje/structured-service-addressing/blob/master/images/usecase1.PNG)

For this example N x /128 subnets is also possible. It is important to have many more subnets (N) than service proxies to be able to scale up the number of service proxies and endpoints without adding addresses or getting imbalanced load. If you do not use /128 subnets you could also split the subnets into smaller slices to enable more scaling when needed without client visibility. Adding addresses is not desirable because scaling now becomes a client visible operation and may require DNS refresh to balance load correctly. 

Example:
Service address space is a single /64 subnet and we want to scale dynamically between 10 and 1000 instances.
Having at least 1000 address slices is needed, and to get full uniform coverage of the /64 address space that would require at least 1024 subnets of size /84, however this would lead to great imbalance as the number of service instances passes 512 and some have 2 address slices and some have only one giving them only half the traffic. Using 16386 subnets of size /78 would eliminate this imbalance all the way up to the maximum anticipated scale of 1000. This will require the perhaps extreme example at minimum scale of each service proxy handling up to 1639 subnets each. One cloud of course have a process to split and combine subnets as the service scales up and down without the clients ever noticing the topology change as long as the whole APL RR is covered at all times and the RR is not changed. It is not necessary to use a /64 for a service, but it plays nice with routing and the APL record.

# Structured Service Addressing - Operation

Client asks DNS for IP address.
DNS returns a list of subnets (APL – Address Prefix List).
Client picks destination address(es) to be used based on predefined rules. NOTE: Rules and their format has not been described.
The address will lead to one of many service endpoints and may depend on one of these example rules;
“Naïve load sharing”. The client picks a random destination address from supplied prefix list for efficient client side load sharing without any knowledge of number of endpoints, anycast, proxies or load balancers. This will not give any advantage over a service mesh in most cases, but can perhaps be used as a means to more efficiently handle extremely scaled services or more likely in combination with 2) and 3). Can also work with round robin DNS. This scheme will be vulnerable to DDoS attacks.
“Controlled load sharing by calculating destination address based on source address”. The cluster resource must enforces adherence to rules. This can be done as a scaled out stateless operation in front of the actual resource, comparing source and destination addresses using the predefined rules. Example: source address 64 MSB must match destination address 64 LSB. Computation based on source address may be used to harden against DDoS attacks such that a large group of clients may not coordinate to overwhelm a small number of service endpoints.
“Directly addressable storage service”. The client computes the destination address to the endpoint holding the content based on (parts of) the contents address. Typically this would be the object address in an object store, but schemes to support some kind of file- or block-storage should be possible.
Some combination of the above methods or other methods, perhaps based on hashing algorithms.
Other schemes might be possible perhaps in combination with a L4 protocol carrying a token and/or "proof of work" concepts.
An analog to frequency hopping could be done by requiring the client to know a combination of time and valid address. This might be useful when deploying military applications on an unrestricted network like the Internet.


# More details
(original document, not maintained)
https://github.com/acje/structured-service-addressing/blob/master/structuredServiceAddressing.pdf
