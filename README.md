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


sudo cp $HOST_NAME.crt /data/cert/
sudo cp $HOST_NAME.key /data/cert/

openssl x509 -inform PEM -in $HOST_NAME.crt -out $HOST_NAME.cert

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

## Copy Self Signed Cert and CA cert from the Harbor VM to your Downloads folder

```bash
export HARBOR_HOST_NAME=$(gcloud compute instances list --filter="$VM_NAME" --format "get(networkInterfaces[0].accessConfigs[0].natIP)").nip.io
gcloud compute scp $VM_NAME:~/$HARBOR_HOST_NAME.crt $VM_NAME:~/ca.crt ~/Downloads/ --zone=$GCP_ZONE

```

## (macOS only) Add the Certs to Keychain
```bash
security add-trusted-cert -r trustRoot -k $HOME/Library/Keychains/login.keychain ~/Downloads/ca.crt
security add-trusted-cert -r trustRoot -k $HOME/Library/Keychains/login.keychain ~/Downloads/$HARBOR_HOST_NAME.crt

```
* Go to Keychain and manually trust the cert for your domain name

### Useful Links Section
- [Harbor docs | Configure HTTPS Access to Harbor](https://goharbor.io/docs/2.1.0/install-config/configure-https/)
- [Self Signed Certificate vs CA Certificate — The Differences Explained](https://sectigostore.com/page/self-signed-certificate-vs-ca/)