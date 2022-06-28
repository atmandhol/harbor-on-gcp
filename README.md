## Harbor on GCP with Self Signed Certs

## Prerequisites
- A Google Cloud Project
- `gcloud` CLI setup on your local and authenticated against that project.

### Create a Ubuntu VM

* Setup Environment (Replace this with your values)
```bash
## Must replace
export GCP_PROJECT=adhol-playground

## May replace
export VM_NAME=harbor-vm
export MACHINE_TYPE=e2-medium
export GCP_ZONE=us-central1-a
```

* Run gcloud command
```bash
gcloud compute instances create $VM_NAME \
--project=$GCP_PROJECT \
--zone=$GCP_ZONE \
--machine-type=$MACHINE_TYPE \
--network-interface=network-tier=PREMIUM,subnet=default \
--maintenance-policy=MIGRATE \
--provisioning-model=STANDARD \
--no-shielded-secure-boot \
--shielded-vtpm \
--shielded-integrity-monitoring \
--reservation-affinity=any \
--tags=https-server \
--scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append \
--create-disk=auto-delete=yes,boot=yes,device-name=$VM_NAME,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20220615,mode=rw,size=100
```

### Bootstrapping

* SSH into the VM
```bash
gcloud compute ssh --zone "$GCP_ZONE" "$VM_NAME"  --project "$GCP_PROJECT"
```

* Install required tools (`docker`, `docker-compose`, `nginx`, `jq`) and make sure `openssl` is installed

```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo apt install docker.io python3-pip nginx jq docker-compose -y
openssl version

```

* Make sure nothing is running on Port 80. Run the following to stop nginx service
```bash
sudo service nginx stop

```

### Download Harbor

```bash
export HARBOR_VERSION=v2.5.1

wget https://github.com/goharbor/harbor/releases/download/$HARBOR_VERSION/harbor-online-installer-$HARBOR_VERSION.tgz

tar xzvf harbor-online-installer-$HARBOR_VERSION.tgz

```

### HTTPS Setup
```bash
export HOST_NAME=$(curl checkip.amazonaws.com).nip.io

# Generate CA Cert private key
openssl genrsa -out ca.key 4096

# Generate the CA certificate.
openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=$HOST_NAME" \
 -key ca.key \
 -out ca.crt

# Check if its good by running 
openssl x509 -in ca.crt -noout -text

# Generate Server certificate
openssl genrsa -out $HOST_NAME.key 4096

# Generate CSR
openssl req -sha512 -new \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=$HOST_NAME" \
    -key $HOST_NAME.key \
    -out $HOST_NAME.csr

# Generate an x509 v3 extension file.
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=$HOST_NAME
EOF

# Use the v3.ext file to generate a certificate for your Harbor host

openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in $HOST_NAME.csr \
    -out $HOST_NAME.crt

sudo mkdir -p /data
sudo mkdir -p /data/cert
sudo cp $HOST_NAME.crt /data/cert/
sudo cp $HOST_NAME.key /data/cert/

openssl x509 -inform PEM -in $HOST_NAME.crt -out $HOST_NAME.cert

sudo mkdir -p /etc/docker/certs.d
sudo mkdir -p /etc/docker/certs.d/$HOST_NAME/
sudo cp $HOST_NAME.cert /etc/docker/certs.d/$HOST_NAME/
sudo cp $HOST_NAME.key /etc/docker/certs.d/$HOST_NAME/
sudo cp ca.crt /etc/docker/certs.d/$HOST_NAME/

sudo systemctl restart docker

sudo cp $HOST_NAME.crt /usr/local/share/ca-certificates/$HOST_NAME.crt 
sudo update-ca-certificates

```

### Update harbor.yml
```bash
cat <<EOF > harbor/harbor.yml
hostname: $HOST_NAME
http:
  port: 80
https:
  port: 443
  certificate: /home/adhol/$HOST_NAME.crt
  private_key: /home/adhol/$HOST_NAME.key

# Change this password to something other than this
harbor_admin_password: Harbor12345

database:
  password: root123
  max_idle_conns: 100
  max_open_conns: 900

data_volume: /data

trivy:
  ignore_unfixed: false
  skip_update: false
  offline_scan: false
  insecure: false

jobservice:
  max_job_workers: 10

notification:
  webhook_job_max_retry: 10

chart:
  absolute_url: disabled

log:
  level: info
  local:
    rotate_count: 50
    rotate_size: 200M
    location: /var/log/harbor
_version: 2.5.0

proxy:
  http_proxy:
  https_proxy:
  no_proxy:
  components:
    - core
    - jobservice
    - trivy

upload_purging:
  enabled: true
  age: 168h
  interval: 24h
  dryrun: false
EOF
```

### Install and Start Harbor

```bash
(cd harbor/ ; sudo ./install.sh)
```

### Copy Self Signed Cert and CA cert from the Harbor VM to your Downloads folder

```bash
export HARBOR_HOST_NAME=$(gcloud compute instances list --filter="$VM_NAME" --format "get(networkInterfaces[0].accessConfigs[0].natIP)").nip.io
gcloud compute scp $VM_NAME:~/$HARBOR_HOST_NAME.crt $VM_NAME:~/ca.crt ~/Downloads/ --zone=$GCP_ZONE

```

### (macOS only) Add the Certs to Keychain
```bash
security add-trusted-cert -r trustRoot -k $HOME/Library/Keychains/login.keychain ~/Downloads/ca.crt
security add-trusted-cert -r trustRoot -k $HOME/Library/Keychains/login.keychain ~/Downloads/$HARBOR_HOST_NAME.crt

```
* Go to Keychain and manually trust the cert for your domain name


## Setup GKE cluster

* Create a GKE cluster from the UI or `gcloud` CLI or using [`tappr`](https://github.com/atmandhol/tappr)

* Create a docker image that has myCA.pem, copies the myCA.pem into /mnt/etc/ssl/certs/, runs update-ca-certificates and restart docker daemon.

```
cat <<EOF > /tmp/insert-ca.sh

#!/bin/bash
cp /myCA.pem /mnt/etc/ssl/certs
nsenter --target 1 --mount update-ca-certificates
nsenter --target 1 --mount bash -c "systemctl is-active --quiet docker && echo 'Restarting docker' && systemctl restart docker"
nsenter --target 1 --mount bash -c "systemctl is-active --quiet containerd && echo 'Restarting containerd' && systemctl restart containerd"
# NOTE: After the CRI restarts subsequent logs may not be visible, however the subsequent commands should
# still have run. You can verify with something like:
# touch /mnt/etc/completed.txt
echo "complete"
EOF

chmod 700 /tmp/insert-ca.sh

cat <<EOF > /tmp/Dockerfile
FROM ubuntu
COPY ~/Downloads/$HARBOR_HOST_NAME.crt /myCA.pem
COPY insert-ca.sh /usr/sbin/

CMD insert-ca.sh
EOF

docker build -t gcr.io/$GCP_PROJECT/custom-cert /tmp/Dockerfile
docker push gcr.io/$GCP_PROJECT/custom-cert

```

* Create a Daemon set

```bash
cat <<EOF | k apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cert-customizations
  labels:
    app: cert-customizations
spec:
  selector:
    matchLabels:
      app: cert-customizations
  template:
    metadata:
      labels:
        app: cert-customizations
    spec:
      hostNetwork: true
      hostPID: true
      initContainers:
      - name: cert-customizations
        image: gcr.io/$GCP_PROJECT/custom-cert
        volumeMounts:
          - name: etc
            mountPath: "/mnt/etc"
        securityContext:
          privileged: true
          capabilities:
            add: ["NET_ADMIN"]
      volumes:
      - name: etc
        hostPath:
          path: /etc
      containers:
      - name: pause
        image: gcr.io/google_containers/pause
EOF
```

The instructions to set certs using Daemonset were taken [from this repo](https://github.com/samos123/gke-node-ca-importer).


## TAP Installation using this registry
Here is a sample tap-values file that will work with self signed registry
```yaml
appliveview_connector:
  backend:
    host: appliveview.127.0.0.1.nip.io
    sslDisabled: 'true'
buildservice:
  descriptor_name: full
  enable_automatic_dependency_updates: true
  kp_default_repository: $HARBOR_HOST_NAME/tap/tbs-ss
  kp_default_repository_secret:
    name: registry-credentials-tbs
    namespace: tap-install
  tanzunet_secret:
    name: tanzunet-registry-creds-tbs
    namespace: tap-install
  ca_cert_data: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----

ceip_policy_disclosed: true
cnrs:
  # Replace this with the domain name
  domain_name: 127.0.0.1.nip.io
contour:
  envoy:
    service:
      type: LoadBalancer
metadata_store:
  app_service_type: NodePort
ootb_supply_chain_basic:
  gitops:
    ssh_secret: ''
  registry:
    repository: tap/supplychain-ss
    server: $HARBOR_HOST_NAME
    ca_cert_data: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
profile: iterate
supply_chain: basic
shared:
  # TODO: use the shared ingress domain
  ca_cert_data: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
```

### Useful Links Section
- [Harbor docs | Configure HTTPS Access to Harbor](https://goharbor.io/docs/2.1.0/install-config/configure-https/)
- [Self Signed Certificate vs CA Certificate — The Differences Explained](https://sectigostore.com/page/self-signed-certificate-vs-ca/)