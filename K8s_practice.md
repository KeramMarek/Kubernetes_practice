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

After applying the manfifest with:

```bash
kubectl apply -f manifest.yaml
```
we can test the volume by creating a file inside /usr/share/nginx/html witth 

```bash
echo "Type your sentance to dislpaly on nginx webpage" > /usr/share/nginx/html/index.html
```

