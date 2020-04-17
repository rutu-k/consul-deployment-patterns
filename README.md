# Consul Deployment patterns

While delving into service mesh or key-value store or a service discovery solution in cloud-native space, you would have definitely come across Consul. [Consul](https://www.consul.io/), developed by HashiCorp is a multi-purpose solution which primarily provides the following features:

- Service discovery and Service Mesh features with Kubernetes
- Secure communication and observability between the services
- Automate load-balancing
- Key-Value store
- Consul watches

In this blog post, we will see how the Consul can be deployed for different scenarios.

## In cluster patterns

Briefly speaking Consul Key-Value (KV) provides a simple way to store namespaced or hierarchical configurations/properties in any form including string, JSON, YAML, and HCL.

A complementary feature for KV is [watches](https://www.consul.io/docs/agent/watches.html). As the name suggests, it watches (monitor) the values of the keys stored in KV. Watches notify changes occurred in one or more keys. In addition to this, we can also watch services, nodes, events, etc. Once, a notification is received for a configuration change, a 'handler' is called for action to the respective notification. A handler can be an executable (can be used to invoke the desired action on reception of notification) or an HTTP endpoint (can be used for health checks). Depending upon this, we can differentiate Consul Deployment patterns within a Kubernetes cluster as follows:

### Consul agent as Deployment: Monitor KV

Let us consider an example in which we deploy the Consul server using a standard helm chart. Consul agent can be deployed as separate deployment with one or more replicas and there is an application that needs to watch a specific configuration in the KV.

![Consul deployment pattern](consul-dep.png)

With proper configuration of Consul watch and handler, application successfully gets notified on KV changes. Generally, watches are configured using CLI command `consul watch` or using JSON file placed in config directory of Consul agent. A typical watch configuration is as shown below:

```
{
   "server":false,
   "datacenter":"dc",
   "data_dir":"/consul/data",
   "log_level":"INFO",
   "leave_on_terminate":true,
   "watches":[
      {
         "type": "key",
         "key": "testKey1",
         "handler_type": "http",
         "http_handler_config": {
           "path":"<Consul Endpoint>",
           "method": "POST",
           "tls_skip_verify": true
         }
      },
      {
         "type":"key",
         "key":"testKey2",
         "handler_type":"script",
         "args": ["/scripts/handler.sh"]
      },
      {
         "type":"key",
         "key":"testKey2",`
         "handler_type":"script",
         "args": ["/scripts/handler2"]
      }
   ]
}
```
From the above snippet you can see that, multiple watches can be configured at a time. Moreover, you can monitor same KV and and have separate actions by providing different scripts. Script handler also execute any binary.

Now suppose, there is a rise in traffic and the application needs to be scaled. In this scenario, the newly scaled Pods may not get the KV change notification. This is due to the fact that the application is exposed to the Consul agent via the Kubernetes service. Service by default load balances between the Pods on round-robin (considering user-space modes) or random (considering iptables mode) basis. Thus, all the running Pods may not receive a notification at once. The solution to this issue is deploying the Consul agent as a DaemonSet.

![Consul app scaled](app-scaled.png)

To summarize, this pattern will be useful for low volume use cases where we just need to monitor KV changes and don't emphasize more on the actions to be executed. The deployment of Consul agents can also be scaled easily whenever required.

### Consul agent as a DaemonSet: HTTP handler

In this scenario, we will use the Consul Helm chart to deploy the Consul agent as a DaemonSet. These changes can be incorporated by `client.enabled: true` and adding Consul watch configuration in `client.extraConfig` in values.yaml of Helm chart.
With the proper configuration, you will see that every application Pod receives the Consul notification on KV changes. But still, there is a twist here. Suppose you want to take a specific action on receiving Consul notification. For this, you need to configure a handler script that will reside in the application Pod. This script in the Pod won't be accessible to the Consul agent, as DaemonSet doesn't have access to file-system inside the Pod.

![Consul DaemonSet pattern](consul-ds.png)

### Consul agent as a sidecar: Executable handler script

In this pattern, the Consul agent runs as a sidecar in each Pod. Watch is configured on this Pod to track changes of the KV and notify subsequently using the executable handler. In this case, the handler is pointed to the localhost in contrast to the DaemonSet approach where the handler is pointed to service endpoint or an IP address. Whenever the Pod terminates or goes down, a new Pod is created with the same configuration and joins the Consul cluster to tracking the same KV changes again.

![Consul sidecar pattern](consul-sidecar.png)

### Service Sync

[Service sync](https://www.consul.io/docs/platform/k8s/service-sync.html) is a feature provided by Consul wherein you can sync services running in the Kubernetes cluster with the services running in the Consul cluster. There can be one-way sync between Consul to Kubernetes or both ways. Thus, Kubernetes services can be accessed by any node that is the part Consul cluster using Consul DNS. On the contrary, syncing Consul services in Kubernetes helps them to be accessed using kube-dns, environment variables, etc., which is a native Kubernetes way. This feature will be more useful in hybrid patterns.

## Hybrid Workloads

This pattern is extremely useful in the scenarios where some services of an application lie in virtual machines (VMs) and remaining services are containerized and managed by Kubernetes. This scenario can be observed while migrating from legacy to cloud-native workloads. Here, services running in VMs can be discovered and accessed by services running in Kubernetes and vice-versa. Service discovery in a hybrid pattern is typically carried out by a feature called [`Cloud Auto-join`](https://www.consul.io/docs/agent/cloud-auto-join.html). For this, we need to provide an appropriate cloud configuration and the rest is taken care of by Consul. All major cloud providers are supported including AWS, GCP, Azure, etc. To enable the sync between Kubernetes services and services running on VMs, we need to provide `syncCatalog.enabled: true` in Helm chart values whereas service discovery flow can be decided by `syncCatalog.toConsul: true` and `syncCatalog.toK8s: true`. In addition to this, services can also be synced based on namespaces, prefix, and tags.

![Consul hybrid pattern](hybrid-pattern.png)

## Consul Connect

[Connect](https://www.consul.io/docs/connect/index.html) is a service mesh feature provided by Consul. In this pattern, Consul adds a sidecar in each application Pod. Sidecar acts as a proxy for all the inbound and outbound communication for the application containers inside each Pod. Sidecar proxy establishes mutual TLS between them as a result, entire communication within the service mesh is highly secured. Moreover, the Connect sidecar acts as a data plane while the Consul cluster acts as a Control plane. This provides fine control to allow/deny traffic between the registered services.

A consul service mesh can be implemented by adding the following annotations in deployments:
```
"consul.hashicorp.com/connect-inject": "true"
"consul.hashicorp.com/connect-service-port": "8080"
"consul.hashicorp.com/connect-service-upstreams": "upstream-svc1:port1, upstream-svc2:port2, ..."
```
`connect-inject` adds the Consul Connect sidecar in the application Pod. `connect-service-port` specifies the port for inbound communication.
`connect-service-upstreams` specifies the upstreams services that the current application service needs to be connected. Port specified is the static port opened to listen for the respective communication.

![Consul Connect](svc-mesh.png)

### References

- [Deployment Patterns for Consul in Kubernetes](https://www.youtube.com/watch?v=-aTJ9uLJXHA&t=851s)
- https://github.com/infracloudio/consul-kubernetes-patterns
- https://www.consul.io/docs/agent/watches.html
- https://www.consul.io/docs/platform/k8s/service-sync.html
- https://www.consul.io/docs/connect/index.html
- https://github.com/hashicorp/consul-helm
