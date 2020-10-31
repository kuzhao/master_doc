## Objective:
1. Pods from different service deployments should never run on the same node.
2. Pods of the same deployment should stay on the same node preferably.
3. The above should work after node autoscaling happens.

## Pathfinding:
- Req #1 repels pods from the node running ones of a different service --> hard podAntiAffinity
- Req #2 collects pods under the same service and runs them on the same node at the best --> soft podAffinity
 We should then use requiredDuringSchedulingIgnoredDuringExecution for AntiAffinity and preferredDuringSchedulingIgnoredDuringExecution for Affinity
- AntiAffinity codes should look like below, so that a pod will NOT run on a node where pods with a different service label are running:
```
requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
              matchExpressions:
    - key: <pod service label>
                operator: NotIn
                values:
                - <labelValue>
                topologyKey: kubernetes.io/hostname
```
- Affinity codes should look like below:
```
preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: component
                  operator: In
                  values:
                  - historical
              topologyKey: kubernetes.io/hostname
```
- All good as it seems so far? Almost but here is the trick: will a pod get scheduled after the autoscaler spawns a new node?
- The test proves negative. And the autoscaler even refuses to do that because it does not think a new node will help. And that’s TRUE!
- Back to the hard AntiAffinity part, does it match a new node where nothing is running?   It does, because no pod service label exists in any pods running on that node.
- Let’s add one more constraint to AntiAffinity and make it look like this, which will effectively free a new node and make it schedulable:
requiredDuringSchedulingIgnoredDuringExecution:
```
    - labelSelector:
              matchExpressions:
                - key: <pod service label>
                operator: Exists
    - key: <pod service label>
                operator: NotIn
                values:
                - <labelValue>
                topologyKey: kubernetes.io/hostname
```
- Finally, sample [manifest](https://github.com/kuzhao/playbooks/blob/master/kubelet/sample-apps/pod-isolation-per-svc.yaml)