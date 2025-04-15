### Minikube 的作用

Minikube 是一个轻量级的 Kubernetes 工具，用于在本地或单台服务器上快速部署和运行一个单节点的 Kubernetes 集群。它主要用于学习、开发和测试 Kubernetes 环境，帮助用户在无需复杂多节点集群的情况下，模拟 Kubernetes 的核心功能。Minikube 支持在 macOS、Linux 和 Windows 上运行，适合开发者在本地环境中部署和管理容器化应用，如 WordPress。

通过 Minikube，你可以在 Google 服务器上创建一个小型 Kubernetes 集群，部署 WordPress 和 MySQL 容器，学习容器编排、Pod 管理、Service 暴露等 Kubernetes 概念。它特别适合初学者快速上手 Kubernetes。

---

### 在 Google 服务器上使用 Minikube 安装 WordPress 个人网站的完整流程

以下是在 Google Cloud Platform (GCP) 服务器上使用 Minikube 部署 WordPress 个人网站的详细步骤。假设你已经有一个运行 Ubuntu 的 GCP 虚拟机实例（VM），并且有基本的 Linux 操作知识。

#### 准备工作
1. **创建或选择 GCP 虚拟机**
   - 登录 Google Cloud Console（https://console.cloud.google.com/）。
   - 创建一个新的 Compute Engine 虚拟机（推荐 Ubuntu 20.04 LTS 或 22.04 LTS）。
   - 选择适当的配置，例如 `e2-standard-2`（2 vCPU，8GB 内存）或更高，以确保 Minikube 运行顺畅。
   - 确保防火墙规则允许 HTTP（端口 80）和 HTTPS（端口 443）流量，以便访问 WordPress 网站。
   - 记录虚拟机的外部 IP 地址。

2. **连接到虚拟机**
   - 使用 SSH 登录到虚拟机：
     ```bash
     gcloud compute ssh [INSTANCE_NAME] --zone [ZONE]
     ```
     或者通过 Google Cloud Console 的 SSH 按钮连接。

3. **更新系统**
   - 确保系统是最新的：
     ```bash
     sudo apt update && sudo apt upgrade -y
     ```

---

#### 步骤 1：安装 Docker
Minikube 需要容器运行时，Docker 是最常用的驱动。

1. 安装 Docker：
   ```bash
   sudo apt install -y docker.io
   sudo systemctl start docker
   sudo systemctl enable docker
   ```

2. 将当前用户添加到 Docker 组，避免每次运行 Docker 命令都需要 `sudo`：
   ```bash
   sudo usermod -aG docker $USER
   ```
   退出并重新登录以应用更改：
   ```bash
   exit
   gcloud compute ssh [INSTANCE_NAME] --zone [ZONE]
   ```

3. 验证 Docker 安装：
   ```bash
   docker --version
   ```

---

#### 步骤 2：安装 kubectl
kubectl 是 Kubernetes 的命令行工具，用于与集群交互。

1. 下载并安装 kubectl：
   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   chmod +x kubectl
   sudo mv kubectl /usr/local/bin/
   ```

2. 验证安装：
   ```bash
   kubectl version --client
   ```

---

#### 步骤 3：安装 Minikube
1. 下载并安装 Minikube：
   ```bash
   curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
   chmod +x minikube-linux-amd64
   sudo mv minikube-linux-amd64 /usr/local/bin/minikube
   ```

2. 验证安装：
   ```bash
   minikube version
   ```

---

#### 步骤 4：启动 Minikube 集群
1. 启动 Minikube，使用 Docker 驱动：
   ```bash
   minikube start --driver=docker
   ```
   - 这将创建一个单节点的 Kubernetes 集群。
   - 如果遇到镜像拉取问题，可以配置国内镜像加速器（例如阿里云或 Docker Hub 镜像）。

2. 验证集群状态：
   ```bash
   minikube status
   kubectl get nodes
   ```
   你应该看到一个节点处于 `Ready` 状态。

---

#### 步骤 5：部署 MySQL（WordPress 的数据库）
WordPress 需要 MySQL 数据库，我们将使用 Kubernetes 的 Pod 和 Secret 来部署。

1. **创建 MySQL Secret**
   创建一个 Kubernetes Secret 来存储 MySQL 的密码：
   ```bash
   kubectl create secret generic mysql-pass --from-literal=password=your-mysql-password
   ```
   将 `your-mysql-password` 替换为你希望的密码。

2. **创建 MySQL 部署文件**
   创建一个文件 `mysql-deployment.yaml`：
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: wordpress-mysql
     labels:
       app: wordpress
   spec:
     ports:
       - port: 3306
     selector:
       app: wordpress
       tier: mysql
     clusterIP: None
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: wordpress-mysql
     labels:
       app: wordpress
   spec:
     selector:
       matchLabels:
         app: wordpress
         tier: mysql
     strategy:
       type: Recreate
     template:
       metadata:
         labels:
           app: wordpress
           tier: mysql
       spec:
         containers:
         - image: mysql:8.0
           name: mysql
           env:
           - name: MYSQL_ROOT_PASSWORD
             valueFrom:
               secretKeyRef:
                 name: mysql-pass
                 key: password
           ports:
           - containerPort: 3306
             name: mysql
           volumeMounts:
           - name: mysql-persistent-storage
             mountPath: /var/lib/mysql
         volumes:
         - name: mysql-persistent-storage
           emptyDir: {}
   ```

3. **部署 MySQL**
   应用配置文件：
   ```bash
   kubectl apply -f mysql-deployment.yaml
   ```

4. **验证 MySQL 部署**
   检查 Pod 状态：
   ```bash
   kubectl get pods
   ```
   确保 `wordpress-mysql` 的 Pod 处于 `Running` 状态。

---

#### 步骤 6：部署 WordPress
1. **创建 WordPress 部署文件**
   创建一个文件 `wordpress-deployment.yaml`：
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: wordpress
     labels:
       app: wordpress
   spec:
     ports:
       - port: 80
         targetPort: 80
         nodePort: 30080
     selector:
       app: wordpress
       tier: frontend
     type: NodePort
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: wordpress
     labels:
       app: wordpress
   spec:
     selector:
       matchLabels:
         app: wordpress
         tier: frontend
     strategy:
       type: Recreate
     template:
       metadata:
         labels:
           app: wordpress
           tier: frontend
       spec:
         containers:
         - image: wordpress:latest
           name: wordpress
           env:
           - name: WORDPRESS_DB_HOST
             value: wordpress-mysql
           - name: WORDPRESS_DB_PASSWORD
             valueFrom:
               secretKeyRef:
                 name: mysql-pass
                 key: password
           - name: WORDPRESS_DB_USER
             value: root
           - name: WORDPRESS_DB_NAME
             value: wordpress
           ports:
           - containerPort: 80
             name: wordpress
           volumeMounts:
           - name: wordpress-persistent-storage
             mountPath: /var/www/html
         volumes:
         - name: wordpress-persistent-storage
           emptyDir: {}
   ```

2. **部署 WordPress**
   应用配置文件：
   ```bash
   kubectl apply -f wordpress-deployment.yaml
   ```

3. **验证 WordPress 部署**
   检查 Pod 状态：
   ```bash
   kubectl get pods
   ```
   检查 Service：
   ```bash
   kubectl get services
   ```

---

#### 步骤 7：访问 WordPress
1. **获取访问地址**
   Minikube 部署的 WordPress 使用 NodePort（默认 30080）暴露服务。你可以通过以下命令获取访问 URL：
   ```bash
   minikube service wordpress --url
   ```
   或者直接使用虚拟机的外部 IP 和 NodePort：
   ```
   http://[VM_EXTERNAL_IP]:30080
   ```

2. **完成 WordPress 安装**
   - 打开浏览器，访问上述 URL。
   - 按照 WordPress 的安装向导，设置管理员账户、站点标题等。
   - 登录 WordPress 后台，开始配置你的网站。

---

#### 步骤 8：（可选）配置持久化存储
在上述配置中，MySQL 和 WordPress 使用 `emptyDir` 作为临时存储，数据会在 Pod 重启后丢失。为生产环境，建议配置持久化存储：

1. **创建 PersistentVolume 和 PersistentVolumeClaim**
   示例 `mysql-pv.yaml`：
   ```yaml
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: mysql-pv
     labels:
       type: local
   spec:
     storageClassName: manual
     capacity:
       storage: 5Gi
     accessModes:
       - ReadWriteOnce
     hostPath:
       path: "/mnt/data/mysql"
   ---
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: mysql-pvc
   spec:
     storageClassName: manual
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 5Gi
   ```

2. 更新 `mysql-deployment.yaml` 和 `wordpress-deployment.yaml`，将 `emptyDir` 替换为 `persistentVolumeClaim`：
   ```yaml
   volumes:
   - name: mysql-persistent-storage
     persistentVolumeClaim:
       claimName: mysql-pvc
   ```

3. 应用更改：
   ```bash
   kubectl apply -f mysql-pv.yaml
   kubectl apply -f mysql-deployment.yaml
   kubectl apply -f wordpress-deployment.yaml
   ```

---

#### 步骤 9：（可选）配置外部访问
如果希望通过域名访问 WordPress，可以：

1. **配置 Ingress**
   安装 Ingress 控制器（如 Nginx Ingress）到 Minikube：
   ```bash
   minikube addons enable ingress
   ```

2. 创建 Ingress 资源：
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: wordpress-ingress
   spec:
     rules:
     - host: your-domain.com
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: wordpress
               port:
                 number: 80
   ```

3. 更新 DNS 记录，将域名指向虚拟机的外部 IP。

---

#### 步骤 10：清理和停止
1. **停止 Minikube**
   ```bash
   minikube stop
   ```

2. **删除 Minikube 集群**
   ```bash
   minikube delete
   ```

3. **删除 Kubernetes 资源**
   ```bash
   kubectl delete -f wordpress-deployment.yaml
   kubectl delete -f mysql-deployment.yaml
   kubectl delete secret mysql-pass
   ```

---

### 注意事项
- **资源分配**：确保虚拟机有足够的 CPU 和内存（建议至少 2 vCPU 和 8GB 内存）。
- **镜像加速**：在国内环境，可配置 Docker 镜像加速器（如阿里云镜像）以加快镜像拉取。
- **安全性**：生产环境中，建议使用安全的 MySQL 密码，并配置 SSL（通过 Ingress 或 Let's Encrypt）。
- **备份**：定期备份 WordPress 数据和 MySQL 数据库。
- **GCP 费用**：注意 GCP 的免费额度使用情况，避免产生额外费用。

### 参考资源
- Minikube 官方文档：https://minikube.sigs.k8s.io/docs/[](https://minikube.kubernetes.ac.cn/docs/)
- Kubernetes WordPress 示例：https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/[](https://k8smeetup.github.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/)
- Google Cloud 部署 WordPress：https://wpjian.com/how-to-install-wordpress-on-google-cloud/[](https://wpjian.com/tips/2020042932417.html)

