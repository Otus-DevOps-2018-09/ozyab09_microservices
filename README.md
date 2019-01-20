# ozyab09_microservices
ozyab09 microservices repository


### Homework 21 (kubernetes-1)
[![Build Status](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices.svg?branch=kubernetes-1)](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices)

* Creted new Deploement manifests in <i>kubernetes/reddit</i> folder:
```
comment-deployment.yml
mongo-deployment.yml
post-deployment.yml
ui-deployment.yml
```

### [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

This lab assumes you have access to the <i>Google Cloud Platform</i>. This lab we use <i>MacOS</i>.

#### Prerequisites

* Install the <i>Google Cloud SDK</i>

Follow the <i>Google Cloud SDK</i> [documentation](https://cloud.google.com/sdk/) to install and configure the gcloud command line utility.

Verify the Google Cloud SDK version is 218.0.0 or higher: `gcloud version`

* Default Compute Region and Zone
The easiest way to set default compute region: `gcloud init`.

Otherwise set a default compute region: `gcloud config set compute/region us-west1`.

Set a default compute zone: `gcloud config set compute/zone us-west1-c`.

#### Installing the Client Tools

* Install CFSSL

The <i>cfssl</i> and <i>cfssljson</i> command line utilities will be used to provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) and generate TLS certificates.

Installing <i>cfssl</i> and <i>cfssljson</i> using packet manager <i>brew</i>: `brew install cfssl`.

* Verification Installing
```
cfssl version
```
* Install </i>kubectl</i>

The <i>kubectl</i> command line utility is used to interact with the Kubernetes API Server.

* Download and install <i>kubectl</i> from the official release binaries:
```
curl -o kubectl https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/darwin/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```
* Verify <i>kubectl</i> version 1.12.0 or higher is installed:
```
kubectl version --client
```

#### Provisioning Compute Resources

* Virtual Private Cloud Network
Create the <i>kubernetes-the-hard-way</i> custom VPC network:
```
gcloud compute networks create kubernetes-the-hard-way --subnet-mode custom
```
A subnet must be provisioned with an IP address range large enough to assign a private IP address to each node in the <i>Kubernetes</i> cluster.

Create the <i>kubernetes</i> subnet in the <i>kubernetes-the-hard-way</i> VPC network:
```
gcloud compute networks subnets create kubernetes \
  --network kubernetes-the-hard-way \
  --range 10.240.0.0/24
```
The <i>10.240.0.0/24</i> IP address range can host up to 254 compute instances.

* Firewall

Create a firewall rule that allows internal communication across all protocols:
```
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-internal \
  --allow tcp,udp,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 10.240.0.0/24,10.200.0.0/16
```
Create a firewall rule that allows external SSH, ICMP, and HTTPS:
```
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-external \
  --allow tcp:22,tcp:6443,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 0.0.0.0/0
```
List the firewall rules in the <i>kubernetes-the-hard-way</i> VPC network:
```
gcloud compute firewall-rules list --filter="network:kubernetes-the-hard-way"
```
> output
```
NAME                                    NETWORK                  DIRECTION  PRIORITY  ALLOW                 DENY  DISABLED
kubernetes-the-hard-way-allow-external  kubernetes-the-hard-way  INGRESS    1000      tcp:22,tcp:6443,icmp        False
kubernetes-the-hard-way-allow-internal  kubernetes-the-hard-way  INGRESS    1000      tcp,udp,icmp                False
```
* Kubernetes Public IP Address

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:
```
gcloud compute addresses create kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region)
```
Verify the <i>kubernetes-the-hard-way</i> static IP address was created in your default compute region:
```
gcloud compute addresses list --filter="name=('kubernetes-the-hard-way')"
```
* Compute Instances
The compute instances in this lab will be provisioned using Ubuntu Server 18.04, which has good support for the containerd container runtime. Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

* Kubernetes Controllers
Create three compute instances which will host the Kubernetes control plane:
```
for i in 0 1 2; do
  gcloud compute instances create controller-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --private-network-ip 10.240.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,controller
done
```
* Kubernetes Workers
Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The <i>pod-cidr</i> instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to <i>10.200.0.0/16</i>, which supports 254 subnets.

Create three compute instances which will host the <i>Kubernetes</i> worker nodes:
```
for i in 0 1 2; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,worker
done
```
* Verification
List the compute instances in your default compute zone:
```
gcloud compute instances list
```
> output
```
NAME          ZONE            MACHINE_TYPE               PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
controller-0  europe-west4-a  n1-standard-1                           10.240.0.10  X.X.X.X         RUNNING
controller-1  europe-west4-a  n1-standard-1                           10.240.0.11  X.X.X.X         RUNNING
controller-2  europe-west4-a  n1-standard-1                           10.240.0.12  X.X.X.X         RUNNING
worker-0      europe-west4-a  n1-standard-1                           10.240.0.20  X.X.X.X         RUNNING
worker-1      europe-west4-a  n1-standard-1                           10.240.0.21  X.X.X.X         RUNNING
worker-2      europe-west4-a  n1-standard-1                           10.240.0.22  X.X.X.X         RUNNING
```

* Configuring SSH Access
SSH will be used to configure the controller and worker instances. When connecting to compute instances for the first time SSH keys will be generated for you and stored in the project or instance metadata as describe in the [connecting to instances](https://cloud.google.com/compute/docs/instances/connecting-to-instance) documentation.

Test SSH access to the <i>controller-0</i> compute instances:
```
gcloud compute ssh controller-0
```
If this is your first time connecting to a compute instance SSH keys will be generated for you.

#### Provisioning a CA and Generating TLS Certificates

* Certificate Authority

Generate the CA configuration file, certificate, and private key:

```json
{
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

}
```

* Client and Server Certificates

Generate the admin client certificate and private key:

```json
{
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
}
```

* The Kubelet Client Certificates

Kubernetes uses a [special-purpose authorization mode](https://kubernetes.io/docs/admin/authorization/node/) called Node Authorizer, that specifically authorizes API requests made by [Kubelets](https://kubernetes.io/docs/concepts/overview/components/#kubelet). 

Generate a certificate and private key for each Kubernetes worker node:
```json
for instance in worker-0 worker-1 worker-2; do
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

EXTERNAL_IP=$(gcloud compute instances describe ${instance} \
  --format 'value(networkInterfaces[0].accessConfigs[0].natIP)')

INTERNAL_IP=$(gcloud compute instances describe ${instance} \
  --format 'value(networkInterfaces[0].networkIP)')

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${EXTERNAL_IP},${INTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done
```

* The Controller Manager Client Certificate

Generate the <i>kube-controller-manager</i> client certificate and private key:
```json
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

```

* The Kube Proxy Client Certificate

Generate the <i>kube-proxy</i> client certificate and private key:
```json
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

* The Scheduler Client Certificate

Generate the <i>kube-scheduler</i> client certificate and private key:
```json
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```

* The Kubernetes API Server Certificate


The kubernetes-the-hard-way static IP address will be included in the list of subject alternative names for the Kubernetes API Server certificate. This will ensure the certificate can be validated by remote clients.

Generate the Kubernetes API Server certificate and private key:

```json
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,kubernetes.default \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

* The Service Account Key Pair

The Kubernetes Controller Manager leverages a key pair to generate and sign service account tokens as describe in the [managing service accounts](https://kubernetes.io/docs/admin/service-accounts-admin/) documentation.

Generate the <i>service-account</i> certificate and private key:
```json
cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
```

* Distribute the Client and Server Certificates

Copy the appropriate certificates and private keys to each worker instance:
```
for instance in worker-0 worker-1 worker-2; do
  gcloud compute scp ca.pem ${instance}-key.pem ${instance}.pem ${instance}:~/
done
```
Copy the appropriate certificates and private keys to each controller instance:
```
for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem ${instance}:~/
done
```

#### Generating Kubernetes Configuration Files for Authentication
In this lab you will generate [Kubernetes configuration files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/), also known as kubeconfigs, which enable Kubernetes clients to locate and authenticate to the Kubernetes API Servers.

In this section you will generate kubeconfig files for the <i>controller manager</i>, <i>kubelet</i>, <i>kube-proxy</i>, <i>and scheduler</i> clients and the <i>admin</i> user.

* Kubernetes Public IP Address
Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the IP address assigned to the external load balancer fronting the Kubernetes API Servers will be used.

Retrieve the <i>kubernetes-the-hard-way</i> static IP address:

```json
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')
```

* The kubelet Kubernetes Configuration File

When generating kubeconfig files for Kubelets the client certificate matching the Kubelet's node name must be used. This will ensure Kubelets are properly authorized by the Kubernetes [Node Authorizer](https://kubernetes.io/docs/admin/authorization/node/).

Generate a kubeconfig file for each worker node:

```
for instance in worker-0 worker-1 worker-2; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```

* The kube-proxy Kubernetes Configuration File

Generate a kubeconfig file for the <i>kube-proxy</i> service:

```json
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

* The kube-controller-manager Kubernetes Configuration File

Generate a kubeconfig file for the <i>kube-controller-manager</i> service:
```
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```

* The kube-scheduler Kubernetes Configuration File

Generate a kubeconfig file for the <i>kube-scheduler</i> service:
```json
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```

* The admin Kubernetes Configuration File

Generate a kubeconfig file for the <i>admin</i> user:
```json
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
```

* Distribute the Kubernetes Configuration Files
Copy the appropriate <i>kubelet</i> and <i>kube-proxy</i> kubeconfig files to each worker instance:
```json
for instance in worker-0 worker-1 worker-2; do
  gcloud compute scp ${instance}.kubeconfig kube-proxy.kubeconfig ${instance}:~/
done
```
Copy the appropriate <i>kube-controller-manager</i> and <i>kube-scheduler</i> kubeconfig files to each controller instance:
```json
for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ${instance}:~/
done
```

#### Generating the Data Encryption Config and Key
Kubernetes stores a variety of data including cluster state, application configurations, and secrets. Kubernetes supports the ability to [encrypt](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data) cluster data at rest.

In this lab you will generate an encryption key and an [encryption config](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration) suitable for encrypting Kubernetes Secrets.

* The Encryption Key

Generate an encryption key:
```
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

* The Encryption Config File
Create the <i>encryption-config.yaml</i> encryption config file:
```yml
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

Copy the encryption-config.yaml encryption config file to each controller instance:
```json
for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp encryption-config.yaml ${instance}:~/
done
```

#### Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/coreos/etcd). In this lab you will bootstrap a three node etcd cluster and configure it for high availability and secure remote access.

* Prerequisites

The commands in this lab must be run on each controller instance: <i>controller-0</i>, <i>controller-1</i>, and <i>controller-2</i>. Login to each controller instance using the <i>gcloud</i> command. Example: `gcloud compute ssh controller-0`

* Bootstrapping an etcd Cluster Member

Download and Install the etcd Binaries from the [coreos/etcd](https://github.com/coreos/etcd) GitHub project:
```
wget -q --show-progress --https-only --timestamping \
  "https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz"
```

Extract and install the <i>etcd</i> server and the <i>etcdctl</i> command line utility:
```
  tar -xvf etcd-v3.3.9-linux-amd64.tar.gz
  sudo mv etcd-v3.3.9-linux-amd64/etcd* /usr/local/bin/
```
* Configure the etcd Server
```
  sudo mkdir -p /etc/etcd /var/lib/etcd
  sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```
The instance internal IP address will be used to serve client requests and communicate with etcd cluster peers. Retrieve the internal IP address for the current compute instance:
```
INTERNAL_IP=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip)
```
Each etcd member must have a unique name within an etcd cluster. Set the etcd name to match the hostname of the current compute instance:
```
ETCD_NAME=$(hostname -s)
```
Create the <i>etcd.service</i> systemd unit file:
```
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://10.240.0.10:2380,controller-1=https://10.240.0.11:2380,controller-2=https://10.240.0.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
* Start the etcd Server
```
  sudo systemctl daemon-reload
  sudo systemctl enable etcd
  sudo systemctl start etcd
```

* Verification

List the etcd cluster members:

```
sudo ETCDCTL_API=3 etcdctl member list \
   --endpoints=https://127.0.0.1:2379 \
   --cacert=/etc/etcd/ca.pem \
   --cert=/etc/etcd/kubernetes.pem \
   --key=/etc/etcd/kubernetes-key.pem
```
> output:
```
3a57933972cb5131, started, controller-2, https://10.240.0.12:2380, https://10.240.0.12:2379
f98dc20bce6225a0, started, controller-0, https://10.240.0.10:2380, https://10.240.0.10:2379
ffed16798470cab5, started, controller-1, https://10.240.0.11:2380, https://10.240.0.11:2379
```








####
####
####
####
####
####
####
####



```










```











```
* Пройти Kubernetes The Hard Way;
* Проверить, что kubectl apply -f проходит по созданным до этого
deployment-ам (ui, post, mongo, comment) и поды запускаются;
* Удалить кластер после прохождения THW;
* Все созданные в ходе прохождения THW файлы (кроме
бинарных) поместить в папку kubernetes/the_hard_way
репозитория (сертификаты и ключи тоже можно коммитить, но
только после удаления кластера).

```



### Homework 20 (logging-1)
[![Build Status](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices.svg?branch=logging-1)](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices)

* Создание <i>docker-machine</i>:
```
docker-machine create --driver google \
  --google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-os-cloud/global/images/family/ubuntu-1604-lts \
  --google-machine-type n1-standard-1 \
  --google-open-port 5601/tcp \
  --google-open-port 9292/tcp \
  --google-open-port 9411/tcp \
logging 
```
* Переключение на созданную <i>docker-machine</i>: `eval $(docker-machine env logging)`
* Узнать ip-адрес: `docker-machine ip logging`

* Новая версия приложения [reddit](https://github.com/express42/reddit/tree/microservices)

* Сборка образов: 
```
for i in ui post-py comment; do cd src/$i; bash docker_build.sh; cd -; done
```
или:
```
/src/ui $ bash docker_build.sh && docker push $USER_NAME/ui
/src/post-py $ bash docker_build.sh && docker push $USER_NAME/post
/src/comment $ bash docker_build.sh && docker push $USER_NAME/comment
```

* Отдельный <i>compose</i>-файл для системы логирования:
```yml
docker/docker-compose-logging.yml

version: '3.5'
services:
  fluentd:
    image: ${USERNAME}/fluentd
    ports:
      - "24224:24224"
      - "24224:24224/udp"
  elasticsearch:
    image: elasticsearch
    expose:
      - 9200
    ports:
      - "9200:9200"
  kibana:
    image: kibana
    ports:
      - "5601:5601"
```
* <i>Fluentd</i> - инструмент для отправки, агрегации и преобразования лог-сообщений:
```yml
logging/fluentd/Dockerfile

FROM fluent/fluentd:v0.12
RUN gem install fluent-plugin-elasticsearch --no-rdoc --no-ri --version 1.9.5
RUN gem install fluent-plugin-grok-parser --no-rdoc --no-ri --version 1.0.0
ADD fluent.conf /fluentd/etc
```
* Файл конфигурации <i>fluentd</i>:
```
logging/fluentd/fluent.conf

<source>
  @type forward #плагин <i>in_forward</i> для приема логов
  port 24224
  bind 0.0.0.0
</source>
<match *.**>
  @type copy #плагин copy
  <store>
    @type elasticsearch #для перенаправления всех входящих логов в elasticseacrh
    host elasticsearch
    port 9200
    logstash_format true
    logstash_prefix fluentd
    logstash_dateformat %Y%m%d
    include_tag_key true
    type_name access_log
    tag_key @log_name
    flush_interval 1s
  </store>
  <store>
    @type stdout #а также в stdout
  </store>
</match>
```

* Сборка образа <i>fluentd</i>: `docker build -t $USER_NAME/fluentd .`

* Просмотр логов <i>post</i> сервиса: `docker-compose logs -f post`

* Драйвер для логирования для сервиса <i>post</i> внутри <i>compose</i>-файла:
```yml
docker/docker-compose.yml

version: '3.5'
services:
  post:
    image: ${USER_NAME}/post
    environment:
      - POST_DATABASE_HOST=post_db
      - POST_DATABASE=posts
    depends_on:
      - post_db
    ports:
      - "5000:5000"
    logging:
      driver: "fluentd"
      options:
        fluentd-address: localhost:24224
        tag: service.post
```

* Запуск инфраструктуры централизованной системы логирования и перезапуск сервисов приложения:
```
docker-compose -f docker-compose-logging.yml up -d
docker-compose down
docker-compose up -d 
```
* <i>Kibana</i> будет доступна по адресу http://logging-ip:5061. Необходимо создать индекс паттерн <i>fluentd-*</i>
* Поле <i>log</i> документа <i>elasticsearch</i> содержит в себе <i>JSON</i>-объект. Необходимо выделить эту информацию в поля, чтобы иметь возможность производить по ним поиск. Это достигается за счет использования фильтров для выделения нужной информации
* Добавление фильтра для парсинга <i>json</i>-логов, приходящих от <i>post</i>-сервиса, в конфиг <i>fluentd</i>:
```
logging/fluentd/fluent.conf

<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>
<filter service.post>
  @type parser
  format json
  key_name log
</filter>
<match *.**>
  @type copy
...
```

* Пересборка образа и перезапуск сервиса <i>fluentd</i>:
```
logging/fluentd $ docker build -t $USER_NAME/fluentd .
docker/ $ docker-compose -f docker-compose-logging.yml up -d fluentd 
```
* По аналогии с <i>post</i>-сервисом необходимо для <i>ui</i>-сервиса определить драйвер для логирования <i>fluentd</i> в <i>compose</i>-файле:
```yml
docker/docker-compose.yml

...
logging:
  driver: "fluentd"
  options:
    fluentd-address: localhost:24224
    tag: service.post
...
```
* Перезапуск <i>ui</i> сервиса:
```
docker-compose stop ui
docker-compose rm ui
docker-compose up -d 
```

* Когда приложение или сервис не пишет структурированные логи, используются регулярные выражения для их парсинга. Выделение информации из лога <i>UI</i>-сервиса в поля:
```
logging/fluentd/fluent.conf

<filter service.ui>
  @type parser
  format /\[(?<time>[^\]]*)\]  (?<level>\S+) (?<user>\S+)[\W]*service=(?<service>\S+)[\W]*event=(?<event>\S+)[\W]*(?:path=(?<path>\S+)[\W]*)?request_id=(?<request_id>\S+)[\W]*(?:remote_addr=(?<remote_addr>\S+)[\W]*)?(?:method= (?<method>\S+)[\W]*)?(?:response_status=(?<response_status>\S+)[\W]*)?(?:message='(?<message>[^\']*)[\W]*)?/
  key_name log
</filter>
```

* Для облегчения задачи парсинга вместо стандартных регулярок можно использовать <i>grok</i>-шаблоны. <i>Grok</i> - это именованные шаблоны регулярных выражений. Можно использовать готовый <i>regexp</i>, сославшись на него как на функцию:
```
docker/fluentd/fluent.conf

...
<filter service.ui>
  @type parser
  format grok
  grok_pattern %{RUBY_LOGGER}
  key_name log
</filter> 
...
```

* Часть логов нужно еще распарсить. Для этого можно использовать несколько <i>Grok</i>-ов по очереди:
```
docker/fluentd/fluent.conf

<filter service.ui>
  @type parser
  format grok
  grok_pattern service=%{WORD:service} \| event=%{WORD:event} \| request_id=%{GREEDYDATA:request_id} \| message='%{GREEDYDATA:message}'
  key_name message
  reserve_data true
</filter>

<filter service.ui>
  @type parser
  format grok
  grok_pattern service=%{WORD:service} \| event=%{WORD:event} \| path=%{GREEDYDATA:path} \| request_id=%{GREEDYDATA:request_id} \| remote_addr=%{IP:remote_addr} \| method= %{WORD:method} \| response_status=%{WORD:response_status}
  key_name message
  reserve_data true
</filter>
```

### Homework 19 (monitoring-2)
[![Build Status](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices.svg?branch=monitoring-2)](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices)

* `compose-monitoring.yml` - для мониторинга приложений
Для запуска использовать: `docker-compose -f docker-compose-monitoring.yml up -d`
* [cAdvisor](https://github.com/google/cadvisor) - для наблюдения за состоянием `Docker`-контейнеров (использование `CPU`, памяти, объем сетевого трафика)
Сервис помещен в одну сеть с `Prometheus`, для сбора метрик с `cAdvisor'а`

* В `Prometheus` добавлена информация о новом севрисе:
```yml
- job_name: 'cadvisor'
  static_configs:
    - targets:
      - 'cadvisor:8080' 
```
После внесения изменений необходима пересборка образа:
```
cd monitoring/prometheus
docker build -t $USER_NAME/prometheus .
```
* Запуск сервисов:
```
docker-compose up -d
docker-compose -f docker-compose-monitoring.yml up -d 
```
* Информация из `cAdvisor` будет доступна по адресу http://docker-machine-host-ip:8080
* Данные также собираются в `Prometheus`

* Для визуализации данных следует использовать `Graphana`:
```yml
services:
...
  grafana:
    image: grafana/grafana:5.0.0
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=secret
    depends_on:
      - prometheus
    ports:
      - 3000:3000
volumes:
  grafana_data:
```
* Запуск: `docker-compose -f docker-compose-monitoring.yml up -d grafana`

* `Grapahana` доступна по адресу: http://docker-mahine-host-ip:3000 

* Настройка источника данных в `Graphana`:
```yml
Type: Prometheus
URL: http://prometheus:9090
Access: proxy
```

* Для сбора информации о post сервисе необходимо добавить информацию в файл `prometheus.yml`, чтобы `Prometheus` начал собирать метрики и с него:
```yml
scrape_configs:
...
  - job_name: 'post'
  static_configs:
   - targets:
     - 'post:5000'
```
* Пересборка образа:
```
export USER_NAME=username
docker build -t $USER_NAME/prometheus .
```
* Пересоздание `Docker` инфраструктуры мониторинга:
```
docker-compose -f docker-compose-monitoring.yml down
docker-compose -f docker-compose-monitoring.yml up -d 
```
* Загруженные файл дашбоардов расположены в директории `monitoring/grafana/dashboards/`

* `Alertmanager` - дополнительный компонент для системы мониторинга `Prometheus`
* Сборка образа для `alertmanager`'а - файл `monitoring/alertmanager/Dockerfile`:
```yml
FROM prom/alertmanager:v0.14.0
ADD config.yml /etc/alertmanager/ 
```
* Содержимое файла `config.yml`:
```yml
global:
  slack_api_url: 'https://hooks.slack.com/services/$token/$token/$token'
route:
  receiver: 'slack-notifications'
receivers:
- name: 'slack-notifications'
  slack_configs:
  - channel: '#userchannel'
```

* Сервис алертинга в `docker-compose-monitoring.yml`:
```yml
services:
...
  alertmanager:
    image: ${USER_NAME}/alertmanager
    command:
      - '--config.file=/etc/alertmanager/config.yml'
  ports:
    - 9093:9093 
```

* Файл с условиями, при которых должен срабатывать алерт и посылаться `Alertmanager`'у: `monitoring/prometheus/alerts.yml`
* Простой алерт, который будет срабатывать в ситуации, когда одна из наблюдаемых систем (`endpoint`) недоступна для сбора метрик (в этом случае метрика `up` с лейблом `instance` равным имени данного `endpoint`'а будет равна нулю):
```yml
groups:
  - name: alert.rules
    rules:
    - alert: InstanceDown
      expr: up == 0
      for: 1m
      labels:
        severity: page
      annotations:
        description: '{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute'
        summary: 'Instance {{ $labels.instance }} down'
```
* Файл `alerts.yml` также должен быть скопирован в сервис `prometheus`:
```yml
ADD alerts.yml /etc/prometheus/
```
* Пересборка образа `prometheus`:
```
docker build -t $USER_NAME/prometheus .
```
* Пересоздание `Docker` инфраструктуры мониторинга:
```
docker-compose down -d
docker-compose -f docker-compose-monitoring.yml down
docker-compose up -d
docker-compose -f docker-compose-monitoring.yml up -d 
```
* Пуш всех образов в `dockerhub`:
```
docker login
docker push $USER_NAME/ui
docker push $USER_NAME/comment
docker push $USER_NAME/post
docker push $USER_NAME/prometheus
docker push $USER_NAME/alertmanager 
```
Ссылка на собранные образы на [DockerHub](https://hub.docker.com/u/ozyab)



### Homework 18 (monitoring-1)
[![Build Status](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices.svg?branch=monitoring-1)](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices)

* Правило фаервола для <i>Prometheus</i> и <i>Puma</i>:
```yml
gcloud compute firewall-rules create prometheus-default --allow tcp:9090
gcloud compute firewall-rules create puma-default --allow tcp:9292 
```

* Создание <i>docker-host</i>:
```
export GOOGLE_PROJECT=_project_id_

docker-machine create --driver google \
    --google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-os-cloud/global/images/family/ubuntu-1604-lts \
    --google-machine-type n1-standard-1 \
    --google-zone europe-west1-b \
    docker-host
```

* Переключение на <i>docker-host</i>: 
```
eval $(docker-machine env docker-host)
```

* Запуск <i>Prometheus</i> из готового образа с <i>DockerHub</i>:

```
docker run --rm -p 9090:9090 -d --name prometheus  prom/prometheus
```
<i>Prometheus</i> будет запущен по адресу http://docker-host-ip:9090/
Узнать docker-host-ip можно командой: `docker-machine ip docker-host`

* Остановка контейнера: `docker stop prometheus`

* Файл <i>monitoring/prometheus/Dockerfile</i>:
```yml
FROM prom/prometheus:v2.1.0
ADD prometheus.yml /etc/prometheus/
```
Файл <i>monitoring/prometheus/prometheus.yml</i>:
```yml
---
global:
  scrape_interval: '5s' #частота сбора метрик
scrape_configs:
  - job_name: 'prometheus'  #джобы
    static_configs:
      - targets:
        - 'localhost:9090' #адреса для сбора метрик
  - job_name: 'ui'
    static_configs:
      - targets:
        - 'ui:9292'
  - job_name: 'comment'
    static_configs:
      - targets:
        - 'comment:9292'
```

* Сборка <i>docker-образа</i>:
```
export USER_NAME=username
docker build -t $USER_NAME/prometheus .
```
* Сборка образов при помощи скриптов <i>docker_build.sh</i> в директории каждого сервиса:
```
cd src/ui & ./docker_build.sh
cd src/post-py & ./docker_build.sh
cd src/comment & ./docker_build.sh
```

* Добавление сервиса <i>Prometheus</i> в <i>docker/Dockerfile</i>:
```yml
  prometheus:
    image: ${USERNAME}/prometheus
    ports:
      - '9090:9090'
    volumes:
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention=1d'
    networks:
      - back_net
      - front_net
```
* Запуск микросервисов:
```
docker-compose up -d 
```

* [Node exporter](https://github.com/prometheus/node_exporter) для <i>docker-host</i>:
```yml
services:
...
  node-exporter:
    image: prom/node-exporter:v0.15.2
    user: root
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.ignored-mount-points="^/(sys|proc|dev|host|etc)($$|/)"'
```
<i>Job</i> для <i>Prometheus (prometheus.yml)</i>:
```yml
  - job_name: 'node'
    static_configs:
      - targets:
        - 'node-exporter:9100'
```

* Пересоздание образов:
```
cd /monitoring/prometheus && docker build -t $USER_NAME/prometheus .
```
Пересоздание сервисов:
```
docker-compose down
docker-compose up -d 
```
* Push образов на DockerHub:
```
docker login
docker push $USER_NAME/ui:1.0
docker push $USER_NAME/comment:1.0
docker push $USER_NAME/post:1.0
docker push $USER_NAME/prometheus
```
Ссылка на собранные образы на [DockerHub](https://hub.docker.com/u/ozyab)


### Homework 17 (gitlab-ci-2)
[![Build Status](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices.svg?branch=gitlab-ci-2)](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices)

* Создание нового проекта <i>example2</i>

* Добавление проекта в <i>username_microservices</i>
```
git checkout -b gitlab-ci-2
git remote add gitlab2 http://vm-ip/homework/example2.git
git push gitlab2 gitlab-ci-2
```

* <b>Dev-окружение</b>: Изменение пайплайна таким образом, чтобы <i>job deploy</i> стал определением окружения <i>dev</i>, на которое условно будет выкатываться каждое изменение в коде проекта:
1. Переименуем <i>deploy stage</i> в <i>review</i>
2. <i>deploy_job</i> заменим на <i>deploy_dev_job</i>
3. Добавим <i>environment</i>
```yaml
name: dev
url: http://dev.example.com
```
В разделе <i>Operations</i> - <i>Environment</i> появится окружение <i>dev</i>

* Два новых этапа: <i>stage</i> и <i>production</i>. <i>Stage</i> будет содержать <i>job</i>, имитирующий выкатку на <i>staging</i> окружение, <i>production</i> - на <i>production</i> окружение. <i>Job</i> будут запускаться с кнопки

* Директива <i>only</i> описывает список условий, которые должны быть истинны, чтобы <i>job</i> мог запуститься. Регулярное выражение  `/^\d+\.\d+\.\d+/` означает, что должен стоять <i>semver</i> тэг в <i>git<i>, например, <i>2.4.10</i>

* Пометка текущего коммита тэгом:
```
git tag 2.4.10
```

* Пуш с тэгами:
```
git push gitlab2 gitlab-ci-2 --tags
```

* Динамические окружения позволяет вам иметь выделенный стенд для каждой <i>feature</i>-ветки в <i>git</i>. Определяются динамические окружения с
помощью переменных, доступных в <i>.gitlab-ci.yml</i>.
<i>Job</i> определяет динамическое окружение для каждой ветки в репозитории, кроме ветки <i>master</i>

```yml
branch review:
  stage: review
  script: echo "Deploy to $CI_ENVIRONMENT_SLUG"
  environment:
    name: branch/$CI_COMMIT_REF_NAME
    url: http://$CI_ENVIRONMENT_SLUG.example.com
  only:
    - branches
  except:
    - master
```



### Homework 16 (gitlab-ci-1)
[![Build Status](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices.svg?branch=gitlab-ci-1)](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices)

* Установка <i>Docker</i>:
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository "deb https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
apt-get update
apt-get install docker-ce docker-compose
```

* Подготовка окружения:
```
mkdir -p /srv/gitlab/config /srv/gitlab/data /srv/gitlab/logs
cd /srv/gitlab/
touch docker-compose.yml
```
<i>docker-compose.yml</i>:
```
web:
  image: 'gitlab/gitlab-ce:latest'
  restart: always
  hostname: 'gitlab.example.com'
  environment:
    GITLAB_OMNIBUS_CONFIG: |
      external_url 'http://<VM-IP>'
  ports:
    - '80:80'
    - '443:443'
    - '2222:22'
  volumes:
    - '/srv/gitlab/config:/etc/gitlab'
    - '/srv/gitlab/logs:/var/log/gitlab'
    - '/srv/gitlab/data:/var/opt/gitlab'
```
* Запуск <i>Gitlab CI</i>: `docker-compose up -d`

* <i>GUI GitLab</i>: Отключение регистрации, создание группы проектов <i>homework</i>, создание проекта <i>example</i>

* Добавление <i>remote</i> в проект <i>microservices</i>:
`git remote add gitlab http://<ip>/homework/example.git`

* <i>Push</i> в репозиторий:
`http://35.204.52.154/homework/example`

* Определение <i>CI/CD Pipeline</i> проекта производится в файле <i>.gitlab-ci.yml</i>:
```
stages:
  - build
  - test
  - deploy

build_job:
  stage: build
  script:
    - echo 'Building'

test_unit_job:
  stage: test
  script:
    - echo 'Testing 1'

test_integration_job:
  stage: test
  script:
    - echo 'Testing 2'

deploy_job:
  stage: deploy
  script:
    - echo 'Deploy'
```

* Установка GitLab Runner:
```
docker run -d --name gitlab-runner --restart always \
-v /srv/gitlab-runner/config:/etc/gitlab-runner \
-v /var/run/docker.sock:/var/run/docker.sock \
gitlab/gitlab-runner:latest
```

* Запуск <i>runner</i>'а:
`docker exec -it gitlab-runner gitlab-runner register`

* Добавление исходного кода в репозиторий:
```
git clone https://github.com/express42/reddit.git && rm -rf ./reddit/.git
git add reddit/
git commit -m 'Add reddit app'
git push gitlab gitlab-ci-1
```

* Изменение описания пайплайна в <i>.gitlab-ci.yml</i>:
```
image: ruby:2.4.2
stages:
...
variables:
  DATABASE_URL: 'mongodb://mongo/user_posts'
before_script:
  - cd reddit
  - bundle install
...
test_unit_job:
  stage: test
  services:
    - mongo:latest
  script:
    - ruby simpletest.rb
...
```

* В пайплайне выше добавлен вызов <i>reddit/simpletest.rb</i>:
```
require_relative './app'
require 'test/unit'
require 'rack/test'

set :environment, :test

class MyAppTest < Test::Unit::TestCase
  include Rack::Test::Methods

  def app
    Sinatra::Application
  end

  def test_get_request
    get '/'
    assert last_response.ok?
  end
end
```

* Добавление библиотеки для тестирования в <i>reddit/Gemfile</i>: `gem 'rack-test'`

### Homework 15 (docker-4)
[![Build Status](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices.svg?branch=docker-4)](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices)

* Подключение к `docker-host`:
`eval $(docker-machine env docker-host)`

* Запуск контейнера `joffotron/docker-net-tools` с набором сетевых утилит:
`docker run -ti --rm --network none joffotron/docker-net-tools -c ifconfig`

Использован `none-driver`, вывод работы контейнера:
```
lo Link encap:Local Loopback
   inet addr:127.0.0.1  Mask:255.0.0.0
   UP LOOPBACK RUNNING  MTU:65536  Metric:1
   RX packets:0 errors:0 dropped:0 overruns:0 frame:0
   TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
   collisions:0 txqueuelen:1000
   RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```
* Запуск контейнера в сетевом пространстве `docker`-хоста:
`docker run -ti --rm --network host joffotron/docker-net-tools -c ifconfig`

Запуск `ipconfig` на `docker-host`'е приведет к аналогичному выводу:
`docker-machine ssh docker-host ifconfig`

* Запуст nginx в фоне в сетевом пространстве docker-host:
`docker run --network host -d nginx`

При повторном выполнении команды получим ошибку:
```
2018/11/30 19:50:53 [emerg] 1#1: bind() to 0.0.0.0:80 failed (98: Address already in use)
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
```
По причине того, что порт 80 уже занят.

* Для просмотра существующих net-namespaces необходимо выполнить на docker-host:
`sudo ln -s /var/run/docker/netns /var/run/netns`
Просмотр:
`sudo ip netns`
- При запуске контейнера с сетью `host` net-namespace один - default.
- При запуске контейнера с сетью `none` в спсике добавится id net-namespace.
Вывод списка `net-namespac`'ов:
```
user@docker-host:~$ sudo ip net
88f8a9be77ca
default
```
Можно выполнить команду в выбранном `net-namespace`:
```
user@docker-host:~$ sudo ip netns exec 88f8a9be77ca ifconfig
lo  Link encap:Local Loopback  
  inet addr:127.0.0.1  Mask:255.0.0.0
  UP LOOPBACK RUNNING  MTU:65536  Metric:1
  RX packets:0 errors:0 dropped:0 overruns:0 frame:0
  TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
  collisions:0 txqueuelen:1000 
  RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

* Создание `bridge`-сети:
`docker network create reddit --driver bridge`

* Запуск проекта `reddit` с использованием `bridge`-сети:
```
docker run -d --network=reddit mongo:latest
docker run -d --network=reddit ozyab/post:1.0
docker run -d --network=reddit ozyab/comment:1.0
docker run -d --network=reddit -p 9292:9292 ozyab/ui:1.0
```
В данной конфигурации `web`-сервис `puma` не сможет подключиться к БД `mongodb`.

Сервисы ссылаются друг на друга по `dns`-именам, прописанным в `ENV`-переменных `Dockerfil`'а. Встроенный `DNS docker`'а ничего не знает об этих именах.

Присвоение контейнерам имен или сетевых алиасов при старте:
```
--name <name> (max 1 имя)
--network-alias <alias-name> (1 или более)
```

* Запуск контейнеров с сетевыми алиасами:
```
docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db mongo:latest
docker run -d --network=reddit --network-alias=post ozyab/post:1.0
docker run -d --network=reddit --network-alias=comment ozyab/comment:1.0
docker run -d --network=reddit -p 9292:9292 ozyab/ui:1.0
```
* Запуск проекта в 2-х `bridge` сетях. Сервис `ui` не имеет доступа к базе данных.

Создание `docker`-сетей:
```
docker network create back_net --subnet=10.0.2.0/24
docker network create front_net --subnet=10.0.1.0/24
```
Запуск контейнеров:
```
docker run -d --network=front_net -p 9292:9292 --name ui ozyab/ui:1.0
docker run -d --network=back_net --name comment ozyab/comment:1.0
docker run -d --network=back_net --name post ozyab/post:1.0
docker run -d --network=back_net --name mongo_db  --network-alias=post_db --network-alias=comment_db mongo:latest 
```
Docker при инициализации контейнера может подключить к нему только 1 сеть, поэтому контейнеры `comment` и `post` не видят контейнер `ui` из соседних сетей. 

Нужно поместить контейнеры post и comment в обе сети. Дополнительные сети подключаются командой: `docker network connect <network> <container>`:
```
docker network connect front_net post
docker network connect front_net comment 
```
Установка пакета `bridge-utils`:
```
docker-machine ssh docker-host
sudo apt-get update && sudo apt-get install bridge-utils
```
Выполнив `docker network ls` можно увидеть список виртуальных сетей `docker`'а.
`ifconfig | grep br` покажет список `bridge`-интерфейсов:
```
br-45935d0f2bbf Link encap:Ethernet  HWaddr 02:42:6d:5a:8b:7e
br-45bbc0c70de1 Link encap:Ethernet  HWaddr 02:42:94:69:ab:35
br-b6342f9c65f2 Link encap:Ethernet  HWaddr 02:42:9a:b1:73:d9
```
Можно просмотреть информацию о каждом `bridge`-интерфейсе командой `brctl show <interface>`:
```
docker-user@docker-host:~$brctl show br-45935d0f2bbf
bridge name      bridge id          STP enabled   interfaces
br-45935d0f2bbf  8000.02426d5a8b7e  no            veth05b2946
                                                  veth2f50985
                                                  vetha882d28
```
_veth-интерфейс_ - часть виртуальной пары интерфейсов, которая лежат в сетевом пространстве хоста и
также отображаются в ifconfig. Вторые часть виртуального интерфейса находится внутри контейнеров.

Просмотр `iptables`: `sudo iptables -nL -t nat`
```
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  10.0.1.0/24          0.0.0.0/0
MASQUERADE  all  --  10.0.2.0/24          0.0.0.0/0
MASQUERADE  tcp  --  10.0.1.2             10.0.1.2             tcp dpt:9292
```
Первые правила отвечают за выпуск трафика из контейнеров наружу.
```
Chain DOCKER (2 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:9292 to:10.0.1.2:9292
```
Последняя строка прокидывает порт 9292 внутрь контейнера.

## Docker-compose
* Файлу `./src/docker-compose.yml` требуется переменная окружения `USERNAME`: `export USERNAME=ozyab`

Можно выполнить: `docker-compose up -d`

* Изменить `docker-compose` под кейс с множеством сетей, сетевых алиасов 
* Файл `.env` - переменные для `docker-compose.yml`
* Базовое имя создается по имени папки, в которой происходит запуск `docker-compose`.

Для задания базового имени проекта необходимо добавить переменную `COMPOSE_PROJECT_NAME=dockermicroservices`

### Homework 14 (docker-3)
[![Build Status](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices.svg?branch=docker-3)](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices)

Работа в папке src:
- `post-py` - сервис отвечающий за написание постов
- `comment` - сервис отвечающий за написание комментариев
- `ui` - веб-интерфейс, работающий с другими сервисами
* Сборка образов:
```
docker build -t ozyab/post:1.0 ./post-py
docker build -t ozyab/comment:1.0 ./comment
docker build -t ozyab/ui:1.0 ./ui
```
* Отдельная bridge-сеть для контейнеров, так как сетевые алиасы не работают в сети по умолчанию:
`docker network create reddit`

* Запуск контейнеров в этой сети с сетевыми алиасами контейнеров:
```
docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db mongo:latest
docker run -d --network=reddit --network-alias=post ozyab/post:1.0
docker run -d --network=reddit --network-alias=comment ozyab/comment:1.0
docker run -d --network=reddit -p 9292:9292 ozyab/ui:1.0
```
* Остановка всех контейнеров:
`docker kill $(docker ps -q)`
* Создание Docker volume:
`docker volume create reddit_db`
* Запуск контейнеров с docker volume:
```
docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db -v reddit_db:/data/db mongo:latest
docker run -d --network=reddit --network-alias=post ozyab/post:1.0
docker run -d --network=reddit --network-alias=comment ozyab/comment:1.0
docker run -d --network=reddit -p 9292:9292 ozyab/ui:2.0
```
* После перезапуска информация остается в базе

### Homework 13 (docker-2)
[![Build Status](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices.svg?branch=docker-2)](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices)

* Работа с `docker-machine`:

`docker-machine create <имя>` - создание docker-хоста

`eval $(docker-machine env <имя>)` - перемеключение на docker-хост

`eval $(docker-machine env --unset)` - переключение на локальный docker

`docker-machine rm <имя>` - удаление docker-хоста

* Создание `docker-host`:
```
docker-machine create --driver google \
  --google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-os-cloud/global/images/family/ubuntu-1604-lts \
  --google-machine-type n1-standard-1 \
  --google-zone europe-west1-b \
  docker-host
```
* После этого увидеть созданный `docker-host` можно, выполнив: `docker-machine ls`
* Запустим `htop` в `docker`'е:
`docker run --rm -ti tehbilly/htop`

Будет виден только процесс `htop`.

Если выполнить `docker run --rm --pid host -ti tehbilly/htop`, то видны будут все процессы на хостовой машине

* Добавлены: 

`Dockerfile` - текстовое описание нашего образа

`mongod.conf` - подготовленный конфиг для `mongodb`

`db_config` - переменная окружения со ссылкой на `mongodb`

`start.sh` - скрипт запуска приложения

* Сборка образа: `docker build -t reddit:latest .`
* Запуск контейнера: `docker run --name reddit -d --network=host reddit:latest`
* Создание правило на входящий порт 9292:
```
gcloud compute firewall-rules create reddit-app \
  --allow tcp:9292 \
  --target-tags=docker-machine \
  --description="Allow PUMA connections" \
  --direction=INGRESS 
```
* Команды по работе с образом:

`docker tag reddit:latest <login>/otus-reddit:1.0` - добавить тэг образу `reddit`

`docker push <login>/otus-reddit:1.0` - отправка образа в registry

`docker logs reddit -f` - просмотр логов

`docker inspect <login>/otus-reddit:1.0` - просмотр информации об образе

`docker inspect <login>/otus-reddit:1.0 -f '{{.ContainerConfig.Cmd}}'` - просмотр только определенной информации о контейнере

`docker diff reddit` - просмотр изменений, произошедних в файловой системе запущенного контейнера



### Homework 12 (docker-1)
[![Build Status](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices.svg?branch=docker-1)](https://travis-ci.com/Otus-DevOps-2018-09/ozyab09_microservices)
* Добавлен файл шаблона PR `.github/PULL_REQUEST_TEMPLATE`
* Интеграция со slack выполняется командой `/github subscribe Otus-DevOps-2018-09/ozyab09_microservices`
* Запуск контейнера: `docker run hello-world`
* Список запущенных контейнеров: `docker ps`
* Список всех контейнеров: `docker ps -a`
* Создание и запуск контейнера: `docker run -it ubuntu:16.04 /bin/bash`
* Команда `run` создает и запускает контейнер из `image`
* `start` запускает остановленный  созданный ранее контейнер
* `attach` подсоединяет терминал к созданному контейнеру
* `docker system df` отображает сколько дискового пространства занято образами, контейнерами и `volume`'ами
* Создание образа из контейнера: `docker commit <container_id> username/ubuntu-tmp-file`
