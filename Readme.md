# Creating a Highly Available Aarch64/Arm64 etcd Cluster on Raspberry Pi

This tutorial explains how to build and configure a highly available etcd cluster on 64-bit Raspberry Pis. Please refer to the Installing 64-bit HypriotOS to a Raspberry Pi post for an in-depth tutorial on how to run a fully functional 64-bit OS on Raspberry Pi 3/4b+. Please note that etcd on aarch64/arm64 is experimental. I haven't run into any issues with it, but that doesn't mean they don't exist.

## Assumptions
It is assumed that you are running 3 Raspberry Pi 3b+ or 4b+ (recommended) with 64-bit HypriotOS installed. 
The configurations below assume the hostname/ip combination 
`k3s-master01/172.16.11.131`, `k3spi4-master02/172.16.11.132`, and 
`k3spi4-master03/172.16.11.133` for each of the systems.

## Generate SSL Certificates for etcd
### Generate the CA certificate
```bash
# Create private key for CA
openssl genrsa -out ca.key 2048

# Create CSR using the private key
openssl req -new -key ca.key -subj "/CN=CA" -out ca.csr

# Self sign the csr using its own private key
openssl x509 -req -in ca.csr -signkey ca.key -CAcreateserial  -out ca.crt -days 1000
```

### Generate and sign the etcd certificate
Create the etcd openssl configuration file
```bash
# Change the DNS.# and IP.# items accordingly for your environment
cat > openssl-etcd.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = k3spi4-master01
DNS.2 = k3spi4-master02
DNS.3 = k3spi4-master03
IP.1 = 172.16.11.131
IP.2 = 172.16.11.132
IP.3 = 172.16.11.133
IP.4 = 127.0.0.1
EOF
```

```bash
# Create private key for etcd
openssl genrsa -out etcd-server.key 2048

# Create CSR using openssl-etcd.conf and private key
openssl req -new -key etcd-server.key -subj "/CN=etcd-server" -out etcd-server.csr -config openssl-etcd.cnf

# Sign etcd certificate with CA
openssl x509 -req -in etcd-server.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out etcd-server.crt -extensions v3_req -extfile openssl-etcd.cnf -days 1000
```

Distribute etcd certificate to **all other** masters
```bash
scp ca.* etcd-server.* openssl-etcd.cnf 192.168.1.192:~/
scp ca.* etcd-server.* openssl-etcd.cnf 192.168.1.193:~/
```

Create necessary system directories on **_each_** master
```bash
sudo mkdir -p /var/lib/etcd /etc/etcd
```

Install certificates
```bash
sudo cp ca.crt etcd-server.crt etcd-server.key /etc/etcd/
```

# Download the latest etcd binaries to the masters
Navigate to the etcd releases page and copy the arm64 version download link.
```bash
# Download the aarch64/arm64 binaries tarball
wget https://github.com/etcd-io/etcd/releases/download/v3.4.1/etcd-v3.4.1-linux-arm64.tar.gz

# Extract tarball
tar -xvf etcd-v3.4.1-linux-arm64.tar.gz

# Copy binaries to /usr/local/bin/
sudo cp etcd-v3.4.1-linux-arm64/etcd* /usr/local/bin/
```

# Configure the etcd service
Set etcd configuration variables on each of the etcd servers
```bash
# Change the ETCD_NAME# and ETCD_IP# items accordingly for your environment
INTERNAL_IP=$(ip addr show eth0 | grep "inet " | awk '{print $2}' | cut -d / -f1)
ETCD_NAME=$(hostname -s)
ETCD_NAME1=k3s-master-1
ETCD_NAME2=k3s-master-2
ETCD_NAME3=k3s-master-3
ETCD_IP1=192.168.1.191
ETCD_IP2=192.168.1.192
ETCD_IP3=192.168.1.193
```

## Create the etcd.service systemd unit file
```bash
cat | sudo tee /etc/systemd/system/etcd.service <<EOF
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Environment=ETCD_UNSUPPORTED_ARCH=arm64
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/etcd-server.crt \\
  --key-file=/etc/etcd/etcd-server.key \\
  --peer-cert-file=/etc/etcd/etcd-server.crt \\
  --peer-key-file=/etc/etcd/etcd-server.key \\
  --trusted-ca-file=/etc/etcd/ca.crt \\
  --peer-trusted-ca-file=/etc/etcd/ca.crt \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster ${ETCD_NAME1}=https://${ETCD_IP1}:2380,${ETCD_NAME2}=https://${ETCD_IP2}:2380,${ETCD_NAME3}=https://${ETCD_IP3}:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

## Reload systemd daemon
```bash
sudo systemctl daemon-reload
```

## Start etcd on all nodes
```bash
sudo systemctl start etcd.service
```

## Check etcd member list
```bash
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/etcd-server.crt --key=/etc/etcd/etcd-server.key member list
```

## Test replication
```bash
# Put a temporary key into the store at 192.168.1.191
sudo ETCDCTL_API=3 etcdctl --endpoints=https://192.168.1.191:2379 --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/etcd-server.crt --key=/etc/etcd/etcd-server.key put foo bar

# Fetch the temporary key from the store at 192.168.1.192
sudo ETCDCTL_API=3 etcdctl --endpoints=https://192.168.1.192:2379 --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/etcd-server.crt --key=/etc/etcd/etcd-server.key get foo

# Fetch the temporary key from the store at 192.168.1.193
sudo ETCDCTL_API=3 etcdctl --endpoints=https://192.168.1.193:2379 --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/etcd-server.crt --key=/etc/etcd/etcd-server.key get foo

# Delete the temporary key
sudo ETCDCTL_API=3 etcdctl --endpoints=https://192.168.1.193:2379 --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/etcd-server.crt --key=/etc/etcd/etcd-server.key del foo

# Check if the key has been deleted
sudo ETCDCTL_API=3 etcdctl --endpoints=https://192.168.1.192:2379 --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/etcd-server.crt --key=/etc/etcd/etcd-server.key get foo
```

## Check the health of the cluster
```
curl --cacert /etc/etcd/ca.crt --cert /etc/etcd/etcd-server.crt --key /etc/etcd/etcd-server.key https://192.168.1.191:2379/health
```

# Installing K3s Servers
Let’s start by installing the servers in all the nodes where etcd is installed. SSH into the first node, and set the below environment variables. This assumes that you followed the steps explained in the previous tutorial to configure the etcd cluster.

```bash
export K3S_DATASTORE_ENDPOINT='https://192.168.1.191:2379,https://192.168.1.192:2379,https://192.168.1.193:2379'
export K3S_DATASTORE_CAFILE='/etc/etcd/ca.crt'
export K3S_DATASTORE_CERTFILE='/etc/etcd/etcd-server.crt'
export K3S_DATASTORE_KEYFILE='/etc/etcd/etcd-server.key'
```
These environment variables instruct K3s installer to utilize the existing etcd database for state management.

Next, we will populate the K3S_TOKEN with a token that’s used by the agents to join the cluster.
```bash
export K3S_TOKEN="secret_edgecluster_token"
```

We are ready to install the server in the first node. Run the below command to start the process.
```bash
curl -sfL https://get.k3s.io | sh -
```

Repeat these steps in node-2 and node-3 to launch additional servers.

At this point, you have a three-node K3s cluster that runs the control plane and etcd components in a highly available mode.

```bash
sudo kubectl get nodes
```


