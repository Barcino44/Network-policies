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
- We create an environment variable to the password.
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

- We allow the egress of traffic to the backend. To do that we specify the destiny namespace and pods using the label.
- The communication will be done using the port 5678.
- It is allowed DNS policies to solve the communication to the backend using names instead of ips.  

# 2.2 Backend policies

Again, in the ``backend-policies.yaml`` file, we create a ingress and egress policies according to the good practices presented in class.

- Frontend should have access to backend.
- Backend should have access to db.

````yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-ingress-from-frontend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: frontend
      podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 5678 
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-egress-to-db
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: db
      podSelector:
        matchLabels:
          app: db
    ports:
    - protocol: TCP
      port: 5432  
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

-  We allow the ingress of traffic of the frontend. To do that, we reference the namespace and the labels corresponding to the frontend. 

**Egress policies**

- We allow the egress of traffic to the db. To do that we specify the destiny namespace and pods using the label.
- The communication will be done using the port 5432. (Postgres port)
- It is allowed DNS policies to solve the communication to the backend using names instead of ips.  

# 2.3 database policies

Finally, in the ``db-policies.yaml`` file, we create a ingress and egress policies.

- Backend should have access to db
- db should not have access to any service.

````yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-ingress-from-backend
  namespace: db
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: backend
      podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432
  egress: []   
````
**Ingress policies**

-  We allow the ingress of traffic of the backend. To do that, we reference the namespace and the labels corresponding to the backend. 

**Egress policies**

- We do not define any policy to the egress traffic because we do not want to communicate to anyone using the db.

## 3. Proving the policies.

### 3.1 Access to frontend from internet.

To do that, we are going to take advantage of the LoadBalancer service we create for the frontend and we are going to expone a tunnel to access it.

````
minikube tunnel
````
<img width="1039" height="174" alt="image" src="https://github.com/user-attachments/assets/cea1dbc8-adb8-4d5f-b086-e285d9c3d468" />

### 3.2 Access to backend from frontend.

To do that, we are going to use the pod created in the frontend namespace to access to the pod created in the backend namespace.

<img width="664" height="88" alt="image" src="https://github.com/user-attachments/assets/a2f8ad2f-d951-4cbc-b20c-d96b3c2df8d1" />

````
kubectl exec -it frontend-67ddc659fd-jk7q2 -n frontend -- bash
````

We can access to backend using the backend-service.

**Using the ip**

<img width="992" height="88" alt="image" src="https://github.com/user-attachments/assets/53442d7a-024d-49e6-aa28-507f062448bb" />

**Using dns**

<img width="1012" height="89" alt="image" src="https://github.com/user-attachments/assets/e6eb8d54-22d3-4556-894a-735ea299c1bb" />

**We access to the 80 port because the backend-svc is listening form there.**

Also, we can access using the pod ip and the port which is listening the httpd service.

<img width="995" height="91" alt="image" src="https://github.com/user-attachments/assets/419efba2-1e4d-4af2-8867-a2559f8e0fb2" />

### 3.3 Access to database from backend.

To do that, we are going to use the pod created in the backend namespace to access to the pod created in the db namespace.

<img width="670" height="87" alt="image" src="https://github.com/user-attachments/assets/f868f8d9-4ad0-410e-95ed-db612748593e" />


````
kubectl exec -it backend-7fd7778f9f-2trpj -n backend -- sh
````
<img width="984" height="215" alt="image" src="https://github.com/user-attachments/assets/073fa425-9b66-422a-abc7-fc7915535a0b" />

In this case we can see we have connection to db. **It responds empty because there are no prepare to response and http request**




