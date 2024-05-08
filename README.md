# samuel_awx_test


#ref: https://docs.docker.com/engine/install/rhel/
#Remove old version of docker 
sudo yum remove -y docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine \
                  podman \
                  runc
				  
#Set up the repository
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo	  

#Install Docker Engine
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

#Start Docker 
sudo systemctl start docker	  
sudo systemctl enable docker	  


# Add current user into docker group
sudo usermod -aG docker $USER


#ref: https://minikube.sigs.k8s.io/docs/start/
# Install minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64

# Setup docker driver
minikube config set driver docker

# Start Minikube
minikube start --cpus=4 --memory=6g --addons=ingress
minikube kubectl -- get po -A
alias kubectl="minikube kubectl --"

# Start Minikube dashboard
minikube addons enable metrics-server
minikube dashboard

# Install kubectl
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl cluster-info
kubectl version

# Install kustomize
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
sudo mv ./kustomize /usr/local/bin/kustomize

# Check minikube status
minikube status
minikube delete

# https://www.server-world.info/en/note?os=CentOS_Stream_9&p=ansible&f=9
dnf -y install git make

# minikube start
minikube start --cpus=4 --memory=8g --addons=ingress
minikube status
kubectl get pods -A

# Create awx.yaml
echo "---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  service_type: nodeport
  nodeport_port: 30080"  > awx.yaml

# Create kustomization.yaml
echo   "apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # Find the latest tag here: https://github.com/ansible/awx-operator/releases
  - github.com/ansible/awx-operator/config/default?ref=2.16.1
  - awx.yaml

# Set the image tags to match the git version from above
images:
  - name: quay.io/ansible/awx-operator
	newTag: 2.16.1

# Specify a custom namespace in which to install AWX
namespace: awx"  > kustomization.yaml

# build kustomization & kubectl apply
kustomize build . | kubectl apply -f -
kubectl config set-context --current --namespace=awx

# Check AWX execution logs
kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager


kubectl get pods -l "app.kubernetes.io/managed-by=awx-operator"
kubectl get service -l "app.kubernetes.io/managed-by=awx-operator"

# display service URL
minikube service awx-service --url -n awx

# confirm password for admin account
kubectl get secret awx-admin-password -o jsonpath="{.data.password}" | base64 --decode ;

# if you access from outside of Kubernetes cluster, it needs to set port forwarding
# [10445] ⇒ the port that Minikube installed host listens ⇒ specify any free port you like
# [80] ⇒ the port AWX container listens
kubectl port-forward service/awx-service --address 0.0.0.0 10445:80


# if using port forwarding and Firewalld is running, allow port with root privilege
firewall-cmd --add-port=10445/tcp
firewall-cmd --runtime-to-permanent


# Uninstalling AWX Operator
kustomize build . | kubectl delete -f -