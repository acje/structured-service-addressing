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
“Structured Service Addressing”; Using a large number of IPv6 addresses, mapping one or many addresses to each actual service
endpoint, the client computes the destination address or addresses needed to reach the resource based on a DNS APL RR with a large address range
describing how to reach the service. Optionally a destination address mapping pattern and “hard to tamper” client state, like the network part of its own source
address, is used to communicate with the service. The client is responsible for calculating the destination address to reach the service. Depending on the rules demanded by the service, this could be as simple as picking
destination address(es) at random to get naïve load sharing or computing the destination address based on the source address, predefined rules and the
service IP-range to allow for stateless L4 DDoS protection at the cluster side or computing destination address based on the content the client needs form
the service to directly address a scaled content service or a combination of these three methods.
Caveats: NAT and IP spoofing will at least complicate implementation and operation.

# Resources
https://tools.ietf.org/html/rfc5375 "IPv6 Unicast Address Assignment Considerations“

https://tools.ietf.org/html/rfc3123 "A DNS RR Type for Lists of Address Prefixes (APL RR)“

https://tools.ietf.org/html/rfc3587 "IPv6 Global Unicast Address Format"

# More details
https://github.com/acje/structured-service-addressing/blob/master/structuredServiceAddressing.pdf
