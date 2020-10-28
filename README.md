# Kubernetes Setup

Prepare At least 1 Virtual Machine (Master), better to have more VM for workers (Worker)

## Install Tool

Official Steps: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

Other Source (My reference): https://github.com/justmeandopensource/kubernetes/blob/master/docs/install-cluster-ubuntu-20.md

Calico Source: https://docs.projectcalico.org/getting-started/kubernetes/

> Run as su

    sudo su

> Disable Firewall

    sudo ufw disable

> Disable swap

    swapoff -a; sed -i '/swap/d' /etc/fstab

> Update sysctl settings for Kubernetes networking

    cat >>/etc/sysctl.d/kubernetes.conf<<EOF
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    EOF
    sysctl --system

> Install Docker

    apt update
    apt-get install docker.io

> Add Kubernetes Apt Repository

    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list

> Install Kubeadm Kubelet Kubectl

    apt update
    apt-get install kubeadm kubelet kubectl

## Create Cluster

> Init Cluster

    kubeadm init

> Deploy Calico network

    kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml

* If you are using cloud platform (GCP, AWS, AZURE), remember to open internal access in VPC network.

> Setup User

    exit (Exit su)
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Kubeadm

> On Master, Create Token

    kubeadm token create --print-join-command

> On Slave

    Run the Command outputed from master.

# kubectl

## Taint

Official Page: https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/

> Add Taint on kmaster (By default master node dont deploy pods)

    kubectl taint node kmaster node-role.kubernetes.io/master:NoSchedule

> Remove Taint on kmaster

    kubectl taint node kmaster node-role.kubernetes.io/master:NoSchedule-

# Kubernetes Dashboard

Official Steps: https://kubernetes.io/zh/docs/tasks/access-application-cluster/web-ui-dashboard/

> Deploy and Expose kubernetes dashboard

    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml

## Create Admin User

Official Steps: https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md

> Create service acount

    cat <<EOF | kubectl apply -f -
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: admin-user
      namespace: kubernetes-dashboard
    EOF

> Create ClusterRoleBinding

    cat <<EOF | kubectl apply -f -
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: admin-user
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
    - kind: ServiceAccount
      name: admin-user
      namespace: kubernetes-dashboard
    EOF

> Get Token (\*\*\*Fking GCP SSH Panel will break the token)

    kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')

> Delete Service Account

    kubectl -n kubernetes-dashboard delete serviceaccount admin-user
    kubectl -n kubernetes-dashboard delete clusterrolebinding admin-user

> Run on Local Host

    kubectl proxy

Then you can access on http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/.

## Access via Internet

> Change dashboard service to NodePort

    kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard

Access it on `https://MasterHostIP:DashboardPort`

# Jenkins

For CICD Usage

## Deploy Jenkins in Cluster

Official Step: https://www.jenkins.io/doc/book/installing/kubernetes/

> Creata Namespaces

    kubectl create namespace jenkins

> Create Jenkins Service Account

    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: jenkins
      namespace: jenkins

    ---

    kind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: jenkins
    rules:
      - apiGroups: ["extensions", "apps"]
        resources: ["deployments"]
        verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
      - apiGroups: [""]
        resources: ["services"]
        verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
      - apiGroups: [""]
        resources: ["pods"]
        verbs: ["create","delete","get","list","patch","update","watch"]
      - apiGroups: [""]
        resources: ["pods/exec"]
        verbs: ["create","delete","get","list","patch","update","watch"]
      - apiGroups: [""]
        resources: ["pods/log"]
        verbs: ["get","list","watch"]
      - apiGroups: [""]
        resources: ["secrets"]
        verbs: ["get"]
      - apiGroups: [""]
        resources: ["events"]
        verbs: ["get","list","watch"]

    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: jenkins
      namespace: jenkins
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: jenkins
    subjects:
      - kind: ServiceAccount
        name: jenkins
        namespace: jenkins

> Deploy Jenkins by yaml (should deploy on master)

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: jenkins
      namespace: jenkins
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: jenkins
      template:
        metadata:
          labels:
            app: jenkins
            spec:
          serviceAccount: jenkins
          containers:
          - name: jenkins
            image: jenkins/jenkins:lts
            ports:
            - containerPort: 8080
              name: web
              protocol: TCP
            - containerPort: 50000
              name: agent
              protocol: TCP
            volumeMounts:
              - name: jenkins-home
                mountPath: /var/jenkins_home
          volumes:
            - name: jenkins-home
              emptyDir: {}

\*\* Remember to untaint master node, otherwise pods will not deploy on master node

> Deploy Jenkins Service by yaml

    apiVersion: v1
    kind: Service
    metadata:
      name: jenkins
      namespace: jenkins
    spec:
    type: NodePort
    ports:
    - name: web
      port: 8080
      targetPort: web
      nodePort: 30002
    - name: agent
      port: 50000
      targetPort: agent
    selector:
        app: jenkins

Access Jenkins UI on `MasterHostIP:JenkinPort`

> Get Initial Password

    kubectl get pods -n jenkins
    kubectl logs <pod_name> -n jenkins

## Jenkins Setup

Official Site: https://plugins.jenkins.io/kubernetes/

1. Install Kubernetes Plugin (Manage Jenkins > Manage Plugin)

2. Setup Jenkins Slave (Manage Jenkins > Manage Node And Cloud > Configure Cloud)

\*\* Add New Kubernetes Cloud

\*\* Test Connection to ensure that jenkins is running perfectly

- Jenkins URL: http://MasterIP:ExposedPort(8080)

- Jenkins Tunnel: MasterIP:ExposedPort(50000)  *No http://, No / at the end

3. Add Pod Template

- Name: jenkins-slave
- Label: jenkins-slave
- Usage: Use this node as much as possible
- Add Container Template
  - Name: jenkins-slave
  - Docker image: jenkinsci/jnlp-slave
  - Working Directory: /home/jenkins
  - Add Volume (Host path volume) \*\*Allow Slave to run docker and kubectl
    - key: /usr/bin/docker
    - value: /usr/bin/docker
    - key: /var/run/docker.sock
    - value: /var/run/docker.sock
    - key: /usr/bin/kubectl
    - value: /usr/bin/kubectl
- Service Account: jenkins

(Optional, set jenkins slave to be ran on master)
- Set Node Selector

    `kubernetes.io/hostname=$MasterNodeName`

4. Edit permission for slave

   `sudo chmod 666 /var/run/docker.sock`

## Jenkins Script

> Put Node('jenkins-slave)

    node('jenkins-slave') {
        stage('Test') {
        echo "2.Test Stage"
        }
    }
# Kubernetes Setup

Prepare At least 1 Virtual Machine (Master), better to have more VM for workers (Worker)

## Install Tool

Official Steps: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

Other Source (My reference): https://github.com/justmeandopensource/kubernetes/blob/master/docs/install-cluster-ubuntu-20.md

Calico Source: https://docs.projectcalico.org/getting-started/kubernetes/

> Run as su

    sudo su

> Disable Firewall

    sudo ufw disable

> Disable swap

    swapoff -a; sed -i '/swap/d' /etc/fstab

> Update sysctl settings for Kubernetes networking

    cat >>/etc/sysctl.d/kubernetes.conf<<EOF
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    EOF
    sysctl --system

> Install Docker

    apt update
    apt-get install docker.io

> Add Kubernetes Apt Repository

    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list

> Install Kubeadm Kubelet Kubectl

    apt update
    apt-get install kubeadm kubelet kubectl

## Create Cluster

> Init Cluster

    kubeadm init

> Deploy Calico network

    kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml

* If you are using cloud platform (GCP, AWS, AZURE), remember to open internal access in VPC network.

> Setup User

    exit (Exit su)
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Kubeadm

> On Master, Create Token

    kubeadm token create --print-join-command

> On Slave

    Run the Command outputed from master.

# kubectl

## Taint

Official Page: https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/

> Add Taint on kmaster (By default master node dont deploy pods)

    kubectl taint node kmaster node-role.kubernetes.io/master:NoSchedule

> Remove Taint on kmaster

    kubectl taint node kmaster node-role.kubernetes.io/master:NoSchedule-

# Kubernetes Dashboard

Official Steps: https://kubernetes.io/zh/docs/tasks/access-application-cluster/web-ui-dashboard/

> Deploy and Expose kubernetes dashboard

    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml

## Create Admin User

Official Steps: https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md

> Create service acount

    cat <<EOF | kubectl apply -f -
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: admin-user
      namespace: kubernetes-dashboard
    EOF

> Create ClusterRoleBinding

    cat <<EOF | kubectl apply -f -
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: admin-user
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
    - kind: ServiceAccount
      name: admin-user
      namespace: kubernetes-dashboard
    EOF

> Get Token (\*\*\*Fking GCP SSH Panel will break the token)

    kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')

> Delete Service Account

    kubectl -n kubernetes-dashboard delete serviceaccount admin-user
    kubectl -n kubernetes-dashboard delete clusterrolebinding admin-user

> Run on Local Host

    kubectl proxy

Then you can access on http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/.

## Access via Internet

> Change dashboard service to NodePort

    kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard

Access it on `https://MasterHostIP:DashboardPort`

# Jenkins

For CICD Usage

## Deploy Jenkins in Cluster

Official Step: https://www.jenkins.io/doc/book/installing/kubernetes/

> Creata Namespaces

    kubectl create namespace jenkins

> Create Jenkins Service Account

    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: jenkins
      namespace: jenkins

    ---

    kind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: jenkins
    rules:
      - apiGroups: ["extensions", "apps"]
        resources: ["deployments"]
        verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
      - apiGroups: [""]
        resources: ["services"]
        verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
      - apiGroups: [""]
        resources: ["pods"]
        verbs: ["create","delete","get","list","patch","update","watch"]
      - apiGroups: [""]
        resources: ["pods/exec"]
        verbs: ["create","delete","get","list","patch","update","watch"]
      - apiGroups: [""]
        resources: ["pods/log"]
        verbs: ["get","list","watch"]
      - apiGroups: [""]
        resources: ["secrets"]
        verbs: ["get"]
      - apiGroups: [""]
        resources: ["events"]
        verbs: ["get","list","watch"]

    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: jenkins
      namespace: jenkins
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: jenkins
    subjects:
      - kind: ServiceAccount
        name: jenkins
        namespace: jenkins

> Deploy Jenkins by yaml (should deploy on master)

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: jenkins
      namespace: jenkins
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: jenkins
      template:
        metadata:
          labels:
            app: jenkins
            spec:
          serviceAccount: jenkins
          containers:
          - name: jenkins
            image: jenkins/jenkins:lts
            ports:
            - containerPort: 8080
              name: web
              protocol: TCP
            - containerPort: 50000
              name: agent
              protocol: TCP
            volumeMounts:
              - name: jenkins-home
                mountPath: /var/jenkins_home
          volumes:
            - name: jenkins-home
              emptyDir: {}

\*\* Remember to untaint master node, otherwise pods will not deploy on master node

> Deploy Jenkins Service by yaml

    apiVersion: v1
    kind: Service
    metadata:
      name: jenkins
      namespace: jenkins
    spec:
    type: NodePort
    ports:
    - name: web
      port: 8080
      targetPort: web
      nodePort: 30002
    - name: agent
      port: 50000
      targetPort: agent
    selector:
        app: jenkins

Access Jenkins UI on `MasterHostIP:JenkinPort`

> Get Initial Password

    kubectl get pods -n jenkins
    kubectl logs <pod_name> -n jenkins

## Jenkins Setup

Official Site: https://plugins.jenkins.io/kubernetes/

1. Install Kubernetes Plugin (Manage Jenkins > Manage Plugin)

2. Setup Jenkins Slave (Manage Jenkins > Manage Node And Cloud > Configure Cloud)

\*\* Add New Kubernetes Cloud

\*\* Test Connection to ensure that jenkins is running perfectly

- Jenkins URL: http://MasterIP:ExposedPort(8080)

- Jenkins Tunnel: MasterIP:ExposedPort(50000)  *No http://, No / at the end

3. Add Pod Template

- Name: jenkins-slave
- Label: jenkins-slave
- Usage: Use this node as much as possible
- Add Container Template
  - Name: jenkins-slave
  - Docker image: jenkinsci/jnlp-slave
  - Working Directory: /home/jenkins
  - Add Volume (Host path volume) \*\*Allow Slave to run docker and kubectl
    - key: /usr/bin/docker
    - value: /usr/bin/docker
    - key: /var/run/docker.sock
    - value: /var/run/docker.sock
    - key: /usr/bin/kubectl
    - value: /usr/bin/kubectl
- Service Account: jenkins

(Optional, set jenkins slave to be ran on master)
- Set Node Selector

    `kubernetes.io/hostname=$MasterNodeName`

4. Edit permission for slave

   `sudo chmod 666 /var/run/docker.sock`

## Jenkins Script

> Put Node('jenkins-slave)

    node('jenkins-slave') {
        stage('Test') {
        echo "2.Test Stage"
        }
    }
