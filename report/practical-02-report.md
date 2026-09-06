 DSO202 Practical 2 Report

## 1. Objective

The point of this practical was to learn how persistent storage actually works in Kubernetes, and why a normal Deployment cannot be used for something like a database. I worked through static provisioning, dynamic provisioning, StorageClasses, why a Deployment fails to give each replica its own data, StatefulSets, scaling and rolling updates, and finally deployed a real PostgreSQL database on retained storage. This covers Unit I sections 1.2.5, 1.4.1, 1.4.2, 1.4.3, 1.5.3 and Unit II sections 2.1.1 to 2.1.4.

## 2. Environment

- **OS:** Windows 11 with WSL2 Ubuntu 24.04.4
- **Docker:** 29.1.3
- **kind:** v0.32.0
- **kubectl (client):** v1.36.0
- **Kubernetes (cluster, from kind):** v1.36.1
- **PostgreSQL image:** postgres:18-alpine

## 3. Procedure and Observations

### Stage 0, Prerequisites

I confirmed docker, kind and kubectl versions matched what was needed, deleted any old cluster from Practical 1, made the host folder `/tmp/dso202-p2-storage`, and checked I had enough disk space (about 951G free).

![Tool versions](../screenshot/stage0-step1-versions.png)
![Host directory created](../screenshot/stage0-step3-hostdir.png)

### Stage 1: Cluster and Namespace

I created the 3 node cluster with `kind create cluster`, and confirmed the node names came out correctly (control-plane, worker-node-1, worker-node-2) mapped to the docker container names. I applied the namespace, the quota, and the retain StorageClass. Checking `kubectl describe resourcequota` showed all the storage limits I set. I also found the local-path-provisioner pod and its config, which showed it writes to `/var/local-path-provisioner`.

![Cluster created](../screenshot/stage1-step1-cluster-create.png)


![Nodes confirmed](../screenshot/stage1-step2-nodes.png)


![Mount check](../screenshot/stage1-step3-mount-check.png)

![Namespace, quota and StorageClass applied](../screenshot/stage1-step4-apply-namespace-quota-sc.png)


![Quota described](../screenshot/stage1-step5-quota-describe.png)
![StorageClasses listed](../screenshot/stage1-step6-storageclasses.png)

![Provisioner located](../screenshot/stage1-step7-8-provisioner.png)

### Stage 2: Static Provisioning

I made a PersistentVolume by hand pointing at a folder on worker-node-1, then a claim that bound right away since `manual` is not a real StorageClass, just a matching label. I ran a pod that wrote a line into a file on the volume. Deleting the pod and remaking it kept the old lines, proving the volume holds the data, not the pod. Deleting the claim moved the PV to `Released` instead of `Available`. The file was still on the host even after I deleted the PV object completely. Recreating everything gave a file with three lines total, from three different pods.

![PV created](../screenshot/stage2-step1-pv-create.png)
![PV fields](../screenshot/stage2-step2-pv-fields.png)
![PVC bound](../screenshot/stage2-step3-pvc-bound.png)
![No manual StorageClass exists](../screenshot/stage2-step4-manual-notfound.png)
![Static writer pod running](../screenshot/stage2-step5-pod-static-writer.png)
![Ledger seen from both container and host](../screenshot/stage2-step6-ledger-both-views.png)
![Ledger with two lines](../screenshot/stage2-step7-ledger-two-lines.png)
![PV released after claim deletion](../screenshot/stage2-step8-pv-released.png)
![Data survives PV deletion](../screenshot/stage2-step9-pv-deleted-data-survives.png)
![Ledger with three lines](../screenshot/stage2-step10-ledger-three-lines.png)


### Stage 3, Dynamic Provisioning

A claim against `standard` stayed Pending, since that class uses WaitForFirstConsumer. Once a pod used the claim, the volume got created automatically, named after the claim's UID. `df -h` inside the container showed the whole node disk, not just 1Gi, so this provisioner does not enforce requested size. Resizing to 2Gi was rejected since `allowVolumeExpansion` is false. Deleting the pod and claim removed the PV and the node folder completely (after a short delay), unlike Stage 2.

![PVC pending](../screenshot/stage3-step1-pvc-pending.png)
![PVC events explain why](../screenshot/stage3-step2-pvc-events.png)
![Pod, PVC and PV bound](../screenshot/stage3-step3-pod-pvc-pv-bound.png)
![Volume directory on node](../screenshot/stage3-step4-node-directory.png)
![Disk not capped at 1Gi](../screenshot/stage3-step5-df-uncapped.png)
![Resize forbidden](../screenshot/stage3-step6-resize-forbidden.png)
![Deletion confirmed](../screenshot/stage3-step7-delete-confirmed.png)

### Stage 4, Why a Deployment Cannot Own State

A Deployment with 3 replicas sharing one claim, on purpose, to see it fail. All three pods landed on the same node, since the volume only existed there. All three wrote into the same shared log file. Deleted pods came back with brand new random names, no way to say "this is replica 1" consistently.

![All pods on one node](../screenshot/stage4-step1-all-pods-one-node.png)
![Shared log file](../screenshot/stage4-step2-shared-log.png)
![New pod names after deletion](../screenshot/stage4-step3-new-pod-names.png)
![Cleanup confirmed](../screenshot/stage4-step4-cleanup-confirmed.png)

### Stage 5, StatefulSets

A headless Service first, then a StatefulSet called webnote with 3 replicas. Pods came up in strict order, webnote-0 fully ready before webnote-1 started. Each pod got its own claim, so they spread across different nodes freely. DNS lookup for the whole set returned three addresses, one per pod, and I could reach one specific pod by its own name. A note written into webnote-0 did not appear in webnote-1, proving private volumes. Deleting webnote-1 brought back the same name, same claim, same content, only the IP changed.

![Headless Service](../screenshot/stage5-step1-headless-service.png)
![Ordered creation](../screenshot/stage5-step2-ordered-creation.png)
![One PVC per ordinal](../screenshot/stage5-step3-pvcs-per-ordinal.png)
![Pods spread across nodes](../screenshot/stage5-step4-pods-spread.png)
![nslookup returns all pods](../screenshot/stage5-step5-nslookup-all-pods.png)
![Fetch from one specific pod](../screenshot/stage5-step6-individual-pod-fetch.png)
![EndpointSlice](../screenshot/stage5-step7-endpointslice.png)
![Private volumes proven](../screenshot/stage5-step8-private-volumes.png)
![Identity survives deletion](../screenshot/stage5-step9-identity-survives.png)

### Stage 6, Scaling and Updates

Scaling up to 4 created a new claim automatically. Scaling down to 2 killed pods in reverse order and kept all 4 claims, since `whenScaled` is Retain. Scaling back to 3 brought back the same data on the same ordinal. For the rolling update, I had to edit the committed yaml directly. The first attempt did not actually save in VS Code, so `kubectl apply` ran against the old file and nothing changed even though `kubectl rollout status` said complete. I caught this by grepping the file directly. After confirming the save, `partition: 2` updated only webnote-2, leaving 0 and 1 alone. Setting partition back to 0 rolled out the rest in descending order. Deleting the whole StatefulSet kept all claims, and recreating it brought back the old data with original timestamps intact.

![Scale up creates a claim](../screenshot/stage6-step1-scale-up-pvc-count.png)
![Descending termination order](../screenshot/stage6-step2-descending-termination.png)
![Claims retained after scale down](../screenshot/stage6-step3-retained-claims.png)
![Reclaimed volume on scale up](../screenshot/stage6-step4-reclaimed-volume.png)
![Partitioned update, only one pod](../screenshot/stage6-step5-partition-update.png)
![Rollout completed](../screenshot/stage6-step6-rollout-complete.png)
![Volume data unchanged by update](../screenshot/stage6-step6b-volume-unchanged.png)
![StatefulSet deleted, claims remain](../screenshot/stage6-step7-statefulset-deleted-claims-remain.png)
![Recreated with data intact](../screenshot/stage6-step8-recreated-with-data.png)

### Stage 7, PostgreSQL

A Secret for the database credentials, decoded in one command, showing a Secret is not encryption. Both a headless Service and a normal ClusterIP Service for postgres. The StatefulSet took about 43 seconds to become fully ready first time, since it had to run initdb. PGDATA was at `/var/lib/postgresql/18/docker`, one level below the mount, a PostgreSQL 18 specific detail. Created a table, inserted 3 rows, then deleted the pod entirely. `SELECT count(*)` still returned 3 afterward, and the log showed initialization was skipped since the data folder was not empty. Both DNS names resolved correctly.

![Secret decoded](../screenshot/stage7-step1-secret-decoded.png)
![Both Services created](../screenshot/stage7-step2-both-services.png)
![Postgres running](../screenshot/stage7-step3-postgres-running.png)
![Log and storage confirmed](../screenshot/stage7-step4-log-and-storage.png)
![PGDATA location](../screenshot/stage7-step5-pgdata-location.png)
![Table created and rows inserted](../screenshot/stage7-step6-table-created.png)
![Rows survived pod deletion](../screenshot/stage7-step7-rows-survived.png)
![Recovery log](../screenshot/stage7-step8-recovery-log.png)
![Claim reused and DNS resolves](../screenshot/stage7-step9-claim-and-dns.png)

### Stage 8, Cleanup

Saved the final state and a pg_dump before deleting anything. Deleted all workloads, confirmed 6 claims remained with no pods running. Deleting all claims destroyed the 4 standard class ones completely (after a short delay), but the 2 retain class ones just went to Released, data intact. Deleted the pv-web-static object, postgres data untouched. Deleted the whole cluster, no containers left. The static host folder still had the ledger file, completely untouched, even with the cluster and every dynamic volume gone.

![All pods deleted](../screenshot/stage8-step2-all-pods-deleted.png)
![Orphaned claims remain](../screenshot/stage8-step3-orphaned-claims.png)
![Two reclaim policies diverge](../screenshot/stage8-step4-two-classes-diverge.png)
![Static PV deleted](../screenshot/stage8-step5-static-pv-deleted.png)
![Cluster deleted](../screenshot/stage8-step6-cluster-deleted.png)
![Final asymmetry: host data survives, cluster does not](../screenshot/stage8-step7-final-asymmetry.png)


## 4. Analysis

**1. Why was the Stage 3 claim Pending while the Stage 2 claim bound right away?**

The field is `volumeBindingMode` on the StorageClass. Stage 2 used `manual`, not a real StorageClass object, so no binding mode applies and Kubernetes just looks for a matching Available volume right away. Stage 3 used `standard`, a real class set to WaitForFirstConsumer, so it waits until it knows which node a pod will run on before choosing storage.

**2. Why did Stage 2 data survive deletion but Stage 3 data did not?**

The field is `reclaimPolicy`, and it lives on the StorageClass, decided by whoever created the class, not the person writing the claim. `manual` volumes are Retain (Listing 5), so deleting the claim moves the PV to Released. `standard` is Delete, so removing the claim removes the volume and the data too.

**3. Why did all Stage 4 Deployment replicas end up on one node?**

The shared claim was bound to a volume that physically exists on one node only. The scheduler has no choice but to place every pod needing that volume there. On a managed cloud cluster, other replicas would most likely fail with a multi attach error instead, since a network disk usually only attaches to one node at a time.

**4. DNS name for the second webnote replica, and what must exist for it to resolve.**

`webnote-1.webnote.dso202-practical-02.svc.cluster.local`. This needs the headless Service `webnote` (with `clusterIP: None`), the StatefulSet's `serviceName` matching that Service exactly, and the pod being Ready.

**5. What happened to the claims scaling 4 down to 2 then back to 3?**

Scaling down left the removed pods' claims sitting there unused. Scaling back up reused the old claim by name instead of making a new one. The fields controlling this are `whenScaled` and `whenDeleted` under `persistentVolumeClaimRetentionPolicy`, both defaulting to Retain.

**6. Why mount at `/var/lib/postgresql` and not the data directory directly?**

PostgreSQL 18's image keeps its data one level down, at `/var/lib/postgresql/18/docker`. Mounting directly onto the data directory could fail, since initdb refuses to initialize a folder that already has something in it from the storage driver.

**7. Two things a StatefulSet does not provide for a database.**

It does not replicate data, each replica's volume is separate and unrelated. It does not do backups either, a surviving volume is not a backup since one bad command can destroy it. Replication needs an Operator, and real backups need something like a pg_dump saved outside the cluster.

**8. Why were released PVs not Available again, and what must an admin do?**

Kubernetes will not automatically hand a volume that might hold important data to a new claim, so it stays Released until someone deals with it deliberately, in my case by deleting the PV object itself. Only then can new storage be provisioned in its place.



## 5. Reflection

Two real mistakes happened while doing this, and fixing them taught me more than if everything had just worked.

In Stage 3, I named the dynamic claim `pvc-dynamic-demo` instead of the correct `dynamic-data` from the manifest file. I caught it by rereading the source guide properly instead of relying on memory, then deleted the wrong claim and recreated it correctly.

The bigger one was in Stage 6, during the partitioned rolling update. I edited the yaml to set `partition: 2` and change the image tag, but VS Code did not actually save the file. `kubectl apply` and `kubectl rollout status` both said the change went through, but nothing had actually changed. I found this by running `grep "partition:"` directly on the file and seeing the old value was still there. After that I made a habit of grepping a file right after saving it, before applying anything.

I also hit a VS Code and WSL disconnect in Stage 2, and at one point it opened the wrong folder entirely from Windows File Explorer instead of my Linux home. Closing it and reopening with `code .` from the correct terminal fixed it.

One thing I noticed but did not dig into fully: the standard class volumes briefly stayed `Released` before fully disappearing when deleted, instead of vanishing instantly. I think the local path provisioner deletes the folder asynchronously, but I would want to check its logs to confirm.

If I did this again, I would grep or cat every file right after editing it, instead of trusting the VS Code tab looked fine.



## 6. References

- DSO202_Practical2_Guide.md, accessed 4 to 6 September 2026
- DSO202_Practical2_Manifests.md, accessed 4 to 6 September 2026
- Kubernetes official docs, `kubectl explain` output used directly from the cluster for field checks, accessed 4 to 6 September 2026