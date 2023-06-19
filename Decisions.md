## Overview
This README enlists all the key decisions and its descriptions made while writing the `elasticsearch.yaml` manifest. Please find them in more details below:

#### 1. Update/Upgrade Resiliency
While running any kind of updates/upgrades we want to make sure we are not under the capacity or overcapacity. `maxsurge` specifies maximum addition number of nodes than our existing pool of es nodes, this will have us prevent from excessive utilization of resource during the upgrades. `maxUnavailable` specifies minimum number of nodes that can be available at a time.

```yaml
...
updateStrategy:
    changeBudget:
      maxSurge: 1 #creates only one new node at a time.
      maxUnavailable: 1 #using 1 because we only have 2 data nodes. Can be increased if we have more data/master nodes. Low value increases stability but increases deployment time i.e we need to find the right balance.
...
```

#### 2. Pod Disruption Budget
During the upgrades, maintenance and failure of any nodes we still want to maintain the pool of the nodes such that the es cluster continues to serve the traffic. We are using `podDistributionBudget` where `minAvailable` is set to `4` and the selector matches the es nodes.
```yaml
...
podDisruptionBudget:
  spec:
    minAvailable: 4 #using 4 as we have 3 master nodes and 2 data nodes.
    selector:
      matchLabels:
        elasticsearch.k8s.elastic.co/cluster-name: "{{.metadata.name}}"
...
```

#### 3. Uniform ES nodes distribution
The config tries to ensure the es nodes are distributed uniformly across the worker nodes for better availability and resiliency using `topologyKey: kubernetes.io/hostname`.
```yaml
...
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution: #using preferred instead of required as there always won't be unique k8s node per es node and using required might make es nodes unschedulable
    - weight: 100 #give high priority for this rule
      podAffinityTerm:
        labelSelector:
          matchLabels:
            elasticsearch.k8s.elastic.co/cluster-name: "{{.metadata.name}}"
        topologyKey: kubernetes.io/hostname #this ensures distribution of es nodes based on the hostname
...
```
#### 4. ES Node Roles/Counts
For this scenario the configuration specifies `3 master node`(for minimal H/A) and `2 data nodes`(for minimal H/A). For the sake of simplicity other roles are ignored.

#### 5. Memory mapping configuration
Default values for virtual address space on Linux distributions are too low for Elasticsearch to work properly, which may result in out-of-memory exceptions. For production workloads, it is strongly recommended to increase the kernel setting vm.max_map_count to `262144`. 

#### 6. Storage Class
The storage class used in the `volumeClaimTemplates` template is `storageClassName: gp2` default storage class created when EKS cluster is created which can be changed if you have different requirements for different roles.

#### 7. JVM flags
JVM flags can be overridden using the environment variables and this can largely impact the performance of the es cluster overall. It should be different for different roles for now the basic configuration is JVM start memory: `1GB` and JVM max: `1GB`.
```shell
...
env:
  - name: ES_JAVA_OPTS
    value: "-Xms1g -Xmx1g"
...
```
#### 8. Termination Period
The termination grace period for pod is used to give enough time to gracefully exit the es node during any kinds of upgrades and maintenance tasks. Current scenario specific `150` which should be enough but can highly differ based on the node type and size of traffic/data.
```yaml
...
spec:
  terminationGracePeriodSeconds: 300
...
```
