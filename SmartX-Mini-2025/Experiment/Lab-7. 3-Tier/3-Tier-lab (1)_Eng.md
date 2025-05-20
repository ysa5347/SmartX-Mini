# 3-Tier Lab - Week 1

# 0. Objective

In this lab, we will go through the process of deploying a web service based on **3 Tier Architecture** in a Kubernetes environment. The entire lab spans two weeks, and the Week 1 lab focuses on the following components:

- Deploying a PostgreSQL Database on Kubernetes
- Deploying a NestJS-based Backend on Kubernetes and connecting it to the database
- Delivering a React-based web page via NGINX, or forwarding requests to the Backend

> [!note]
>
> This Lab is conducted on a Kubernetes Cluster set up in <b>Lab#5 (Cluster Lab)</b> and consists of the following structure:
>
> ![Kubernetes Installation](img/nuc-prep.png)
>
> Each participant remotely accesses NUC1 (master node) from their PC using SSH, creates their own Kubernetes namespace, and proceeds with the lab in an isolated environment within that namespace.

# 1. Concept

## 1-1. 3-Tier Architecture

**3-Tier Architecture** refers to an architectural design method that separates a single application into three layers (physically/logically) based on functionality.

It is widely used especially in systems like web services that interact with users, process data, and need to store that data.

### Components

![3-Tier Architecture](img/3-Tier-Arch.png)

1. Presentation Tier (Frontend)
   - This is the layer that directly interacts with users.
   - Representative technologies include web frameworks/libraries like React, Vue, and Angular that run in browsers.
   - Mobile apps also fall into this tier, including native (Android, SwiftUI) and cross-platform (Flutter, React Native) apps.
2. Application Tier (Backend)
   - Handles user requests from the Presentation Tier, executes business logic, and communicates with the database.
   - Typically built using frameworks like Express, NestJS, Django, and Spring.
   - Also called the Business Logic Tier or Transaction Tier.
3. Data Tier (Database)
   - Manages reading and writing to the database.
   - Permanently stores application data.
   - Databases like PostgreSQL, MySQL, and MongoDB are commonly used.

### Background

In the past, web applications were often built in **a Monolithic structure**, where a single server handled everything (UI, logic, data storage).

However, due to the following issues, the 3-Tier Architecture emerged:

- As functionality grows, project and code size become large and hard to maintain
- Server load increases as the number of users grows
- Structural limitations where changing one feature affects the entire service
- Inefficient scaling as it requires scaling the whole service

To solve these issues, separating each layer and enabling independent deployment and management became necessary. This is the 3-Tier Architecture.

### Differences from Traditional Structure

| Category        | Monolithic Architecture      | 3-Tier Architecture                |
| --------------- | ---------------------------- | ---------------------------------- |
| Code Separation | X (Functions mixed)          | O (Functions separated by layer)   |
| Maintainability | Hard to trace issues         | Manageable per layer               |
| Scalability     | Must scale entire service    | Can scale each layer independently |
| Failure Scope   | Can propagate across service | Can isolate within a single layer  |
| Deployment      | Single deployment            | Independent layer deployments      |

### Advantages

- **Independence**: Frontend, Backend, and DB can be developed and deployed independently
- **Maintainability**: Easy to identify which layer has an issue
- **Scalability**: Can scale only the high-load layer (e.g., scale BE but not DB)
- **Security**: DB is not directly exposed and only accessible via Backend

### Disadvantages

- **Operational Complexity**: More complex communication and deployment structure
- **Network Overhead**: Inter-tier traffic increases network usage
- **Higher Development Effort**: Requires interface/API/data format design between tiers

### Why Use 3-Tier Architecture on Kubernetes?

Using 3-Tier Architecture in Kubernetes provides several modern cloud-native benefits:

1. **Automated Deployment**

   - Each tier is a separate Deployment, allowing independent rolling updates

2. **Self-Healing**

   - If a Pod in any tier crashes, it is automatically restarted

3. **Horizontal Scaling**

   - High-load tiers can scale automatically to distribute traffic

4. **Unified Observability**

   - Prometheus, Grafana, and other tools allow cross-tier monitoring

5. **Infrastructure Independence**

   - Consistent deployments across cloud and on-prem environments

6. **Namespace-Based Isolation**
   - Practice or run services in isolated environments by namespace

In conclusion, 3-Tier Architecture works very well with Kubernetes and is foundational to modern service deployment and management.

## 1-2. Why Companies Use Kubernetes?

As mentioned in <b>Lab#5 (Cluster Lab)</b>, many companies are migrating to Kubernetes-based infrastructure for the following reasons:

### 1. **High Availability**

- Pods automatically recover if they crash
- Failing health checks trigger Pod restarts
- Allows partial recovery without full service downtime

### 2. **Automated Operations**

- Use of `Deployment`, `StatefulSet` for rolling updates and rollbacks
- Easy integration with CI/CD pipelines

### 3. **Maintainability**

- YAML-based declarative configuration allows version control of infrastructure
- Tools like Helm and Kustomize simplify complex configuration

### 4. **Scalability**

- Adjust replica count of `Deployment` to handle high traffic
- Use HPA (Horizontal Pod Autoscaler) for automatic scaling

### 5. **Resource Efficiency**

- Schedules workloads at the Pod level for better resource utilization
- CPU and memory limits prevent overuse

### 6. **Environment Consistency**

- Keep development, testing, and production environments aligned
- Developers' settings can be reflected in production

### 7. **Cloud & On-Prem Support**

- Works consistently on public clouds (AWS, GCP, Azure) and on-prem servers
- Enables multi-cloud strategy without vendor lock-in

As a result, Kubernetes is becoming the de facto standard for modern infrastructure—not only in enterprises, but also in startups, educational institutions, and public sectors.

# 2. Lab Preparation

Let's begin the 3-Tier Lab in earnest.

## 2-1. Remote Access to Master Node (For NUC2, NUC3)

If you're using NUC2 or NUC3, enter the following command in your terminal to remotely access the Master Node:

```bash
ssh <username>@nuc01
# After entering the command, input your password.
```

## 2-2. Checking Kubernetes Cluster Status (For all NUCs)

Once connected to the Kubernetes Master Node, enter the following commands to check the status of nodes and pods to ensure the Kubernetes cluster is running correctly:

```bash
# View the status of all nodes in the Kubernetes cluster
kubectl get nodes

# View the status of all pods across all namespaces
kubectl get pods -A
```

All nodes should be in the `Ready` state, and all pods should be in the `Running` state.

> [!warning]
>
> If any nodes are not `Ready` or any pods are not in `Running` state, the lab may not proceed properly.

## 2-3. Creating a Kubernetes Namespace (For all NUCs)

Since each student is remotely accessing the Kubernetes Master Node on their own NUC, we will use Kubernetes namespaces to logically separate resources among users.

> [!note]
>
> **What is a Namespace?**
>
> A namespace is a logical way to isolate resources within a Kubernetes cluster. It allows multiple users or teams to work simultaneously in the same cluster without interfering with each other.
>
> In this lab, each student will create **their own namespace**, and all resources must be created/managed **within that namespace only**. This allows multiple users to perform the lab concurrently without conflicts.

Run the following commands to create your own namespace:

```bash
# Check the current list of namespaces
kubectl get namespace

# Replace <your_namespace> with your machine name (e.g., nuc01, nuc02, or nuc03)
kubectl create namespace <your_namespace>

# Verify that your namespace has been created
kubectl get namespace
```

## 2-4. Practicing Basic Kubernetes Commands

In this section, we’ll practice the basic commands needed to operate a Kubernetes cluster.  
During the lab, you will primarily use `kubectl` to deploy and manage resources.  
These commands are frequently used and serve as the foundation for upcoming tasks.

### (1) Viewing Resources: `kubectl get <resource>`

Use this command to check the current state of Kubernetes resources.

```bash
kubectl get pods          # View pods in the current namespace
kubectl get services      # View services in the current namespace
kubectl get deployments   # View deployments in the current namespace
kubectl get namespaces    # View all namespaces in the cluster
kubectl get nodes         # View all nodes in the cluster
```

> [!tip]
>
> You can use **abbreviations** like:
>
> `pods` → `po`  
> `services` → `svc`  
> `deployments` → `deploy`  
> `namespaces` → `ns`

### (2) Namespace Options: `-n <namespace>` and `-A`

- `-n <namespace>`: View resources in a specific namespace
- `-A` or `--all-namespaces`: View resources across all namespaces

```bash
# View all pods in the 'nuc01' namespace
kubectl get po -n nuc01

# View all services across all namespaces
kubectl get svc -A
```

### (3) Show Extra Info: `-o wide`

Add the `-o wide` option to display more information, such as IP and node location.

```bash
# Shows which node each pod is running on, and its cluster IP
kubectl get pods -o wide
```

### (4) Create / Apply / Delete Resources: `create`, `apply`, `delete`

- `create`: Creates a new resource. Only works if the resource doesn’t already exist.
- `apply`: Creates or updates a resource based on a YAML file.
- `delete`: Deletes a resource.

```bash
# Example usage
kubectl apply -f deployment.yaml     # Apply or create the resource
kubectl delete -f deployment.yaml    # Delete the resource
```

These commands will be used repeatedly throughout the lab. You don’t need to memorize them, but getting used to them is important.

## 2-5. What is a YAML File?

In Kubernetes, resources are managed in a **declarative** way, and the core of this is the **YAML file**.

### What is YAML?

`YAML (YAML Ain’t Markup Language)` is a human-readable data serialization format.  
It expresses structured data based on indentation. Kubernetes uses YAML to define resources such as Deployment, Pod, and Service.

### Basic Structure Example

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: my-container
      image: nginx
```

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: my-backend:v1
```

You can apply YAML files to the cluster using:

```bash
kubectl apply -f <file>.yaml
```

#### Main Fields

- `apiVersion`: API version used for the resource
- `kind`: Type of resource (Pod, Service, Deployment, Secret, etc.)
- `metadata`: Metadata such as name, namespace, labels
- `spec`: The main configuration and behavior of the resource

### Tips for Writing YAML

- Use consistent indentation (2 or 4 spaces)
- Always put a space after `:`
- Badly formatted YAML will cause `kubectl apply` to fail

**Understanding YAML is one of the core skills in Kubernetes labs.**

> All `.yaml` files used in this lab are provided in the Git repository.  
> You may modify them or write your own as needed.

## 2-6. Cloning the Git Repository

### 2-6-1. Create Directory and Clone the Repository

First, create your personal lab directory based on your PC name:

```bash
cd ~
mkdir <your_directory>  # e.g., nuc01, nuc02, or nuc03
cd <your_directory>
```

This lab includes **code templates** for the Frontend and Backend, along with `yaml` templates to deploy them on Kubernetes.

Run the following command to clone the prepared repository:

```bash
git clone https://github.com/SmartX-Labs/SmartX-Mini.git
cp -r SmartX-Mini/SmartX-Mini-2025/Experiment/Lab-7.\ 3-Tier/* ./

# Check the templates
ls
```

### 2-6-2. Directory Architecture

```bash
.
├── backend
│   ├── Dockerfile
│   ├── eslint.config.mjs
│   ├── nest-cli.json
│   ├── package-lock.json
│   ├── package.json
│   ├── prisma
│   ├── README.md
│   ├── src
│   ├── tsconfig.build.json
│   └── tsconfig.json
├── frontend
│   ├── app.conf
│   ├── deploy.sh
│   ├── Dockerfile
│   ├── eslint.config.js
│   ├── index.html
│   ├── package-lock.json
│   ├── package.json
│   ├── public
│   ├── README.md
│   ├── src
│   └── vite.config.js
├── kubernetes
│   ├── backend
│   ├── container
│   ├── database
│   └── frontend
```

# 3. Database & Backend Deployment on Kubernetes

## 3-1. Database Deployment on Kubernetes

### 3-1-1. Creating a Persistent Volume

#### What is a Persistent Volume?

> **PersistentVolume (PV)**: Represents actual storage on a node in the Kubernetes cluster. It defines a connection to external storage (e.g., disk) so that data persists even when a Pod is deleted.
>
> **PersistentVolumeClaim (PVC)**: A resource used to request specific storage. A Pod uses a PVC to indirectly access the PV.

In other words, the PV is the actual storage, and the PVC is the user request. Connecting the two allows Pods to safely use external storage.

**First, edit the predefined Persistent Volume template to match your own namespace.**

```bash
cd ~/<your_directory>/kubernetes/database
vim postgres-pv.yaml
```

```yaml
# postgres-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv-<your_namespace>
  labels:
    volume: postgres-pv-<your_namespace>
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  hostPath:
    path: /mnt/data/<your_namespace>/postgres
  persistentVolumeReclaimPolicy: Retain
```

```bash
# Deploy the Persistent Volume for PostgreSQL
kubectl apply -f postgres-pv.yaml

# Check the created Persistent Volume
kubectl get pv
```

You may see PVs from other students as well.

### 3-1-2. Deploying PostgreSQL Database

Next, deploy the PostgreSQL Database:

```bash
vim postgres.yaml
```

The `postgres.yaml` defines three resources:

1. **Secret**: Stores user, password, and database information.
2. **Service**: Exposes PostgreSQL within the cluster.
3. **StatefulSet**: Maintains the actual PostgreSQL instance.

> [!note]
>
> **Secret**: Used to securely store sensitive information like passwords or API keys. Here, it securely stores PostgreSQL credentials so they can be referenced by other resources.
>
> **StatefulSet**: Used for deploying applications that maintain state (e.g., databases). Unlike Deployments, it ensures fixed names and volumes for each instance.

```yaml
# postgres.yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: <your_namespace>
type: Opaque
stringData:
  POSTGRES_USER: myuser
  POSTGRES_PASSWORD: mypassword
  POSTGRES_DB: mydb
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: <your_namespace>
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
  type: ClusterIP
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: <your_namespace>
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          envFrom:
            - secretRef:
                name: postgres-secret
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard
        selector:
          matchLabels:
            volume: postgres-pv-<your_namespace>
        resources:
          requests:
            storage: 5Gi
```

Apply the file:

```bash
kubectl apply -f postgres.yaml

# Check the created Secret
kubectl get secret -n <your_namespace>

# Check the Service
kubectl get svc -n <your_namespace>

# Check the StatefulSet
kubectl get statefulset -n <your_namespace>
```

You’ve now deployed the **PostgreSQL Database**, the Data Tier of the 3-Tier Architecture.

## 3-2. Backend Deployment on Kubernetes

Now, let’s deploy the **Backend Service**, which corresponds to the Application Tier in the 3-Tier Architecture.

### 3-2-1. Creating a Secret for Database Access

To connect the backend to PostgreSQL, it needs credentials and connection information.  
We will store this info in a Kubernetes Secret.

```bash
cd ~/<your_directory>/kubernetes/backend
vim secret.yaml
```

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: backend-secret
  namespace: <your_namespace>
type: Opaque
stringData:
  DATABASE_URL: "postgresql://myuser:mypassword@postgres.<your_namespace>.svc.cluster.local:5432/mydb"
  PASSWORD_SECRET: <your_password_secret>
```

Apply the file:

```bash
kubectl apply -f secret.yaml
kubectl get secret -n <your_namespace>
```

### 3-2-2. Creating the Backend Deployment

Now that the backend can access the database, let’s deploy it using a pre-built Docker image.

```bash
vim deployment.yaml
```

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: <your_namespace>
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend-api
  template:
    metadata:
      labels:
        app: backend-api
    spec:
      containers:
        - name: api
          image: cjfgml0306/backend:latest
          ports:
            - containerPort: 3000
          envFrom:
            - secretRef:
                name: backend-secret
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

Apply and check:

```bash
kubectl apply -f deployment.yaml
kubectl get deploy -n <your_namespace>
```

### 3-2-3. Creating the Backend Service

Now create the Service that exposes the backend inside the cluster.

```bash
vim service.yaml
```

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: <your_namespace>
spec:
  selector:
    app: backend-api
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
```

Apply the file:

```bash
kubectl apply -f service.yaml
kubectl get svc -n <your_namespace>
```

At this point, you’ve completed the deployment of the Data Tier (PostgreSQL) and Application Tier (Backend) in the 3-Tier Architecture.

# 4. Frontend Deployment on Kubernetes

## 4-1. Frontend Deployment 배경지식 요약

### 4-1-1. Kubernetes Object: ConfigMap

`4.2.1`에서 `ConfigMap`을 이용하여 설정 파일을 컨테이너에 삽입합니다. 그런데 여기서 사용하는 `ConfigMap`은 무엇일까요?

`ConfigMap`이란 평문으로 설정값을 저장하는 쿠버네티스 오브젝트입니다. Backend에서 활용했던 `Secret`과는 다르게, 외부에 노출되어도 괜찮은 정보들, 가령 URL이나 Port 번호, Log Level 등을 저장하여 여러 Container에서 환경 변수나 파일 형태로 가져와 활용할 수 있습니다.

`Secret`과 비슷하게, `ConfigMap`은 크게 2가지 방법으로 활용할 수 있습니다.

1. `ConfigMap`을 컨테이너의 환경 변수로 사용
2. `ConfigMap`을 컨테이너 내부에 파일로 마운트 (공식 문서 명칭으로 ConfigMap을 `Projection`(투사)한다고 말합니다.)

이번 실습의 Frontend 배포에서 Configmap은 후술할 NGINX의 설정 파일을 추가하는 데에 사용되며, 이를 파일 형태로 가져오기 위한 방법을 `kubernetes/frontend/fe-proxy.yaml` 파일에서 확인할 수 있습니다.

> [!note]
>
> 본 내용은 "시작하세요! 도커/쿠버네티스:친절한 설명으로 쉽게 이해하는 컨테이너 관리"(용찬호 저. 위키북스). pp 344~355의 내용을 일부 활용했습니다. 자세한 사항은 해당 도서를 참고해주시기 바랍니다.

### 4-1-2. React

Meta에서 개발한 Javascript 기반 Frontend 개발 라이브러리입니다. Angular와 Vue와 함께 Frontend 개발을 위해 주로 사용하는 라이브러리입니다.

단일 HTML 페이지에서 Javascript를 통해 페이지를 부분적으로 변경하여 서비스를 제공하는 SPA(Single Page Application) 방식의 웹서비스를 개발할 때 사용하며, 페이지의 구성요소를 Component라는 단위로 쪼개어 개발하고, 웹페이지를 동적으로 구성할 때 페이지 전체를 새로 구성하는 대신, 변경된 부분만을 수정합니다.

웹서비스 뿐만 아니라, 모바일 환경과 같은 네이티브 환경에서 UI를 개발할 때에 사용할 수 있습니다.

> [!note]
>
> 본 실습을 진행하기 위해 React 프로젝트의 소스코드를 이해하지 않아도 되기에 많은 내용을 기술하지는 않았습니다. 하지만 React 코드가 궁금한 경우 `frontend/README.md`에 코드를 이해하는 데에 필요한 정보를 적어두었으니 참고하시기 바랍니다.

### 4-1-3. NGINX

사용자나 브라우저에게 HTML 및 Javascript를 제공하려면, Backend 서버가 사용자 요청에 따라 파일을 전달해주거나 Proxy 서버가 대신 파일을 전달해야 합니다. 전자의 경우 NestJS 서버에 별도의 설정을 통해 HTML 파일을 전달하도록 구성하며, 후자의 경우 NGINX를 도입하여 사용자의 요청을 NGINX가 대신 받고, 요청에 따라 파일을 전달해주거나, Backend 서버에게 요청을 포워딩합니다.

![What-is-proxy](img/proxy.png)

Proxy 서버를 배치할 경우, Backend 서버는 파일을 제공하는 로직을 신경쓰지 않고 서비스에만 집중할 수 있게 되며, Proxy가 일부 요청을 대신 처리하므로 부하 감소를 얻을 수 있습니다.

NGINX는 이러한 목적을 위해 사용할 수 있는 프록시 서버로, 특정 요청에 대해 HTML과 같은 Static File을 대신 전달하거나, 요청에 알맞은 서버에게 요청을 포워딩하거나, 포워딩 과정에서 요청을 적절히 분산하여 부하를 고르게 부여하는 로드밸런싱을 수행할 수 있습니다.

본 실습에서는 React 프로젝트로 생성된 HTML 및 Javascript 파일을 사용자에게 전달하고, 요청을 Backend Server에게 포워딩하는 역할을 수행합니다. 이를 위해 사전에 빌드된 Docker 이미지를 활용할 예정입니다.

## 4-2. Frontend Deployment on Kubernetes

> [!note]
>
> 본 실습은 사전에 빌드된 이미지파일을 사용합니다.  
> 만약 웹UI를 수정하여 배포하고자 하는 경우, `./frontend` 경로에 프로젝트 원본 및 Dockerfile이 저장되어있으므로, 이를 활용하여 코드 수정 후 이미지를 새로 빌드해 활용하시기 바랍니다.

### 4-2-1. Frontend ConfigMap 생성

먼저, HTML 및 Javascript 파일을 NGINX를 통해 브라우저에게 전달할 수 있도록 설정하기 위해, ConfigMap을 생성하도록 하겠습니다.

다음의 명령어로 `fe-proxy-cm.yaml` 파일을 열도록 하겠습니다.

```bash
cd ~/<your_directory\>/kubernetes/frontend
vim fe-proxy-cm.yaml
```

`fe-proxy-cm.yaml` 파일에서 Namespace를 자신의 Namespace로 수정하고, `upstream` 부분의 URL을 Backend 서버의 URL로 수정하겠습니다. 파일 내용은 다음과 같이 구성됩니다.

```YAML
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-proxy-cm
  namespace: <your_namespace>    # EDIT THIS
data:
  app.conf: |    # EDIT THIS
    upstream backend-svc {
      server backend-svc.<your_namespace>.svc.cluster.local:80;
    }
    server {
      listen 80;

      root /usr/share/nginx/html;
      index index.html;

      location /api/ {
        rewrite           ^/api(.*)$ $1 break;
        proxy_pass        http://backend-svc;
        proxy_redirect    off;
        proxy_set_header  Host  $host;
        proxy_set_header  X-Real-IP         $remote_addr;
        proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header  X-Forwarded-Host  $server_name;
      }

      location / {
        try_files $uri /index.html;
      }
    }

```

위의 내용에서 `metadata.namespace`의 값을 자신의 Namespace로 수정하고, `data."app.conf"`의 `upstream.server`의 값을 Backend Service의 URL로 수정해주십시오. (참고: 동일 Cluster 내 Service의 FQDN: `<svc-name>.<namespace>.svc.cluster.local`)

수정 후, 다음의 명령어를 입력하여 ConfigMap을 생성합니다.

```bash
kubectl apply -f ./fe-proxy-cm.yaml
```

이후, 제대로 생성되었는지 확인하기 위해 다음의 명령어를 입력합니다.

```bash
kubectl get cm -n <your_namespace>  # 네임스페이스에 존재하는 ConfigMap 목록 확인
kubectl describe cm nginx-proxy-cm -n <your_namespace> # nginx-proxy-cm의 내용 확인
```

### 4-2-2. Frontend Deployment 생성

다음으로, HTML 파일이 포함된 NGINX Deployment를 생성하도록 하겠습니다.

하단의 명령어를 입력하여 `fe-proxy.yaml` 파일을 열도록 하겠습니다.

```bash
vim ./fe-proxy.yaml
```

`fe-proxy.yaml` 파일의 내용은 다음과 같습니다. 파일 내용에서 `metadata.namespace`의 값을 자신의 Namespace로 수정합니다.

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-proxy
  namespace: <your_namespace>    # EDIT THIS
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-proxy
  template:
    metadata:
      labels:
        app: nginx-proxy
    spec:
      volumes:
        - name: fe-serving-config
          configMap:
            name: nginx-proxy-cm
            items:
              - key: app.conf
                path: app.conf
      containers:
        - name: nginx
          image: "tori209/smartx-fe:latest"
          imagePullPolicy: Always
          ports:
            - containerPort: 80
          volumeMounts:
            - name: fe-serving-config
              mountPath: /etc/nginx/conf.d
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

수정 후, 다음의 명령어를 입력하여 NGINX Deployment를 생성합니다.

```bash
kubectl apply -f ./fe-proxy.yaml
```

이후, 제대로 생성되었는지 확인하기 위해 다음의 명령어를 입력합니다.

```bash
kubectl get deploy -n <your_namespace>  # 네임스페이스에 존재하는 Deployment 목록 확인
kubectl describe deploy nginx-proxy -n <your_namespace> # nginx-proxy의 내용 확인
kubectl get pod -n <your_namespace> # Deployment에 의한 Pod 생성 확인
```

# 5. 배포된 웹 서비스 확인

## 5-1. 브라우저 접근

### 5-1-1. 웹서버 IP 확인

이제, 브라우저를 통해 배포된 웹 서비스에 접근하기 위해 다음을 입력하여 접근할 IP 주소를 확인하겠습니다.

```bash
# `-o wide`: 출력에 더 많은 정보(Cluster-wide IP, Node Name, ...)를 표시합니다.
kubectl get pod -n <your_namespace> -o wide
```

위의 명령을 통해 다음과 같이 출력됩니다. 이 중에서 `nginx-proxy`로 시작하는 Pod의 IP를 복사합니다.

![Deploy List](./img/deploy-complete.png)

위의 예시의 경우, 접근할 IP 주소는 `10.244.2.107`입니다.

> [!NOTE]
>
> 일반적으로 Pod의 Cluster 내부 IP를 이용하여 Pod에 접근할 수 없으나, 장비가 Cluster의 Node로 포함된 경우에 한하여 직접 접근이 가능합니다.
> 이는 각 Node에서 Pod에게 패킷을 전달하기 위해, Kernel Routing Table과 IPTables에 Pod에게 패킷을 전달할 방법을 명시하기 때문입니다.

### 5-1-2. 웹서비스 확인

브라우저의 주소창에 `http://<nginx-ip-addr>`을 입력해 접근합니다. 그러면 다음과 같은 화면이 표시됩니다.

![WebPage_Main](./img/webpage.png)

우리가 배포한 것은 익명 게시판 서비스로, 회원가입 없이 누구나 게시글을 작성하고 조회할 수 있습니다. 현재는 어떠한 게시글도 등록되지 않아 게시글이 보이지 않는 상태입니다. '게시글 작성' 버튼을 눌러 게시글을 추가해보도록 하겠습니다. 버튼을 클릭하면 다음과 같은 화면이 보입니다.

![WebPage_Write](./img/write.png)

원하는 제목과 닉네임, 패스워드, 본문 내용을 작성한 뒤, "작성" 버튼을 클릭합니다. <br>
그러면 다음과 같이, 자신이 작성한 글을 확인할 수 있습니다.

> [!Note]
>
> Ubuntu 설치 과정에서 한글 입력기를 별도로 추가하지 않았다면, 한글을 입력할 수 없으니 유의 바랍니다. <br>
> 설정 방법은 이번 실습과 무관하므로 별도로 설명하지 않지만, 인터넷에 다양한 참고 자료가 있으니 이를 활용하시기 바랍니다.

![WebPage_with_post](./img/list-with-post.png)

다음으로, 검색 기능을 확인해보겠습니다. 현 검색 기능은 제목+본문 검색을 수행하니 참고 바랍니다.

![WebPage_Search](./img/post-search.png)

확인이 끝났다면 "게시판 목록"을 클릭하여 되돌아오겠습니다. <br>
다음으로 자신이 작성한 글을 클릭하여 확인해보겠습니다.

![WebPage_Detail](./img/post-without-comment.png)

화면과 같이 작성했던 제목과 닉네임, 본문 내용을 볼 수 있으며, 댓글 작성칸, 댓글 목록을 확인할 수 있습니다.

이제 댓글을 추가하겠습니다. 원하는 내용을 적고 "댓글 작성" 버튼을 누르면 다음과 같이 확인할 수 있습니다.

![WebPage_WithComment](./img/post-with-comment.png)

이제 게시글을 수정해보겠습니다. "수정" 버튼을 클릭하면 다음과 같은 수정 페이지를 확인할 수 있습니다.

![WebPage_Edit](./img/edit-post.png)

원하는 문구로 수정한 뒤, 올바른 패스워드를 입력하면 수정이 완료됩니다. <br>
만약 패스워드 오류, 혹은 서버 오류 발생 시 오류 메세지가 출력되니 참고 바랍니다.

추가로, 패스워드도 동일한 방법으로 수정이 가능합니다.

![WebPage_Editing](./img/comment-editing.png)

수정이 완료되면 다음과 같이 확인할 수 있습니다.

![WebPage_AfterEdit](./img/post-after-edit.png)

마지막으로 Post를 삭제하겠습니다. "삭제" 버튼을 누른 뒤, 게시글 패스워드를 입력하여 삭제해주십시오.
그러면 다음과 같이 삭제된 것을 확인할 수 있습니다.

![WebPage_AfterDel](./img/webpage.png)

이것으로 익명게시판의 기능을 모두 점검해보았습니다. 추가로 다른 조원의 서비스에 접근해 확인해보는 것을 권장합니다.

## 5-2. Server 확인하기

> [!TIP]
>
> 후술할 과정으로 하나의 사용자 동작을 처리하기 위해 어떻게 요청이 전달되고 처리되는지 엿볼 수 있습니다. <br>
> 더 나은 체험을 위해, 직접 웹서비스를 사용해보며 실시간 동작을 확인해보는 것을 권장합니다.

### 5-2-1. NGINX 확인

앞서 언급했듯, NGINX는 요청을 대신 받아 직접 처리하거나, 다른 서버에게 요청을 넘겨주는 역할을 합니다.  
NGINX가 어떻게 요청을 처리했는지 확인하려면 이의 로그를 확인해야 합니다. 이는 다음의 명령어로 확인할 수 있습니다.

```bash
# 이름을 모를 경우 `kubectl -n <your_namespace> get po`로 확인합니다.
kubectl -n <your_namespace> logs <nginx-proxy-pod>
```

로그를 통해 어떤 요청이 다른 서버에게 포워딩되었으며, 어떤 요청을 직접 처리했는지 확인할 수 있습니다.

### 5-2-2. DB 확인

웹서버는 데이터를 저장하기 위해 DB를 사용합니다. <br>
여러분이 작성한 댓글이나 게시글은 DB에 저장되고, 사용자가 요청을 보낼 때마다 이를 꺼내어 전달해줍니다.

그렇기에, DB의 Table을 확인하면 여러분의 웹서비스 화면에 보여졌던 것과 동일한 내용을 확인할 수 있습니다.

먼저, PostgreSQL DB에 접근하기 위해, DB Pod에 접근하여 CLI 도구를 실행하도록 하겠습니다. 명령은 다음과 같이 입력합니다.

```bash
# 현 설정에서 PostgreSQL Pod의 이름은 postgres-0 입니다. 아닐 경우 `kubectl -n <your_namespace> get po`로 확인합니다.
# `-U`: PostgreSQL에 등록된 사용자 ID 지정
# `-d`: 접근 시 연결할 database 이름
kubectl -n <your_namespace> exec -it po/postgres-0 -- psql -U myuser -d mydb
```

> [!note]
>
> 위의 명령은 "지정한 Pod에서 다음의 명령을 실행한다"는 의미입니다. 옵션은 다음과 같습니다.
>
> | Option           | Description                                                                                                                                                                                             |
> | :--------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
> | `-n <namespace>` | Pod가 위치한 namespace를 지정합니다.                                                                                                                                                                    |
> | `-it`            | `--stdin`(`-i`)와 `--tty`(`-t`) 옵션으로, 사용자 입력(키보드 등)을 `exec` 실행 동안 Pod에게 전달하기 위해 필요합니다.<br>(간단하게, TTY=로컬 콘솔 접근을 위한 드라이버 / PTY=원격 접속을 위한 드라이버) |
> | `-- <cmd>`       | Pod 내부에서 실행할 명령어를 지정합니다. `--`와 `<cmd>` 사이에 공백이 있으므로 주의합니다.                                                                                                              |

그러면 PostgreSQL CLI 도구가 활성화된 것을 볼 수 있습니다.
![psql_start](img/psql_img.png)

다음으로, 존재하는 Table 목록을 조회하기 위해 `\d`을 입력합니다. <br>
![psql_tables](./img/psql_tables.png)

> [!TIP]
>
> PostgreSQL에서 `\d`는 연결된 Database 내 Table 목록을 출력합니다. <br>
> 그리고 `\d <table_name>`을 입력하면 Table의 Column 정보를 출력합니다.
>
> 또한, `\d`는 기본 정보만 출력하지만, `\d+`는 여러 세부 정보를 추가로 보여줍니다.

여기서 `Post`가 작성한 게시글을 저장하는 Table이며, `Comment`가 댓글을 저장하는 Table입니다. <br>
내용을 보기 위해, 다음의 명령을 입력합니다.

```SQL
-- ※ SQL은 `;`으로 끝맺어야 명령 입력이 완료되었다고 인식합니다.
SELECT * FROM "Post";     -- Post 전체 조회
SELECT * FROM "Comment";  -- Comment 전체 조회
```

그러면 여러분이 입력했던 데이터가 출력될 것입니다.<br>
참고로 Password는 보안을 위해 Hash Function으로 평문을 알아볼 수 없게 처리하니 유의하기 바랍니다.

종료는 `\q`, 혹은 `ctrl+d`를 입력하면 됩니다.

# 6. Lab Review

이번 Lab에서는 `3-tier 구조`의 웹 서비스를 kubernetes 환경에서 배포하는 경험을 해보았습니다.

3-tier 구조는 사용자와 직접 상호작용하는 `Presentation Tier`, Presentation tier에서 발생한 요청을 처리하고 데이터베이스와 통신하는 `Application Tier`, 생성된 데이터를 영구적으로 저장 및 관리하는 `Data Tier`의 형태를 나타냅니다.

이러한 구조를 가진 서비스를 container orchestration 도구인 kubernetes를 이용하여 배포해보았습니다. 실습을 위해 적은 수의 pod가 생성되도록 했지만, 규모있는 실제 서비스를 운영하게 된다면 kubernetes 환경에서 손쉽게 scaling을 할 수 있다는 특성을 바탕으로 적은 노력을 들여 안정적인 서비스를 운영할 수 있을 것입니다.

우리는 yaml 파일을 통한 `선언형` 방식을 통해, kubernetes의 다양한 리소스를 생성할 수 있었고 Frontend, Backend, Database를 배포하여 서비스의 작동을 확인할 수 있었습니다.

또한, Frontend(Presentation Tier)와 Backend(Application Tier) 사이의 proxy server 역할을 하는 NGINX도 사용해보았습니다.

이번 실습을 응용한다면, 여러분은 각자 본인만의 Frontend, Backend 서비스를 개발하고, 이를 Database와 함께 container image로 만든 후, kubernetes 환경에서 배포할 수 있을 것입니다. 이를 통해 구조를 갖춘 하나의 서비스를 개발하는 경험을 해볼 수 있을 것입니다.
