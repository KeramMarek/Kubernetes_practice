# Manifest for Nginx Deployment with resources, volumes using emptyDir and service.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: ng
  name: ng
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ng
  template:
    metadata:
      labels:
        app: ng
    spec:
      containers:
        - image: nginx:latest
          name: ng
          resources:
            requests:
              memory: "128Mi"
              cpu: "0.5"
            limits:
              memory: "128Mi"
              cpu: "1"
          volumeMounts:
            - name: storage-dir
              mountPath: /usr/share/nginx/html
      volumes:
        - name: storage-dir
          emptyDir: {} # remove {} if you are giving more parameters.
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: ng
  name: nginx-service
spec:
  ports:
    - name: http
      port: 8000
      protocol: TCP
      targetPort: 80
      nodePort: 30000
  selector:
    app: ng
  type: NodePort
```

After applying the manifest with:

```bash
kubectl apply -f manifest.yaml
```
we can get inside the container with:

```bash
kubectl get all

kubectl exec -it pods/ng-6b4867bd6f-4nsrd -- bash

```
we can test the volume by creating a file inside /usr/share/nginx/html with 

```bash
echo "Type your sentance to dislpaly on nginx webpage" > /usr/share/nginx/html/index.html
```
when you forward to port 30000 in visal studio you can access the ngix page on localhost:30000

if you want to access the page on port 8000 you have to port forward the port with 

```bash
kubectl port-forward svc/nginx-service 8000:8000
```
in type: LoadBalancer you can only access the page through service Port:
```bash
http://localhost:8000
```
# Manifest for Nginx Deployment with resources, volumes using hostPath and service.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: ng
  name: ng
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ng
  template:
    metadata:
      labels:
        app: ng
    spec:
      containers:
        - image: nginx:latest
          name: ng
          resources:
            requests:
              memory: "128Mi"
              cpu: "0.5"
            limits:
              memory: "128Mi"
              cpu: "1"
          volumeMounts:
            - name: storage-dir
              mountPath: /usr/share/nginx/html
      volumes:
        - name: storage-dir
          hostPath:
            path: /tmp/www
            type: DirectoryOrCreate
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: ng
  name: nginx-service
spec:
  ports:
    - name: http
      port: 9000
      protocol: TCP
      targetPort: 80
  selector:
    app: ng
  type: LoadBalancer
```
the volume /tmp/www that is on the ec2 machine is mounted inside the container in /usr/share/nginx/html.

# Manifest for Nginx Deployment and servise using configMap as a volume:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          volumeMounts:
            - name: nginx-config-volume
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf  # Only mount the specific file, not the whole ConfigMap
      volumes:
        - name: nginx-config-volume
          configMap:
            name: nginx-config  # Mount the nginx-config ConfigMap

```
This mounts the nginx-config ConfigMap as a file inside the container at /etc/nginx/nginx.conf. The file inside the container will have the configuration values from the ConfigMap.

Note: If you want the whole ConfigMap to be mounted as a directory, you can omit the subPath field. The files inside /etc/nginx/nginx.conf will then correspond to the keys of the ConfigMap.

# PV Persistent Volume and PVC Persistance Volume Claim

What Are PV and PVC in Kubernetes?

Persistent Volumes (PV): These represent physical storage in your cluster. PVs are resources in the cluster that are provisioned either manually or dynamically (depending on the setup).

Persistent Volume Claims (PVC): A PVC is a request for storage by a user. It abstracts the actual PV, meaning you request a certain amount of storage, and Kubernetes finds or provisions the actual PV to satisfy that request.

The key idea is that PVs provide the storage, while PVCs allow pods to request and use the storage.

How PV and PVC Work Together:

Create a Persistent Volume (PV): This defines the storage resource.

Create a Persistent Volume Claim (PVC): This requests a certain amount of storage.

Pod Uses PVC: The pod mounts the PVC as a volume, and data persists beyond the pod’s lifecycle.

Step 1: Create a Persistent Volume (PV)

First, we define a Persistent Volume (PV). The PV is where Kubernetes will look for physical storage. The example below creates a PV using a hostPath to a directory on the EC2 instance (this would be the storage path we want to expose to the 

container).
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce  # Can only be mounted as read-write by a single node
  persistentVolumeReclaimPolicy: Retain  # Retain the volume even after PVC is deleted
  storageClassName: manual  # We don't need dynamic provisioning, so we use a manual storage class
  hostPath:
    path: /tmp/www  # Path on the host system (EC2 in this case)
    type: DirectoryOrCreate  # Create the directory if it doesn't exist

```
In this example:

capacity: Defines the size of the volume (1Gi).

accessModes: Specifies how the volume can be accessed. ReadWriteOnce means the volume can only be mounted as read-write by a single node.

hostPath: Refers to the actual path on the host machine (e.g., /tmp/www on the EC2 instance).

Step 2: Create a Persistent Volume Claim (PVC)

Now, we create a Persistent Volume Claim (PVC). This PVC requests storage from the available Persistent Volumes (PVs) in the cluster.
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  accessModes:
    - ReadWriteOnce  # Same access mode as PV
  resources:
    requests:
      storage: 1Gi  # Requesting 1Gi of storage
  storageClassName: manual  # Use the same storage class as the PV

```
In this example:

resources.requests.storage: Defines how much storage we want. This should match the size of the PV that is available.

accessModes: This should also match the PV's access mode (e.g., ReadWriteOnce).

storageClassName: This ensures the PVC requests storage from the correct set of PVs. In this case, we use manual, which corresponds to the storageClassName in the PV.

Step 3: Create a Deployment Using the PVC

Now that we have a PVC, we can use it in a Pod or Deployment. Here’s how to mount the PVC into the container.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          volumeMounts:
            - name: nginx-storage
              mountPath: /usr/share/nginx/html  # Mount PVC to this path inside the container
      volumes:
        - name: nginx-storage
          persistentVolumeClaim:
            claimName: nginx-pvc  # Use the PVC we created earlier

```
In this example:

volumeMounts: This specifies where the volume is mounted inside the container (/usr/share/nginx/html).

volumes: Here, we reference the PVC (nginx-pvc) that we created earlier, and this PVC will map to the storage defined in the PV.

Step 4: Apply the Resources

To apply the PV, PVC, and Deployment YAMLs, you would run:
```bash
kubectl apply -f nginx-pv.yaml
kubectl apply -f nginx-pvc.yaml
kubectl apply -f nginx-deployment.yaml

``` 
This will:

Create the Persistent Volume (PV) on the EC2 host.

Create a Persistent Volume Claim (PVC) that requests storage from the PV.

Create the Nginx Deployment that uses the PVC to mount storage at /usr/share/nginx/html.
# Storage Class

A StorageClass in Kubernetes provides a way to dynamically provision storage. Instead of manually creating PersistentVolumes (PVs) and binding them to PersistentVolumeClaims (PVCs), a StorageClass automates this process by requesting storage from

a storage provider (like AWS EBS, GCE Persistent Disk, NFS, or Ceph).


