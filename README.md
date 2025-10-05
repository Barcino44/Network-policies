# Network-policies

## Description

The objective of this work is to demonstrate the use of network-policies to build an application based in a layered security model.

## Step followed

## 1. Creation of the application architecture.

In the ``deploments.yaml`` file, it is defined the deployment and services who will be used to create the application.

### 1.1. Creation of namespaces

````yaml
ApiVersion: v1
kind: Namespace
metadata:
  name: frontend
---
apiVersion: v1
kind: Namespace
metadata:
  name: backend
---
apiVersion: v1
kind: Namespace
metadata:
  name: db
---
````
In the first part of the file, there are created the corresponding namespaces in order to modularize the application. There are different namespaces to the frontend, backend and db.

### 1.2 Creation of the deployment and service for the frontend

````yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: frontend
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
---
````

In this part was defined the mainly characteristics of the frontend.

**In the deployment:**

- We associate the deployment with the frontend namespace we create before.
- We create a label called ``frontend``. This will be use to associate network policies in the next steps.
- We decided to use the nginx image to create the deployment. 

**In the service**

- We associate the service with the frontend namespace we create before.
- We use a selector to redirect the traffic to the pods with label ``frontend``.
- We are going to use the 80 port for the communication beetween pods in the cluster and 80 will be used for our application.
- We create a service type of LoadBalancer to get allow external access.

### 1.3. Creation of the deployment and service for the backend:

````yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: http-echo
        image: hashicorp/http-echo
        args: ["-text=hello from backend"]
        ports:
        - containerPort: 5678
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: backend
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 5678
  type: ClusterIP
---
````
In this part was defined the mainly characteristics of the frontend.

**In the deployment:**

- We associate the deployment with the backend namespace we create before.
- We create a label called ``backend``. This will be use to associate network policies in the next steps.
- We decided to use the hashicorp/http-echo image to create the deployment. When we consult the backend we will get a ``hello from backend``response.

**In the service:**

- We associate the service with the backend namespace we create before.
- We use a selector to redirect the traffic to the pods with label ``backend``.
- We are going to use the 80 port for the communication beetween pods in the cluster. 5678 will be listened for our httpd service.
- We create a service type of ClusterIP to avoid allow external access.

### 1.3. Creation of the deployment and service for the database:

````yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
  namespace: db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POSTGRES_PASSWORD
          value: "password"
        ports:
        - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: db-svc
  namespace: db
spec:
  selector:
    app: db
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP

````

**In the deployment:**

- We associate the deployment with the db namespace we create before.
- We create a label called ``db``. This will be use to associate network policies in the next steps.
- We decided to use a postgres image to create the deployment.

**In the service:**

- We associate the service with the db namespace we create before.
- We use a selector to redirect the traffic to the pods with label ``db``.
- We are going to use the 5432 port for the communication beetween pods in the cluster (Postgres port).
- We create a service type of ClusterIP to avoid allow external access.

## 2. Creation of the policies

We defined different policies to avoid/allow comunication between diferent modules of our application. To do that, first we have to install a CNI complement in the cluster.

In this case, we decided to use calico.

````
minikube start --cni=calico
````
We can confirm that the complement is correctly installed if we look for its pods.

````
kubectl get pods -l k8s-app=calico-node -A
````
<img width="830" height="89" alt="image" src="https://github.com/user-attachments/assets/daa6b4c5-0ac4-464d-899c-a841a382492b" />

### 2.1 Default policies

In the ``default-deny.yaml`` file, we create a policy to deny all the trafic (minimum access principle). These rules will re-written in the following steps.

````yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: frontend
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress: []
  egress: []
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress: []
  egress: []
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: db
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress: []
  egress: []
````

# 2.2 Frontend policies

In the ``frontend-policies.yaml`` file, we create a ingress and egress policies according to the good practices presented in class.

- Internet should have access to frontend.
- Frontend should have access to backend.

````yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-allow-ingress-internet
  namespace: frontend
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 80
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-allow-egress-to-backend
  namespace: frontend
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: backend
      podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5678
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: UDP
      port: 53     
    - protocol: TCP
      port: 53
````

In this file is defined.

**Ingress policies**

-  We allow the ingress of traffic of any ip (0.0.0.0). The frontend should be accessed from out of the cluster.

**Egress policies**

- We allow the egress od traffic to the backend. To do that we specify the destiny namespace and pods using the label.
- The communication will be done using the port 5678.
- It is allowed DNS policies to solve the communication of the backend using names instead of ips.  



