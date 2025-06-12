# Setup Milvus Backup Demo

## Dependencies


```bash
brew install kind
brew install zilliztech/tap/milvus-backup
```

## Create a kind cluster

```bash
kind create cluster --name milvus-backup-demo --config kind-config.yaml
kind get kubeconfig --name="milvus-backup-demo" > ~/.kube/milvus-backup-demo.yaml 
export KUBECONFIG=~/.kube/milvus-backup-demo.yaml
```

## Install Milvus-Operator

```bash
helm repo add milvus-operator https://zilliztech.github.io/milvus-operator/
helm repo update milvus-operator
helm -n milvus-operator upgrade --install --create-namespace milvus-operator milvus-operator/milvus-operator
```

## Install Milvus

```bash
kubectl apply -f milvus.yaml
```

Wait for Milvus to be ready then port-forward the service:

```bash
kubectl port-forward svc/my-release-milvus 19530:19530&
kubectl port-forward svc/my-release-milvus 9091:9091&
kubectl port-forward svc/my-release-minio 9000:9000&
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
milvus-backup create -c demo_collection -n demobackup
```


## Restore backup

To restore the backup we should delete etcd storage and scale milvus to 0. We need to keep minio, as the backup is stored there.

```bash
kubectl scale statefulset my-release-etcd --replicas=0
kubectl delete pvc data-my-release-etcd-0
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
milvus-backup restore -n demobackup
```

## Verify the Restore

Using [Attu](https://github.com/zilliztech/attu?tab=readme-ov-file#install-desktop-application), verify the collection exists. The collection needs to be loaded before we can query it.


> **_NOTE_**: A vector index is not created by default. You can create it in Attu.
