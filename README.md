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

1. Client asks DNS for IP address.
2. DNS returns a list of subnets (APL – Address Prefix List).
3. Client picks destination address(es) to be used based on predefined rules. NOTE: Rules and their format has not been described.
4. The address will lead to one of many service endpoints and may depend on one of these example rules;
 * “Naïve load sharing”. The client picks a random destination address from supplied prefix list for efficient client side load sharing without any knowledge of number of endpoints, anycast, proxies or load balancers. This will not give any advantage over a service mesh in most cases, but can perhaps be used as a means to more efficiently handle extremely scaled services or more likely in combination with 2) and 3). Can also work with round robin DNS. This scheme will be vulnerable to DDoS attacks.
 * “Controlled load sharing by calculating destination address based on source address”. The cluster resource must enforces adherence to rules. This can be done as a scaled out stateless operation in front of the actual resource, comparing source and destination addresses using the predefined rules. Example: source address 64 MSB must match destination address 64 LSB. Computation based on source address may be used to harden against DDoS attacks such that a large group of clients may not coordinate to overwhelm a small number of service endpoints.
 * “Directly addressable storage service”. The client computes the destination address to the endpoint holding the content based on (parts of) the contents address. Typically this would be the object address in an object store, but schemes to support some kind of file- or block-storage should be possible.
 * Some combination of the above methods or other methods, perhaps based on hashing algorithms.
 * Other schemes might be possible perhaps in combination with a L4 protocol carrying a token and/or "proof of work" concepts.
 * An analog to frequency hopping could be done by requiring the client to know a combination of time and valid address. This might be useful when deploying military applications on an unrestricted network like the Internet.

# Structured Service Addressing - Advantage

Clients may directly connect to endpoints of a massively scaled out resource without having any information about the topology of the cluster (naming abstraction). This may include information like; how many endpoints are actually in use and which IP addresses are mapped to each endpoint.

Clients may directly connect to endpoints without having to go through Stateful proxies accessing topology mapping services, as routing to endpoints is standard (IP) packet routing.

Groups of clients can be forced to share its load on to the cluster by various methods where logic is applied to generate the correct service address to pick. Non-conforming packets would typically be dropped or put on a low priority queue. Stateless access control can be done on a per packet basis without knowledge of a connection. This makes it possible to divide the traffic into many streams before doing stateful inspection or termination.

State replication for highly available endpoints may be limited to an absolute minimum for Stateful protocols or sessions by having only one or a small sett of endpoints responsible for tracking the state of each connection. Client could be aware of how to reach all relevant state replicated endpoints without any cluster topology knowledge. This could make Stateful services useful in a cloud based environment again as the responsibility for state replication is deterministically distributed.
No client side visibility into how or when the cluster side is scaled.

# Structured Service Addressing - Requirements

Client side
Client side capability to handle lists of subnets (DNS APL “address prefix lists” Resource Record).
Predefined rules for the client to pick addresses from the address space provided with APL. Not aware of any such implementation today. The need for this capability is also mentioned in the RFC for APL. In chapter 7. Applicability Statement. 
Cluster side
Many cluster side loopback interfaces for each endpoint (POD, VM) capable of handling large (IPv6) subnets to terminate TCP or other protocol requiring a Stateful endpoint. Handling many subnets per endpoint serves to hide the topology from the client while scaling the resource by allowing the resource to reallocate subnets to other endpoints. If the resource is not content addressable the subnets can be of size /128. For a content addressable resource the large subnets can be used to map a portion of a storage address space to an IP address range, allowing the client and network to cooperate to locate the endpoint(s) that is holding the content.
Network
Large address space, to “carry more information in the address fields”, IPv6 in practice.
IPv6 ('globals') [RFC3587] reachability from client to all service endpoints. NAT will at least complicate operation.
Capability to move many small subnets inside the cluster between endpoints. (L3 SDN and service mesh)
More anti-spoofing capabilities on the Internet?

Possible starting point for a roadmap
Implement some simple client and cluster side rules in a sidecar proxy to test functionality inside a datacenter for service to service connections.
Probably need a “connection upgrade” mechanism (DNS) that legacy devices will ignore. Changing the DNS behavior of end user devices will be a very slow process.

# Use case 2: Client-cluster communication – Internal known client, content addressable storage

![Use case 2](https://github.com/acje/structured-service-addressing/blob/master/images/usecase2.PNG)

Notes:
Allow client to directly connect to the instance holding an object or block with a known address by implementing a distributed addressable memory where part of the IPv6 address matches the object address such that it is possible to calculate which IP address the object is located at.

# Use case 3: Client-cluster communication – External unknown client, forced load sharing: overview
![Use case 3](https://github.com/acje/structured-service-addressing/blob/master/images/usecase3.PNG)

# Use case 3: Client-cluster communication – External unknown client, forced load sharing: routing example
![Use case 3 routing](https://github.com/acje/structured-service-addressing/blob/master/images/usecase3r.PNG)
