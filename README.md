# Immediate triage
First commands to run to get insight about problem root causes:
```
kk get all 
kk describe deployment.apps/fakegameserver-tournament
kk logs pod/fakegameserver-tournament-8585b6dff9-4mswv
kk describe pod/fakegameserver-tournament-8585b6dff9-4mswv
```
## Event log:
Describe pod shows events as:
```
Events: Type Reason Age From Message ---- ------ ---- ---- ------- Warning FailedScheduling 25m default-scheduler 0/1 nodes are available: 1 node(s) didn't match Pod's node affinity/selector. no new claims to deallocate, preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
```
This means there is no node with labels as in `deployment.spec.template.spec.nodeSelector`.

## Fix: add missing labels to the node:
```
kk label node kind-control-plane  node-type=tournament instance-type=kind-control-plane
```

## check pod's events again:
```
kk describe pod/fakegameserver-tournament-8585b6dff9-4mswv | tail
<truncated>

  Warning  FailedMount       84s (x11 over 7m25s)  kubelet            MountVolume.SetUp failed for volume "server-config" : configmap "tournament-config" not found
```

## Fix: Create missing configmap:
```
mkdir configmap
wget https://kubernetes.io/examples/configmap/game.properties -O configmap/game.properties
kk create configmap tournament-config --from-file=configmap/game.properties 
```

## Delete/recreate the deployment
```
kk delete -f app.yaml
kk apply -f app.yaml
```

## Now only one pod is running
```
$ kk get pods

NAME                                         READY   STATUS    RESTARTS   AGE
fakegameserver-tournament-8585b6dff9-4mswv   0/1     Pending   0          7m11s
fakegameserver-tournament-8585b6dff9-5nzs7   0/1     Pending   0          7m11s
fakegameserver-tournament-8585b6dff9-7f86h   0/1     Pending   0          7m11s
fakegameserver-tournament-8585b6dff9-f9pq6   0/1     Pending   0          7m11s
fakegameserver-tournament-8585b6dff9-fbn6z   0/1     Pending   0          7m11s
fakegameserver-tournament-8585b6dff9-qrw8d   1/1     Running   0          7m11s
fakegameserver-tournament-8585b6dff9-tb247   0/1     Pending   0          7m11s
fakegameserver-tournament-8585b6dff9-xrwt9   0/1     Pending   0          7m11s

$ kk describe pod fakegameserver-tournament-8585b6dff9-4mswv | tail
<truncated>

  Warning  FailedScheduling  2m57s  default-scheduler  0/1 nodes are available: 1 node(s) didn't have free ports for the requested pod ports. no new claims to deallocate, preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod.
```
This is because we use `hostNetrwork: true`, so only the first pod can bind to `0.0.0.0:7000`, and the rest stay pending.

# Quick fix
Drop using `hostNetwork` and expose pods via a service of type `NodePort`. See [quick-fix/app.yaml](quick-fix/app.yaml) for more details.

```
$ kk get pods,svc -o wide
NAME                                             READY   STATUS    RESTARTS   AGE   IP            NODE                 NOMINATED NODE   READINESS GATES
pod/fakegameserver-tournament-5d8f8f89f7-248f5   1/1     Running   0          72s   10.244.0.52   kind-control-plane   <none>           <none>
pod/fakegameserver-tournament-5d8f8f89f7-9pn5w   1/1     Running   0          72s   10.244.0.51   kind-control-plane   <none>           <none>
pod/fakegameserver-tournament-5d8f8f89f7-9zqhb   1/1     Running   0          72s   10.244.0.47   kind-control-plane   <none>           <none>
pod/fakegameserver-tournament-5d8f8f89f7-k2jph   1/1     Running   0          72s   10.244.0.53   kind-control-plane   <none>           <none>
pod/fakegameserver-tournament-5d8f8f89f7-lcd5c   1/1     Running   0          72s   10.244.0.48   kind-control-plane   <none>           <none>
pod/fakegameserver-tournament-5d8f8f89f7-lxzt8   1/1     Running   0          72s   10.244.0.50   kind-control-plane   <none>           <none>
pod/fakegameserver-tournament-5d8f8f89f7-tt6mh   1/1     Running   0          72s   10.244.0.49   kind-control-plane   <none>           <none>
pod/fakegameserver-tournament-5d8f8f89f7-w6tql   1/1     Running   0          72s   10.244.0.46   kind-control-plane   <none>           <none>

NAME                     TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                         AGE     SELECTOR
service/fakegameserver   NodePort    10.96.92.97   <none>        7000:30070/UDP,7000:30071/TCP   72s     app=fakegameserver,mode=tournament
service/kubernetes       ClusterIP   10.96.0.1     <none>        443/TCP                         3d19h   <none>
```

# Long term fix
Create a controller for this game server - as requested in Exercise 1 - to handle the port allocation logic.
