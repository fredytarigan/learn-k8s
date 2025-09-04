## Prepare to update system and install some packages

```bash
sudo apt-get update
sudo apt-get -y install wget curl vim openssl git
```

## Configure /etc/hosts for each hosts

```bash
sudo nano /etc/hosts
```

Add (append) lines below to the content of `/etc/hosts`

```bash
10.91.12.54 master-01.kubernetes.local master-01
10.91.12.55 node-01.kubernetes.local node-01
```

Try to ping the host by its hostname

```bash
ping master-01
ping node-01
ping master-01.kubernetes.local
ping node-01.kubernetes.local
```

## Download all required binary

Create a new directory named k8s/1.32.3

```bash
mkdir -p k8s/1.32.3
```

```bash
## Server components
curl -L https://dl.k8s.io/v1.32.3/bin/linux/amd64/kubectl -o kubectl
curl -L https://dl.k8s.io/v1.32.3/bin/linux/amd64/kube-apiserver -o kube-apiserver
curl -L https://dl.k8s.io/v1.32.3/bin/linux/amd64/kube-controller-manager -o kube-controller-manager
curl -L https://dl.k8s.io/v1.32.3/bin/linux/amd64/kube-scheduler -o kube-scheduler

## Node components
curl -L https://dl.k8s.io/v1.32.3/bin/linux/amd64/kube-proxy -o kube-proxy
curl -L https://dl.k8s.io/v1.32.3/bin/linux/amd64/kubelet -o kubelet

## Container components
curl -L https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.32.0/crictl-v1.32.0-linux-amd64.tar.gz -o crictl-v1.32.0-linux-amd64.tar.gz
curl -L https://github.com/opencontainers/runc/releases/download/v1.3.0-rc.1/runc.amd64 -o runc
curl -L https://github.com/containernetworking/plugins/releases/download/v1.6.2/cni-plugins-linux-amd64-v1.6.2.tgz -o cni-plugins-linux-amd64-v1.6.2.tgz
curl -L https://github.com/containerd/containerd/releases/download/v2.1.0-beta.0/containerd-2.1.0-beta.0-linux-amd64.tar.gz -o containerd-2.1.0-beta.0-linux-amd64.tar.gz

## Etcd components
curl -L https://github.com/etcd-io/etcd/releases/download/v3.6.0-rc.3/etcd-v3.6.0-rc.3-linux-amd64.tar.gz -o etcd-v3.6.0-rc.3-linux-amd64.tar.gz
```

Make all binary executeable

```bash
chmod +x kubectl kube-apiserver kube-controller-manager kube-scheduler kubelet kube-proxy runc
```

Try checking kubectl version

```bash
./kubectl version --client
```

## Generate Public Key Infrastructure

Make adjustment to ca.conf file and create dedicated folder to hold the certificates

```bash
mkdir -p certs
```

### Certificate Authority

```bash
openssl genrsa -out ./certs/ca.key 4096
openssl req -x509 -new -sha512 -noenc \
    -key ./certs/ca.key -days 3653 \
    -config ./ca.conf \
    -out ./certs/ca.crt

### results
certs/ca.crt
certs/ca.key
```

### Generate required server certificates

```bash
## kube-apiserver
openssl genrsa -out ./certs/kube-apiserver.key 4096
openssl req -new -key ./certs/kube-apiserver.key -sha256 \
    -config ./ca.conf -section kube-api-server \
    -out ./certs/kube-apiserver.csr
openssl x509 -req -days 3653 -in ./certs/kube-apiserver.csr \
    -copy_extensions copyall \
    -sha256 -CA ./certs/ca.crt \
    -CAkey ./certs/ca.key \
    -CAcreateserial \
    -out ./certs/kube-apiserver.crt

## kube-controller-manager
openssl genrsa -out ./certs/kube-controller-manager.key 4096
openssl req -new -key ./certs/kube-controller-manager.key -sha256 \
    -config ./ca.conf -section kube-controller-manager \
    -out ./certs/kube-controller-manager.csr
openssl x509 -req -days 3653 -in ./certs/kube-controller-manager.csr \
    -copy_extensions copyall \
    -sha256 -CA ./certs/ca.crt \
    -CAkey ./certs/ca.key \
    -CAcreateserial \
    -out ./certs/kube-controller-manager.crt

## kube-scheduler
openssl genrsa -out ./certs/kube-scheduler.key 4096
openssl req -new -key ./certs/kube-scheduler.key -sha256 \
    -config ./ca.conf -section kube-scheduler \
    -out ./certs/kube-scheduler.csr
openssl x509 -req -days 3653 -in ./certs/kube-scheduler.csr \
    -copy_extensions copyall \
    -sha256 -CA ./certs/ca.crt \
    -CAkey ./certs/ca.key \
    -CAcreateserial \
    -out ./certs/kube-scheduler.crt

## service account
openssl genrsa -out ./certs/service-accounts.key 4096
openssl req -new -key ./certs/service-accounts.key -sha256 \
    -config ./ca.conf -section service-accounts \
    -out ./certs/service-accounts.csr
openssl x509 -req -days 3653 -in ./certs/service-accounts.csr \
    -copy_extensions copyall \
    -sha256 -CA ./certs/ca.crt \
    -CAkey ./certs/ca.key \
    -CAcreateserial \
    -out ./certs/service-accounts.crt
```

### Generate required client certificates

```bash
## kubelet
openssl genrsa -out ./certs/node-01.key 4096
openssl req -new -key ./certs/node-01.key -sha256 \
    -config ./ca.conf -section node-01 \
    -out ./certs/node-01.csr
openssl x509 -req -days 3653 -in ./certs/node-01.csr \
    -copy_extensions copyall \
    -sha256 -CA ./certs/ca.crt \
    -CAkey ./certs/ca.key \
    -CAcreateserial \
    -out ./certs/node-01.crt

## kube proxy
openssl genrsa -out ./certs/kube-proxy.key 4096
openssl req -new -key ./certs/kube-proxy.key -sha256 \
    -config ./ca.conf -section kube-proxy \
    -out ./certs/kube-proxy.csr
openssl x509 -req -days 3653 -in ./certs/kube-proxy.csr \
    -copy_extensions copyall \
    -sha256 -CA ./certs/ca.crt \
    -CAkey ./certs/ca.key \
    -CAcreateserial \
    -out ./certs/kube-proxy.crt
```

### Generate required admin access certificates

```bash
openssl genrsa -out ./certs/admin.key 4096
openssl req -new -key ./certs/admin.key -sha256 \
    -config ./ca.conf -section admin \
    -out ./certs/admin.csr
openssl x509 -req -days 3653 -in ./certs/admin.csr \
    -copy_extensions copyall \
    -sha256 -CA ./certs/ca.crt \
    -CAkey ./certs/ca.key \
    -CAcreateserial \
    -out ./certs/admin.crt
```

### Distribute the server certificates

```bash
# replace 'debian' with your ssh username
ssh debian@10.91.12.54 mkdir -p /home/debian/certs
scp ./certs/ca.key ./certs/ca.crt ./certs/kube-apiserver.key ./certs/kube-apiserver.crt ./certs/kube-controller-manager.crt ./certs/kube-controller-manager.key ./certs/kube-scheduler.crt ./certs/kube-scheduler.key ./certs/service-accounts.key ./certs/service-accounts.crt debian@10.91.12.54:/home/debian/certs/
```

### Distribute the client certificates

```bash
# replace 'debian' with your ssh username
ssh debian@10.91.12.55 mkdir -p /home/debian/certs
scp ./certs/ca.crt debian@10.91.12.55:/home/debian/certs/ca.crt
scp ./certs/node-01.crt debian@10.91.12.55:/home/debian/certs/kubelet.crt
scp ./certs/node-01.key debian@10.91.12.55:/home/debian/certs/kubelet.key
scp ./certs/kube-proxy.crt debian@10.91.12.55:/home/debian/certs/kube-proxy.crt
scp ./certs/kube-proxy.key debian@10.91.12.55:/home/debian/certs/kube-proxy.key
```

## Generate Kubernetes Configuration Files

### Run on Master

Generate kubeconfig directory

```bash
mkdir -p kubeconfigs
```

Copy master binary to /usr/local/bin

```bash
sudo cp kubectl /usr/local/bin/kubectl
sudo cp kube-apiserver /usr/local/bin/kube-apiserver
sudo cp kube-controller-manager /usr/local/bin/kube-controller-manager
sudo cp kube-scheduler /usr/local/bin/kube-scheduler
```

#### kube-controller-manager

```bash
./kubectl config set-cluster learn-k8s \
    --certificate-authority=./certs/ca.crt \
    --embed-certs=true \
    --server=https://master-01.kubernetes.local:6443 \
    --kubeconfig=./kubeconfigs/kube-controller-manager.kubeconfig

./kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=./certs/kube-controller-manager.crt \
    --client-key=./certs/kube-controller-manager.key \
    --embed-certs=true \
    --kubeconfig=./kubeconfigs/kube-controller-manager.kubeconfig

./kubectl config set-context default \
    --cluster=learn-k8s \
    --user=system:kube-controller-manager \
    --kubeconfig=./kubeconfigs/kube-controller-manager.kubeconfig

./kubectl config use-context default \
    --kubeconfig=./kubeconfigs/kube-controller-manager.kubeconfig
```

#### kube-scheduler

```bash
./kubectl config set-cluster learn-k8s \
    --certificate-authority=./certs/ca.crt \
    --embed-certs=true \
    --server=https://master-01.kubernetes.local:6443 \
    --kubeconfig=./kubeconfigs/kube-scheduler.kubeconfig

./kubectl config set-credentials system:kube-scheduler \
    --client-certificate=./certs/kube-scheduler.crt \
    --client-key=./certs/kube-scheduler.key \
    --embed-certs=true \
    --kubeconfig=./kubeconfigs/kube-scheduler.kubeconfig

./kubectl config set-context default \
    --cluster=learn-k8s \
    --user=system:kube-scheduler \
    --kubeconfig=./kubeconfigs/kube-scheduler.kubeconfig

./kubectl config use-context default \
    --kubeconfig=./kubeconfigs/kube-scheduler.kubeconfig
```

### Run on Nodes

Generate kubeconfig directory

```bash
mkdir -p kubeconfigs
```

Copy node binary to /usr/local/bin

```bash
sudo cp kubectl /usr/local/bin/kubectl
sudo cp kubelet /usr/local/bin/kubelet
sudo cp kube-proxy /usr/local/bin/kube-proxy
```

### kubelet

```bash
kubectl config set-cluster learn-k8s \
    --certificate-authority=./certs/ca.crt \
    --embed-certs=true \
    --server=https://master-01.kubernetes.local:6443 \
    --kubeconfig=./kubeconfigs/kubelet.kubeconfig

kubectl config set-credentials system:node:node-01 \
    --client-certificate=./certs/kubelet.crt \
    --client-key=./certs/kubelet.key \
    --embed-certs=true \
    --kubeconfig=./kubeconfigs/kubelet.kubeconfig

kubectl config set-context default \
    --cluster=learn-k8s \
    --user=system:node:node-01 \
    --kubeconfig=./kubeconfigs/kubelet.kubeconfig

kubectl config use-context default \
    --kubeconfig=./kubeconfigs/kubelet.kubeconfig
```

### kube-proxy

```bash
./kubectl config set-cluster learn-k8s \
    --certificate-authority=./certs/ca.crt \
    --embed-certs=true \
    --server=https://master-01.kubernetes.local:6443 \
    --kubeconfig=./kubeconfigs/kube-proxy.kubeconfig

./kubectl config set-credentials system:kube-proxy \
    --client-certificate=./certs/kube-proxy.crt \
    --client-key=./certs/kube-proxy.key \
    --embed-certs=true \
    --kubeconfig=./kubeconfigs/kube-proxy.kubeconfig

./kubectl config set-context default \
    --cluster=learn-k8s \
    --user=system:kube-proxy \
    --kubeconfig=./kubeconfigs/kube-proxy.kubeconfig

./kubectl config use-context default \
    --kubeconfig=./kubeconfigs/kube-proxy.kubeconfig
```

### Run on Master or Local

```bash
./kubectl config set-cluster learn-k8s \
    --certificate-authority=./certs/ca.crt \
    --embed-certs=true \
    --server=https://master-01.kubernetes.local:6443 \
    --kubeconfig=./kubeconfigs/admin.kubeconfig

./kubectl config set-credentials admin \
    --client-certificate=./certs/admin.crt \
    --client-key=./certs/admin.key \
    --embed-certs=true \
    --kubeconfig=./kubeconfigs/admin.kubeconfig

./kubectl config set-context default \
    --cluster=learn-k8s \
    --user=admin \
    --kubeconfig=./kubeconfigs/admin.kubeconfig

./kubectl config use-context default \
    --kubeconfig=./kubeconfigs/admin.kubeconfig
```

## Generate Data Encryption Config

Generate an encryption key at master

```bash
head -c 32 /dev/urandom | base64
```

Create a file named 'encryption-config.yaml', and put generated key into file

```bash
mkdir -p configs
touch configs/encryption-config.yaml

### encryption-config.yaml content
---
kind: EncryptionConfiguration
apiVersion: apiserver.config.k8s.io/v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ENCRYPTION_KEY ## replace this with generated key
      - identity: {}
```

## Etcd setup on master

Extract downloaded etcd

```bash
tar -xvf etcd-v3.6.0-rc.3-linux-amd64.tar.gz

## create required directory for etcd
mkdir -p /etc/etcd /var/lib/etcd
chmod 700 /var/lib/etcd

cp ./etcd-v3.6.0-rc.3-linux-amd64/etcd /usr/local/bin
cp ./etcd-v3.6.0-rc.3-linux-amd64/etcdctl /usr/local/bin

chmod +x /usr/local/bin/etcd /usr/local/bin/etcdctl

## create etcd systemd files
sudo nano /etc/systemd/system/etcd.service

## put this into the file content
[Unit]
Description=etcd
Documentation=https://github.com/etcd-io/etcd

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name controller \
  --initial-advertise-peer-urls http://127.0.0.1:2380 \
  --listen-peer-urls http://127.0.0.1:2380 \
  --listen-client-urls http://127.0.0.1:2379 \
  --advertise-client-urls http://127.0.0.1:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster controller=http://127.0.0.1:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

## enable and start etcd service
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl restart etcd

## check if etcd is running
etcdctl member list
```
