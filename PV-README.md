# Kubernetes Persistent Volume (PV) and Persistent Volume Claim (PVC) – Hands-on Lab

## Objective

In this practical, we will learn how to:

- Create a Persistent Volume (PV).
- Create a Persistent Volume Claim (PVC).
- Bind a PVC to a PV.
- Attach persistent storage to a Kubernetes Pod.
- Store data inside a Persistent Volume.
- Verify that data survives Pod deletion and recreation.
- Understand the `Retain` reclaim policy.

## Architecture

```text
Persistent Volume (PV)
       1Gi
        |
        | Binding
        |
Persistent Volume Claim (PVC)
       500Mi
        |
        | Mount
        |
    Nginx Pod
        |
        v
  /usr/share/nginx/html
```

## Prerequisites

- Docker installed
- Minikube installed
- kubectl installed
- Basic understanding of Kubernetes Pods

## Step 1: Start Minikube

Start your local Kubernetes cluster:

```bash
minikube start
```

Verify the cluster:

```bash
kubectl get nodes
```

Check Minikube status:

```bash
minikube status
```

## Step 2: Create a Persistent Volume

Create a file named `pv.yaml`.

### pv.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
    type: DirectoryOrCreate
```

### Apply the PV

```bash
kubectl apply -f pv.yaml
```

Verify:

```bash
kubectl get pv
```

Get detailed information:

```bash
kubectl describe pv my-pv
```

### Important YAML concepts

| Field | Description |
|---|---|
| `capacity.storage` | Defines storage capacity (1 GiB). |
| `accessModes` | Defines how the volume can be mounted. |
| `ReadWriteOnce` | Allows read-write mounting from one node. |
| `persistentVolumeReclaimPolicy` | Defines what happens to storage after PVC deletion. |
| `Retain` | Retains the PV and underlying data after claim deletion. |
| `hostPath` | Uses a directory on the Kubernetes node. |
| `path` | Specifies the storage directory. |
| `DirectoryOrCreate` | Creates the directory if it does not exist. |

**Expected PV status:** `Available`

The PV is available because no PVC has claimed it yet.

## Step 3: Create a Persistent Volume Claim

A PVC is a request for storage.

Create a file named `pvc.yaml`.

### pvc.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

### Apply the PVC

```bash
kubectl apply -f pvc.yaml
```

Verify:

```bash
kubectl get pvc
```

Check the PV:

```bash
kubectl get pv
```

Get detailed information:

```bash
kubectl describe pvc my-pvc
```

**Expected PVC status:** `Bound`

Kubernetes matches the 500 MiB request with the available 1 GiB PV because the capacity and access mode are compatible.

The PVC is bound to `my-pv`.

## Step 4: Create a Pod that uses the PVC

Create a file named `pod.yaml`.

### pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: storage-pod
spec:
  containers:
    - name: nginx
      image: nginx:latest
      volumeMounts:
        - name: my-storage
          mountPath: /usr/share/nginx/html
  volumes:
    - name: my-storage
      persistentVolumeClaim:
        claimName: my-pvc
```

### Apply the Pod

```bash
kubectl apply -f pod.yaml
```

Check the Pod:

```bash
kubectl get pods
```

Wait until the Pod status is `Running`.

### Understand the volume configuration

- `volumes` connects the Pod to the PVC named `my-pvc`.
- `volumeMounts` mounts that volume inside the container.
- `mountPath` specifies where the storage is accessible inside the container.

## Step 5: Write data into the Persistent Volume

Enter the Nginx container:

```bash
kubectl exec -it storage-pod -- bash
```

Create a file inside the mounted directory:

```bash
echo "Hello GUVI! This data is stored in Persistent Volume." > /usr/share/nginx/html/index.html
```

Read the file:

```bash
cat /usr/share/nginx/html/index.html
```

Expected output:

```text
Hello GUVI! This data is stored in Persistent Volume.
```

Exit the container:

```bash
exit
```

## Step 6: Verify data persistence

Now we will delete the Pod and recreate it.

Delete the Pod:

```bash
kubectl delete pod storage-pod
```

Verify:

```bash
kubectl get pods
```

Recreate the Pod:

```bash
kubectl apply -f pod.yaml
```

Wait until it is running:

```bash
kubectl get pods
```

Read the previously created file:

```bash
kubectl exec storage-pod -- cat /usr/share/nginx/html/index.html
```

Expected output:

```text
Hello GUVI! This data is stored in Persistent Volume.
```

**Observation:** Even though the Pod was deleted and recreated, the data is still available because it was stored in the Persistent Volume.

## Step 7: Understand the Retain reclaim policy

Our PV uses:

```yaml
persistentVolumeReclaimPolicy: Retain
```

Let's delete the Pod and PVC:

```bash
kubectl delete pod storage-pod
kubectl delete pvc my-pvc
```

Check the PV:

```bash
kubectl get pv
```

The PV should now show:

```text
STATUS: Released
```

### Why Released?

- The PVC has been deleted.
- The PV still exists.
- The underlying storage data is retained.
- Kubernetes does not automatically make the PV available for a new claim.

This helps protect data from accidental deletion.

## Step 8: Important Kubernetes commands

| Command | Purpose |
|---|---|
| `kubectl get pv` | List Persistent Volumes |
| `kubectl get pvc` | List Persistent Volume Claims |
| `kubectl describe pv my-pv` | Get PV details |
| `kubectl describe pvc my-pvc` | Get PVC details |
| `kubectl get pods` | List Pods |
| `kubectl get pods -o wide` | Show Pod details and node placement |
| `kubectl exec -it storage-pod -- bash` | Enter the container |
| `kubectl delete pod storage-pod` | Delete the Pod |

## Project Folder Structure

```text
kubernetes-pv-pvc-demo/
│
├── pv.yaml
├── pvc.yaml
└── pod.yaml
```

## Important Notes

1. A PV represents storage available in the cluster.
2. A PVC requests storage from the cluster.
3. A Pod uses storage through a PVC.
4. Deleting a Pod does not automatically delete the PV or PVC.
5. The `Retain` policy preserves the underlying storage when the PVC is deleted.
6. `hostPath` is useful for local Minikube demonstrations but is not recommended as a general production storage solution for multi-node clusters.
7. `ReadWriteOnce` means read-write mounting from one node, not necessarily one Pod.

## Practice Questions

1. What is the difference between PV and PVC?
2. Why do we need persistent storage in Kubernetes?
3. What happens to data when a Pod is deleted?
4. What does `ReadWriteOnce` mean?
5. What is the purpose of the `Retain` reclaim policy?
6. What is the difference between `hostPath` and cloud-based persistent storage?

## Bonus Challenge

Modify the existing setup:

1. Change the PV capacity to `2Gi`.
2. Change the PVC request to `1Gi`.
3. Create a new file inside the mounted directory.
4. Delete and recreate the Pod.
5. Verify that the file still exists.

**Learning Outcome:** You have successfully created a Persistent Volume, claimed storage using a PVC, mounted it inside a Pod, and verified data persistence after Pod recreation.