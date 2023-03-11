+++

title = "Getting Started: Issue with Persistent Volume Claims"
description = "foobar"
date = 2023-03-11

[taxonomies]
tags = ["kubernetes"]
+++

One of the first issues with the cluster after getting everything up and running was a pod being stuck in pending state due to a problem with a persistent volume claim.

<!-- more -->
```
atte@vk1 kubectl get pods -n harbor
[...]
harbor-database-0                       1/1     Running   0             80m
harbor-jobservice-87f684c88-qtkfl       0/1     Pending   0             11m
harbor-jobservice-cdfc467f4-4bspd       0/1     Pending   0             11m
```
Looking at one of the pending pods in particular, we see that there seems to be a problem with persistent volume claims:
```
atte@vk1 kubectl describe pod harbor-jobservice-87f684c88-qtkfl -n harbor
[...]
  Warning  FailedScheduling  6m13s  default-scheduler  0/1 nodes are available: 1 pod has unbound immediate PersistentVolumeClaims. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
```
We can see that 2 PVCs are pending.
```
atte@vk1 kubectl get pvc -n harbor
[...]
harbor-chartmuseum                Bound     pvc-7079da7c-0099-4f27-8079-bbb0f75c596d   5Gi        RWO            longhorn       3h14m
harbor-jobservice                 Pending                                                                                       3h14m
harbor-jobservice-scandata        Pending                                                                                       3h14m
```
Let's find out why.
```
atte@vk1 kubectl describe pvc harbor-jobservice -n harbor
Type    Reason         Age                    From                         Message
  ----    ------         ----                   ----                         -------
  Normal  FailedBinding  2m52s (x321 over 82m)  persistentvolume-controller  no persistent volumes available for this claim and no storage class is set
  ```

 The message "no persistent volumes available for this claim and no storage class is set" was odd, as the storage class was set to longhorn, and the other pvc's were working fine:

 ```yaml
 [...]
persistentVolumeClaim:
      registry:
        storageClass: longhorn
      chartmuseum:
        storageClass: longhorn
      jobservice:
        storageClass: longhorn
[...]
 ```

 After spending some time confused about this, I simply ran 
 ```bash
 kubectl delete pvc harbor-jobservice -n harbor
 kubectl delete pvc harbor-jobservice-scandata -n harbor
 ``` 
 and the problem was sorted out.
