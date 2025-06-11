# Setup Milvus Backup Demo

## Dependencies


```bash
brew install kind
brew install zilliztech/tap/milvus-backup
```

## Create a kind cluster

```bash
kind create cluster --name milvus-backup-demo --config kind-config.yaml
export KUBECONFIG="$(kind get kubeconfig-path --name="milvus-backup-demo")" 
```

## Install Milvus-Operator

```bash
helm repo add milvus https://milvus-io.github.io/milvus-helm
helm repo update
helm install milvus-operator milvus/milvus-operator --namespace milvus-operator --create-namespace
```

## Install Milvus

```bash
kubectl apply -f milvus.yaml
```

Wait for Milvus to be ready then port-forward the service:

```bash
kubectl port-forward svc/milvus-standalone 19530:19530
kubectl port-forward svc/milvus-standalone 9091:9091

```

## Create collection and populate with data

```bash
python -m venv .env/
source .env/bin/activate
pip install -r requirements.txt
python populate.py
```

## Create backup

```bash
milvus-backup create -c demo_collection -n demo-backup
```


## Restore backup

To restore the backup we should delete etcd storage and scale milvus to 0. We need to keep minio, as the backup is stored there.

```bash
kubectl delete pvc data-my-release-etcd-0
kubectl scale statefulset my-release-etcd --replicas=0
kubectl scale deployment milvus-operator -n milvus-operator --replicas=0
kubectl scale deployment my-release-milvus-standalone --replicas=0
```

Wait for the pods to be deleted. Then scale back up:

```bash
kubectl scale statefulset my-release-etcd --replicas=1
kubectl scale deployment milvus-operator -n milvus-operator --replicas=1
kubectl scale deployment my-release-milvus-standalone --replicas=1
```

Then we can restore the backup:

```bash
milvus-backup restore -n demo-backup
```

## Verify the Restore

Using [Attu](https://github.com/zilliztech/attu?tab=readme-ov-file#install-desktop-application), verify the collection exists. The collection needs to be loaded before we can query it.


> **_NOTE_**: A vector index is not created by default. You can create it in Attu.
