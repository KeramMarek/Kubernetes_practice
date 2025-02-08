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
I you use in type: LoadBalancer you can only access the page through service Port:
```bash
http://localhost:9000
```

