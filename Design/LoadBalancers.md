## Load Balancers
A load balancer distributes network or application traffic across a number of servers. Load balancers are used to increase the capacity and reliability of applications. Session management can improve the performance of your web applications. It can also provide increased scalability, security, and improved end-user experience.

## Problems Without a load balancer
- **Single point of failure** : If the server goes down or something happens to the server the whole application will be interrupted and it will become unavailable for the users for a certain period. It will create a bad experience for users which is unacceptable for service providers.

- **Overload Servers**: There will be a limitation on the number of requests that a web server can handle. If the business grows and the number of requests increases the server will be overloaded.

- **Limited Scalability**:  Without a load balancer, adding more servers to share the traffic is complicated. All requests are stuck with one server and adding new servers won’t automatically solve the load issue.

## Benefits of Load Balancer
- **Better Performance** - Distributes traffic across servers so no single server gets overloaded, reducing downtime and improving speed.
- **Scalability** - Works with auto-scaling to add more servers during high traffic and remove them when traffic is low.
- **Failure Handling** - Detects unhealthy servers and redirects traffic to healthy ones, keeping the system available.
- **Prevents Bottlenecks** - Handles sudden spikes in traffic smoothly by spreading requests evenly.
- **Efficient Resource Use** - Ensures all servers share the workload fairly.
- **Session Persistence** - Can maintain user sessions so apps that need continuous sessions (like shopping carts) work properly

## Challenges of Load Balancer
- **Single Point of Failure** - If the load balancer itself goes down, it can disrupt traffic flow unless a backup exists.
- **Cost and Complexity** - Good load balancing solutions can be expensive and require proper setup and management.
- **Configuration Issues** - Setting up correctly can be tricky, especially for complex applications.
- **Extra Overhead** - Adds a small delay since every request passes through the load balancer.
- **SSL Management** - Handling encryption (SSL termination) at the balancer can make end-to-end security more complicated.

## Types of LoadBalancers
These load balancers are categorized according to how they are set up and managed in a system. They define whether traffic distribution is handled by hardware, software, or cloud-based configurations.
- 1. **Software Load Balancers** : Software load balancers are applications or components that run on general-purpose servers. They are implemented in softwaer, making them flexible and adaptable to various environments.
    - The application chooses the first one in the list and requests data from the server.
    - If any failure occurs persistently (after a configurable number of retries) and the server becomes unavailable, it discards that server and chooses the other one from the list to continue the process. 
    - This is one of the cheapest ways to implement load balancing. 
- 2. **Hardware Load Balancers**: As the name suggests we use a physical appliance to distribute the traffic across the cluster of network servers. These load balancers are also known as Layer 4-7 Routers and these are capable of handling all kinds of HTTP, HTTPS, TCP, and UDP traffic.
    - HLBs can handle a large volume of traffic but it comes with a hefty price tag and it also has limited flexibility.
    - If any of the servers don’t produce the desired response,  it immediately stops sending the traffic to the servers. 
    - These load balancers are expensive to acquire and configure, which is the reason a lot of service providers use them only as the first entry point for user requests.
    - Later the internal software load balancers are used to redirect the data behind the infrastructure wall. 

- 3. **Virtual Load Balancers**: A virtual load balancer is a type of load balancing solution implemented as a virtual machine (VM) or software instance within a virtualized environment ,such as data centers utilizing virtualization technologies like VMware, Hyper-V, or KVM. It plays a crucial role in distributing incoming network traffic across multiple servers or resources to ensure efficient utilization of resources, improve response times, and prevent server overload.

## Types of Load balancers - Based on functions
These load balancers are classified by the way they manage and distribute network traffic. They operate at different layers (network, transport, or application) to ensure efficient request handling and high availability.

- 1. **Layer 4 (L4) / Network Load Balancer**: Based on the network variables like IP address and destination ports, Network Load balancing is the distribution of traffic at the transport level through the routing decisions. Such load balancing is TCP i.e. level 4, and does not consider any parameter at the application level like the type of content, cookie data, headers, locations, application behavior etc. Performing network addressing translations without inspecting the content of discrete packets, Network Load Balancing cares only about the network layer information and directs the traffic on this basis only.

- 2. **Application Load Balancer / Layer 7(L7)**: Ranking highest in the OSI model, Layer 7 load balancer distributes the requests based on multiple parameters at the application level. A much wider range of data is evaluated by the L7 load balancer including the HTTP headers and SSL sessions and distributes the server load based on the decision arising from a combination of several variables. This way application load balancers control the server traffic based on the individual usage and behavior.

- 3. **Gloabal Server Load Balancer /Multi-site load balancer**: With the increasing number of applications being hosted in cloud data centers, located at varied geographies, the GSLB extends the capabilities of general L4 and L7 across various data centers facilitating the efficient global load distribution, without degrading the experience for end users. In addition to the efficient traffic balancing, multi-site load balancers also help in quick recovery and seamless business operations, in case of server disaster or disaster at any data center, as other data centers at any part of the world can be used for business continuity.

## What is Multi-Cloud Load Balancing
A Multi-Location Load Balancer (MLB) is a web server load balancer that distributes incoming traffic across a pool of endpoints that reside in multiple environments/locations.
Multi-cloud load balancing refers to an advanced form of load balancing where the workload is spread out across multiple cloud environments.

There’s no longer a single public cloud deployment. And it’s now critical to monitor, audit, and distribute traffic to different destinations without any manual intervention. Cloud load balancing operates at either the Transport Layer or the Application Layer of the OSI networking model. In addition to these criteria, there are other ways to split the traffic across multiple cloud end-points, like turn-based, weighted or persistent routing, to name a few. The multi-cloud load balancer ensures that the clients get routed to the most desirable backend servers. Health monitors ensure that the traffic is only sent to healthy backend servers and cloud providers by taking the faulty server out of the load balancing pool.

Multi-cloud load balancers have many benefits over traditional on-premises hardware devices. The global nature of cloud appliances, the ease of deploying a software-based cloud load balancer, and the ability to scale and manage load in a single cloud appliance make demand scalability and flexible control possible across a wide variety of hosting solutions. In addition, it ensures redundancy because it runs in numerous geographic locations.

It is essential to monitor, audit, and distribute traffic to different geolocation end-points, with or without any manual interference. Therefore, DNS (Domain Name System) load balancing is implemented at the DNS level. DNS requests are handled dynamically by the load balancer to direct clients to the geographical servers or load balancing end-points that best fit their requirements.

This reduces the connection time to the web server, improving user experience and interactivity while also decreasing the web server’s load. In addition, there are many ways for companies to preserve business continuity, especially when there’s a sudden server failure or service disruption.

Load balancing redirects traffic to the nearest server not affected by the loss. Traffic distribution is achieved through various predefined policies, like turn-based, weighted or persistent routing, to name a few. Health monitors ensure that the traffic is only sent to healthy backend servers. They’re also known as failover controllers.

## Load Balancing Methods
- 1. **Round Robin Algorithm**:  It relies on a rotation system to sort the traffic when working with servers of equal value. The request is transferred to the first available server and then that server is placed at the bottom of the line.
- 2. **Weighted Round Robin Algorithm** : 