# Get familiar with Consul

## Overview
The following lab walks through installing and getting familiar with Consul. 

## Install Consul 
Start by installing Consul on the lab VM. 
```bash
wget https://releases.hashicorp.com/consul/1.11.1/consul_1.11.1_linux_amd64.zip
unzip consul_1.11.1_linux_amd64.zip
sudo mv consul /usr/local/bin/
```

Now that you've installed `consul` confirm it is in your `$PATH`
```bash
consul

usage: consul [--version] [--help] <command> [<args>]

Available commands are:
    agent          Runs a Consul agent
    event          Fire a new event

...
```

## Run the Consul Agent
After you install Consul you'll need to start the agent. You will run Consul in development mode, which isn't secure or scalable, but is an easy way to experiment with most of Consul's functionality without extra configuration. You will also learn how to gracefully shut down the Consul agent.

As we discussed, `consul` can run in server mode or client mode. A client is a lightweight process that registers services, runs health checks, and forwards queries to servers. A client must be running on every node in the Consul datacenter that runs services, since clients are the source of truth about service health.

Start the agent
```bash 
consul agent -dev
```

In the output, look for the following to confirm the agent is running. 
```
==> Consul agent running!
```

The logs report that the Consul agent has started and is streaming some log data. They also report that the agent is running as a server and has claimed leadership. Additionally, the local agent has been marked as a healthy member of the datacenter.

Check the membership of the Consul datacenter by running the `consul members` command in a new terminal window. The output lists the agents in the datacenter.

```
Node      Address            Status  Type    Build   Protocol  DC   Partition  Segment
consul-0  10.0.102.211:8301  alive   server  1.11.1  2         dc1  default    <all>
```

The `members` command runs against the Consul client, which gets its information via gossip protocol. The information that the client has is eventually consistent, but at any point in time its view of the world may not exactly match the state on the servers. For a strongly consistent view of the world, query the HTTP API, which forwards the request to the Consul servers.

```
curl localhost:8500/v1/catalog/nodes | jq
```

Output: 
```json
[
  {
    "ID": "b0f03856-329e-750d-0e25-86377a1533e0",
    "Node": "consul-0",
    "Address": "10.0.102.211",
    "Datacenter": "dc1",
    "TaggedAddresses": {
      "lan": "10.0.102.211",
      "lan_ipv4": "10.0.102.211",
      "wan": "10.0.102.211",
      "wan_ipv4": "10.0.102.211"
    },
    "Meta": {
      "consul-network-segment": ""
    },
    "CreateIndex": 5,
    "ModifyIndex": 12
  }
]
```


In addition to the HTTP API, you can use the DNS interface to discover the nodes. The DNS interface will send your query to the Consul servers unless you've enabled caching. To perform DNS lookups you have to point to the Consul agent's DNS server, which runs on port `8600` by default. 

In the following command replace `consul-0` with the name of your node.

```bash
dig @127.0.0.1 -p 8600 consul-0.node.consul

```

## Register a service
One of the major use cases for Consul is service discovery. Consul provides a DNS interface that downstream services can use to find the IP addresses of their upstream dependencies.

Consul knows where these services are located because each service registers with its local Consul client. Operators can register services manually, configuration management tools can register services when they are deployed, or container orchestration platforms can register services automatically via integrations.

In this section, you'll register a service and health check manually by providing Consul with a configuration file, and use Consul to discover its location using the DNS interface and HTTP API. 

### Define a service 
You can register services either by providing a service definition, which is the most common way to register services, or by making a call to the HTTP API. Here you will use a service definition.

Create a working directory for `Consul configuration files. 
```bash 
mkdir ~/consul.d
```

Write a service definition configuration file. Pretend there is a service named "web" running on port 80. Use the following command to create a file called `web.json` in the configuration directory. This file will contain the service definition: name, port, and an optional tag you can use to find the service later on.

```bash
$ echo '{
  "service": {
    "name": "web",
    "tags": [
      "rails"
    ],
    "port": 80
  }
}' > ~/consul.d/web.json
```

To apply the changes the consul agent has to be restarted with `-config-dir` specified. 
```bash 
consul agent -dev -enable-script-checks -config-dir=./consul.d
```

You'll notice in the output that Consul "synced" the web service. This means that the agent loaded the service definition from the configuration file, and has successfully registered it in the service catalog.

We do not have a web service running on our machine, so the health checks will fail. 

## Query our web service 
Once the agent adds the service to Consul's service catalog you can query it using either the DNS interface or HTTP API.

First query the web service using Consul's DNS interface. The DNS name for a service registered with Consul is `NAME.service.consul`, where `NAME` is the name you used to register the service (in this case, `web`).

```bash
dig @127.0.0.1 -p 8600 web.service.consul
```

As you can see, `dig` received a response containing the IP address the service is registered.

You can also use `SRV` record to return the entire address and port
```bash
dig @127.0.0.1 -p 8600 web.service.consul SRV
```

Finally, you can also use the DNS interface to filter services by tags. The format for tag-based service queries is `TAG.NAME.service.consul`. In the example below, you'll ask Consul for all web services with the `rails` tag. You'll get a successful response since you registered the web service with that tag.

```bash
dig @127.0.0.1 -p 8600 rails.web.service.consul
```

In addition to the DNS interface, you can also query for the service using the HTTP API.
```bash
curl http://localhost:8500/v1/catalog/service/web | jq
```

```
[
  {
    "ID": "82f64bfa-22c2-5727-0f5d-0bae376f6584",
    "Node": "consul-0",
    "Address": "127.0.0.1",
    "Datacenter": "dc1",
    "TaggedAddresses": {
      "lan": "127.0.0.1",
      "wan": "127.0.0.1"
    },
    "NodeMeta": {
      "consul-network-segment": ""
    },
    "ServiceKind": "",
    "ServiceID": "web",
    "ServiceName": "web",
    "ServiceTags": ["rails"],
    "ServiceAddress": "",
    "ServiceWeights": {
      "Passing": 1,
      "Warning": 1
    },
    "ServiceMeta": {},
    "ServicePort": 80,
    "ServiceEnableTagOverride": false,
    "ServiceProxyDestination": "",
    "ServiceProxy": {},
    "ServiceConnect": {},
    "CreateIndex": 10,
    "ModifyIndex": 10
  }
]
```

The HTTP API lists all nodes hosting a given service. When querying the API you will want to filter for only healthy service instances. 
```bash
curl 'http://localhost:8500/v1/health/service/web?passing'
```

```
[
    {
        "Node": {
            "ID": "82f64bfa-22c2-5727-0f5d-0bae376f6584",
            "Node": "consul-0",
            "Address": "127.0.0.1",
            "Datacenter": "dc1",
            "TaggedAddresses": {
                "lan": "127.0.0.1",
                "wan": "127.0.0.1"
            },
            "Meta": {
                "consul-network-segment": ""
            },
            "CreateIndex": 9,
            "ModifyIndex": 10
        },
...
        "Checks": [
            {
                "Node": "consul-0",
                "CheckID": "serfHealth",
                "Name": "Serf Health Status",
                "Status": "passing",
                "Notes": "",
                "Output": "Agent alive and reachable",
                "ServiceID": "",
                "ServiceName": "",
                "ServiceTags": [],
                "Definition": {},
                "CreateIndex": 9,
                "ModifyIndex": 9
            }
        ]
    }
]
```

## Create a health check 
Next you'll update the web service by registering a health check for it. Remember that because you never started a service on port `80` where you registered `web`, the health check you register will fail.

You can update service definitions without any downtime by changing the service definition file and sending a `SIGHUP` to the agent or running consul reload. Alternatively, you can use the HTTP API to add, remove, and modify services dynamically. In this example, you will update the registration file.

```bash
echo '{
  "service": {
    "name": "web",
    "tags": [
      "rails"
    ],
    "port": 80,
    "check": {
      "args": [
        "curl",
        "localhost"
      ],
      "interval": "10s"
    }
  }
}' > ~/consul.d/web.json
```

The `check` stanza of this service definition adds a script-based health check that tries to connect to the web service every 10 seconds via curl. Script based health checks run as the same user that started the Consul process.

If the command exits with an exit code >= 2, then the check will fail and Consul will consider the service unhealthy. An exit code of 1 will be considered as warning state.

Now reload Consul's configuration to make it aware of the new health check.
```bash
consul reload
```

Notice the following lines in Consul's logs, which indicate that the web check is critical.

```
[INFO] agent: Synced service "web"
    [DEBUG] agent: Check "service:web" in sync
    [DEBUG] agent: Node info in sync
...
    [WARN] agent: Check "service:web" is now critical
    [WARN] agent: Check "service:web" is now critical
...
```

Consul's DNS server only returns healthy results. Query DNS for the web service again. It shouldn't return any IP addresses since web's health check is failing.
```bash
dig @127.0.0.1 -p 8600 web.service.consul
```

```
; <<>> DiG 9.10.6 <<>> @127.0.0.1 -p 8600 web.service.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 28984
;; flags: qr aa rd; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web.service.consul.        IN  A

;; AUTHORITY SECTION:
```
Notice that there is no answer section in the response, because Consul has marked the web service as unhealthy.

## Congrats
