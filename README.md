# Create self signed certificate

### create domains list in a file

```txt
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = home.com
DNS.2 = gitlab.home.com
DNS.3 = registry.gitlab.home.com
```

### create keys using this file

```sh
# creates RootCA.key and RootCA.pem
openssl req -x509 -nodes -new -sha256 -days 1024 -newkey rsa:2048 -keyout RootCA.key -out RootCA.pem -subj "/C=US/CN=Example-Root-CA"
# creates RootCA.crt from RootCA.pem
openssl x509 -outform pem -in RootCA.pem -out RootCA.crt
# creates localhost.csr and localhost.key
openssl req -new -nodes -newkey rsa:2048 -keyout localhost.key -out localhost.csr -subj "/C=US/ST=YourState/L=YourCity/O=Example-Certificates/CN=localhost.local"
# creates localhost.crt and RootCA.srl
openssl x509 -req -sha256 -days 1024 -in localhost.csr -CA RootCA.pem -CAkey RootCA.key -CAcreateserial -extfile domains.ext -out localhost.crt
```

NOTE: localhost.crt file should be present on all systems which will connect to other systems via https otherwise you will see X.509 certificate error.

# Install Docker-ce

```sh
sudo apt update
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
sudo apt update
sudo apt install -y docker-ce
sudo usermod -aG docker ${USER}
# exit and loging into shell again below command creates new shell, on exit you will fall into old shell where docker will not work without sudo. So best option just re-login into shell
su - ${USER}
```

# Install Gitlab-ee

```sh
sudo apt-get update
sudo apt-get install -y curl openssh-server ca-certificates tzdata perl
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.deb.sh | sudo bash
sudo EXTERNAL_URL="http://gitlab.home.com" apt-get install gitlab-ee
```

Convert to support https

```sh
sudo mkdir -p /etc/gitlab/ssl
sudo cp localhost.crt /etc/gitlab/ssl/gitlab.home.com.crt
sudo cp localhost.crt /etc/gitlab/ssl/registry.gitlab.home.com.crt
sudo cp localhost.key /etc/gitlab/ssl/gitlab.home.com.key
sudo cp localhost.key /etc/gitlab/ssl/registry.gitlab.home.com.key
```

Edit file: sudo nano /etc/gitlab/gitlab.rb

```sh
external_url 'https://gitlab.home.com'
letsencrypt['enable'] = false
registry_external_url 'https://registry.gitlab.home.com'
```

```sh
sudo gitlab-ctl reconfigure
```

### Issues:

Incase gitlab is not listening on https, check the logs for nginx to see any certificate related issues.

```sh
sudo gitlab-ctl tail nginx
```

# Install Gitlab Runner

For ubuntu/debian via apt repository
```sh
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt update
sudo apt-get install gitlab-runner
```

If apt repository is not available then following method can be used:
```sh
curl -LJO "https://gitlab-runner-downloads.s3.amazonaws.com/latest/deb/gitlab-runner_amd64.deb"
sudo dpkg -i gitlab-runner_amd64.deb
```

```sh
sudo mkdir -p /etc/gitlab-runner/certs
sudo cp localhost.crt /etc/gitlab-runner/certs/gitlab.home.com.crt
sudo cp localhost.crt /etc/gitlab-runner/certs/registry.gitlab.home.com.crt
sudo cp localhost.key /etc/gitlab-runner/certs/gitlab.home.com.key
sudo cp localhost.key /etc/gitlab-runner/certs/registry.gitlab.home.com.key
```

For Docker to connect to self signed gitlab server, we have to add the certificate in following location:

```sh
sudo mkdir -p /etc/docker/certs.d/registry.gitlab.home.com
sudo cp localhost.crt /etc/docker/certs.d/registry.gitlab.home.com/ca.crt
```

# Register Gitlab Runner

> ref: https://docs.gitlab.com/runner/commands/README.html#gitlab-runner-register

```sh
sudo gitlab-runner register -n \
--url https://gitlab.home.com \
--registration-token aKBP1WqdW-k5_VsCDbbr \
--executor docker --description "gitlab Runner" \
--tag-list build,test,deploy \
--docker-privileged \
--docker-image "docker:latest" \
--docker-volumes /var/run/docker.sock:/var/run/docker.sock \
--docker-volumes /etc/docker/certs.d:/etc/docker/certs.d:ro \
--docker-pull-policy="if-not-present"
```

# Connect local Kubernetes with Gitlab

This is for Kubernetes with http as of now.

### Enable local network query

```txt
Log in to gitlab with the admin account,
Click "settings" -> "network" -> "Outbound requests"
Check the box labeled "Allow requests to the local network from web hooks and services"
Click "Save changes"
```

### create gitlab admin service account

> ref: https://docs.gitlab.com/ee/user/project/clusters/add_remove_clusters.html#existing-kubernetes-cluster

```sh
kubectl apply -f gitlab-admin-service-account.yaml
```

- Kubernetes cluster name (required) - The name you wish to give the cluster.
- Environment scope (required) - The associated environment to this cluster.
- API URL (required):
  Get the API URL by running this command:

```sh
kubectl cluster-info | grep -E 'Kubernetes master|Kubernetes control plane' | awk '/http/ {print $NF}'
```

- CA certificate (required):
  List the secrets with kubectl get secrets, and one should be named similar to default-token-xxxxx. Copy that token name for use below.

```sh
kubectl get secrets
```

```sh
kubectl get secret <secret name> -o jsonpath="{['data']['ca\.crt']}" | base64 --decode
```

- Token (required):
  Apply the service account and cluster role binding to your cluster:

```sh
kubectl apply -f gitlab-admin-service-account.yaml
```

Retrieve the token for the gitlab service account:

```sh
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep gitlab | awk '{print $1}')
```

Copy the <authentication_token> value from the output:

```
...
Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      <authentication_token>
```

- GitLab-managed cluster - Leave this checked if you want GitLab to manage namespaces and service accounts for this cluster.
- Project namespace (optional) - You don’t have to fill it in; by leaving it blank, GitLab creates one for you.
- Finally, click the Create Kubernetes cluster button.

After a couple of minutes, your cluster is ready. You can now proceed to install some pre-defined applications.

# Install NFS provisioner(Without helm)

> ref: https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner

Install NFS kernel server

```sh
sudo apt install nfs-kernel-server

sudo mkdir -p /srv/nfs/kubedata
sudo chown nobody: /srv/nfs/kubedata/

# append to /etc/exportfs
sudo echo "/srv/nfs/kubedata	*(rw,sync,no_subtree_check,no_root_squash,no_all_squash,insecure)" >> /etc/exportfs

# to export
sudo exportfs -rav

# to check
sudo exportfs -v
```

Replace NFS server IP in deployment yaml <server_ip>

Replace NFS server path in deployment yaml <server_path>

You can use following sed command to do it:

```sh
SERVER_IP="192.168.X.X"
SERVER_PATH="/srv/nfs/kubedata"
sed -i -e "s|<server_ip>|$SERVER_IP|g" -e "s|<server_path>|$SERVER_PATH|g" nfs-storage/deployment.yaml
```

Create authorization for NFS Provisioner

```sh
# check there is not rbac for nfs
kubectl get clusterrole,clusterrolebinding,role,rolebinding | grep nfs

# create rbac
kubectl create -f nfs-storage/rbac.yaml
```

Create NFS provisioner

```sh
kubectl create -f nfs-storage/deployment.yaml
```

Create managed storage class:

```sh
kubectl create -f nfs-storage/class.yaml
```

Create default storage class:

```sh
kubectl create -f nfs-storage/default-storage-class.yaml
```

# Create metallb loadbalancer

> ref: https://metallb.universe.tf/installation/#installation-by-manifest

> ref: https://metallb.universe.tf/configuration/#layer-2-configuration

```sh
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.6/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.6/manifests/metallb.yaml
# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

define config map to complete, update IP range

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.240-192.168.1.250
```

# Metric Server

> ref: https://github.com/kubernetes-sigs/metrics-server

The file is not used as it is as we are using self signed certificates. Hence following section is added to the file on line 137-139 to enable insecure tls connection.

```yaml
command:
	- /metrics-server
	- --kubelet-insecure-tls
```

```sh
kubectl create -f metric-server/gitlab-metric-server.yaml
```

# Create ingress-nginx

> ref: https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.45.0/deploy/static/provider/baremetal/deploy.yaml

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.45.0/deploy/static/provider/baremetal/deploy.yaml

or

kubectl apply -f ingress-nginx/deploy.yaml
```
