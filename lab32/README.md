# Consul agent

* In this lab, you will run Consul in development mode, which isn't secure or scalable, but does let you easily experiment with most of Consul's functionality without extra configuration. You will also learn how to gracefully shut down the Consul agent.

## Lab Goals:

* Run the Consul Agent 

### Builds on:
* [Install lab](../lab31)

### Time:
    * 15 min

### Step 0) Intro

* In production you would run each Consul agent in either in server or client mode. Each Consul datacenter must have at least one server, which is responsible for maintaining Consul's state. This includes information about other Consul servers and clients, what services are available for discovery, and which services are allowed to talk to which other services.

* **Warning**: HashiCorp highly discourages single-server production deployments.

* In order to make sure that Consul's state is preserved even if a server fails, you should always run either three or five servers in production. The odd number of servers (and no more than five of them) strikes a balance between performance and failure tolerance.

* Non-server agents run in client mode. A client is a lightweight process that registers services, runs health checks, and forwards queries to servers. A client must be running on every node in the Consul datacenter that runs services, since clients are the source of truth about service health.

* When you're ready to go into production you can find more guidance on production deployment of servers and clients in our Deployment Guide. For now, lets start our local agent in development mode, which is an in memory server mode with some common features enabled (despite security risks) for ease of use, and all persistence options turned off.

* **Warning** : Never run Consul in -dev mode in production.

### Step 1) Start the agent

```shell
consul agent -dev
```

* The logs report that the Consul agent has started and is streaming some log data. They also report that the agent is running as a server and has claimed leadership. Additionally, the local agent has been marked as a healthy member of the datacenter.

* **Note for OS X Users**: Consul uses your hostname as the default node name. If your hostname contains periods, DNS queries to that node will not work with Consul. To avoid this, explicitly set the name of your node with the -node flag.

### Step 2) Discover datacenter members

* Check the membership of the Consul datacenter by running the consul members command in a new terminal window. The output lists the agents in the datacenter. We'll cover ways to join Consul agents together later on, but for now there is only one member (your machine).

```shell
consul members
```

* The output displays your agent, its IP address, its health state, its role in the datacenter, and some version information. You can discover additional metadata by providing the -detailed flag.

* The members command runs against the Consul client, which gets its information via gossip protocol. The information that the client has is eventually consistent, but at any point in time its view of the world may not exactly match the state on the servers. For a strongly consistent view of the world, query the HTTP API, which forwards the request to the Consul servers.

```shell
curl localhost:8500/v1/catalog/nodes
```

* In addition to the HTTP API, you can use the DNS interface to discover the nodes. The DNS interface will send your query to the Consul servers unless you've enabled caching. To perform DNS lookups you have to point to the Consul agent's DNS server, which runs on port 8600 by default. The format of the DNS entries (such as Judiths-MBP.node.consul) will be covered in more detail later.

```shell
dig @127.0.0.1 -p 8600 Judiths-MBP.node.consul
```

### Step 4) Stop the agent

* Stop the Consul agent by using the consul leave command. This will gracefully stop the agent, causing it to leave the Consul datacenter and shut down.

```shell
consul leave
```

* When you issue the leave command, Consul notifies other members that the agent left the datacenter. When an agent leaves, its local services running on the same node and their checks are removed from the catalog and Consul doesn't try to contact that node again.

* Forcibly killing the agent process indicates to other agents in the Consul datacenter that the node failed instead of left. When a node fails, its health is marked as critical, but it is not removed from the catalog. Consul will automatically try to reconnect to a failed node, assuming that it may be unavailable because of a network partition, and that it may be coming back.

* If an agent is operating as a server, a graceful leave is important to avoid causing a potential availability outage affecting the consensus protocol. Check the Adding and Removing Servers tutorial for details on how to safely add and remove servers.

### Step 5) Congrats!

* You did it
