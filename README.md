# Mongo application using kubernetes

### Step 1: Create a secret 

So here you need to convert username and password using base64. I have used the following username and password

echo -n "username" | base64
dXNlcm5hbWU=

echo -n "password" | base64
cGFzc3dvcmQ=


```
apiVersion: v1
kind: Secret
metadata:
  name: mongo-db-secret
type: Opaque
data:
  mongo-root-username: "dXNlcm5hbWU="
  mongo-root-password: "cGFzc3dvcmQ="
```


### Step 2 : Crate the deployment for the mongo-db and the service, following is the yaml file

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-db
  labels:
    app: mongo-db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo-db
  template:
    metadata:
      labels:
        app: mongo-db
    spec:
      containers:
        - name: mongo-db
          image: mongo
          ports:
            - containerPort: 27017
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongo-db-secret
                  key: mongo-root-username

            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongo-db-secret
                  key: mongo-root-password

---

apiVersion: v1
kind: Service
metadata:
  name: mongo-service
spec:
  selector:
    app: mongo-db
  ports:
    - protocol: TCP
      port: 27017
      targetPort: 27017
```

### Step 4 : Create a config map for the db url..

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongo-configmap
data:
  mongo-db-url: mongo-service
```

### Step 5 : Create the deployment and servie for the mongo-express.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongo-configmap
data:
  mongo-db-url: mongo-service
root@ip-172-31-44-197:~/mongo-db# cat mongo-express.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-express
  labels:
    app: mongo-express
spec:
  selector:
    matchLabels:
      app: mongo-express
  template:
    metadata:
      labels:
        app: mongo-express
    spec:
      containers:
        - name: mongo-express
          image: mongo-express
          ports:
            - containerPort: 8081
          env:
            - name: ME_CONFIG_MONGODB_ADMINUSERNAME
              valueFrom:
                secretKeyRef:
                  name: mongo-db-secret
                  key: mongo-root-username
            - name: ME_CONFIG_MONGODB_ADMINPASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongo-db-secret
                  key: mongo-root-password
            - name: ME_CONFIG_MONGODB_SERVER
              valueFrom:
                configMapKeyRef:
                  key: mongo-db-url
                  name: mongo-configmap

---

apiVersion: v1
kind: Service
metadata:
  name: mongo-express-service
spec:
  selector:
    app: mongo-express
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 8081
      targetPort: 8081
      nodePort: 30000

```

Here I am using the minikube and ec2 instance ubuntu so I am not hosted any ingress rule now. So we need to do the tunneling to the local machine. 

```
root@ip-172-31-44-197:~/mongo-db# kubectl get svc
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
hello-minikube          NodePort       10.103.15.41     <none>        8080:32446/TCP   3d
kubernetes              ClusterIP      10.96.0.1        <none>        443/TCP          3d
mongo-express-service   LoadBalancer   10.105.160.133   <pending>     8081:30000/TCP   7m10s
mongo-service           ClusterIP      10.98.134.128    <none>        27017/TCP        35m
```
---
```
root@ip-172-31-44-197:~/mongo-db# minikube service mongo-express-service --url
http://192.168.49.2:30000
```

### Open another terminal in your local machine and type the following.

```
ssh -i privatekey.pem -L 8080:192.168.49.2:30000 ubuntu@65.0.96.158
```

Finally I got the following. 

![](https://i.ibb.co/RDYsSgv/image.png)

Thank you
