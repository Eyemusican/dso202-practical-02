# DSO202 Practical 2 - Persistent Storage in Kubernetes

## What this is

This repo has my work for Practical 2, implementing persistent storage for a stateful application in Kubernetes. I used kind to run a 3 node cluster (1 control plane, 2 workers) on my laptop through WSL2 Ubuntu.

The practical covers static provisioning, dynamic provisioning, why a Deployment cannot own state, StatefulSets, scaling and rolling updates, and deploying a real PostgreSQL database on retained storage.

## Software and image versions used

- Docker: 29.1.3
- kind: v0.32.0
- kubectl: v1.36.0
- Kubernetes (cluster): v1.36.1
- PostgreSQL image: postgres:18-alpine
- OS: Windows with WSL2 Ubuntu 24.04.4

## Repo structure

```

dso202-practical-02/
├── README.md
├── cluster/
│ └── kind-cluster.yaml
├── manifests/
│ ├── 00-namespace.yaml
│ ├── 01-quota-and-limits.yaml
│ ├── 02-storageclass-retain.yaml
│ ├── 03-pv-static.yaml
│ ├── 04-pvc-static.yaml
│ ├── 05-pod-static-writer.yaml
│ ├── 06-pvc-dynamic.yaml
│ ├── 07-pod-dynamic-writer.yaml
│ ├── 08-deployment-shared-pvc.yaml
│ ├── 09-service-webnote.yaml
│ ├── 10-statefulset-webnote.yaml
│ ├── 11-pod-client.yaml
│ ├── 12-secret-postgres.yaml
│ ├── 13-service-postgres.yaml
│ └── 14-statefulset-postgres.yaml
├── evidence/
│ ├── final-state-all.txt
│ ├── final-state-storage.txt
│ ├── final-statefulset-webnote.yaml
│ ├── final-state-events.txt
│ └── tasktracker-dump.sql
├── screenshot/
│ └── (screenshot for every step, named stage<N>-step<N>-description.png)
└── report/
└── practical-02-report.md

```


## How to rebuild this from an empty machine

```bash
mkdir -p /tmp/dso202-p2-storage
mkdir -p cluster manifests evidence report screenshot

kind create cluster --config cluster/kind-cluster.yaml

kubectl apply -f manifests/00-namespace.yaml
kubectl config set-context --current --namespace=dso202-practical-02
kubectl apply -f manifests/01-quota-and-limits.yaml
kubectl apply -f manifests/02-storageclass-retain.yaml

kubectl apply -f manifests/03-pv-static.yaml
kubectl apply -f manifests/04-pvc-static.yaml
kubectl apply -f manifests/05-pod-static-writer.yaml

kubectl apply -f manifests/06-pvc-dynamic.yaml
kubectl apply -f manifests/07-pod-dynamic-writer.yaml

kubectl apply -f manifests/09-service-webnote.yaml
kubectl apply -f manifests/10-statefulset-webnote.yaml
kubectl apply -f manifests/11-pod-client.yaml

kubectl apply -f manifests/12-secret-postgres.yaml
kubectl apply -f manifests/13-service-postgres.yaml
kubectl apply -f manifests/14-statefulset-postgres.yaml
```

Listing 8 (`manifests/08-deployment-shared-pvc.yaml`) is the deliberately broken anti-pattern from Stage 4 and is not meant to be kept running, apply it only to reproduce that stage, then delete it again.

## Cleanup sequence

```bash
kubectl delete pvc --all
kubectl get pv
kubectl delete pv <name for each Released volume>

kind delete cluster --name dso202-p2

# only after evidence is captured
rm -rf /tmp/dso202-p2-storage
```

Deleting the cluster does not delete the claims or the released volumes on its own, they have to be removed on purpose, and the static host directory has to be removed separately too.

## Evidence

Screenshots for every step of every stage are in `screenshot/`, named like `stage2-step10-ledger-three-lines.png`. The command output files from Stage 8 cleanup are in `evidence/`: `final-state-all.txt`, `final-state-storage.txt`, `final-statefulset-webnote.yaml`, `final-state-events.txt`, and `tasktracker-dump.sql`.