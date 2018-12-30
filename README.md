Deploy a Production Ready Kubernetes Cluster
Quick Start
To deploy the cluster you can use :

Setup Kubelet Cluster by kubespray
# Get kubespray.
https://github.com/henshitou/ansible-k8s.git

# Install dependencies from "requirements.txt"
sudo pip install -r requirements.txt

# Copy "inventory/sample" as "inventory/mycluster"
cp -rfp inventory/sample inventory/mycluster

# Update Ansible inventory file with inventory builder
declare -a IPS=(192.168.1.99 192.168.1.100 192.168.1.102)
CONFIG_FILE=inventory/mycluster/hosts.ini python3 contrib/inventory_builder/inventory.py ${IPS[@]}

# Review and change parameters under "inventory/mycluster/group_vars"
cat inventory/mycluster/group_vars/all/all.yml
cat inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml

# Deploy Kubespray with Ansible Playbook
ansible-playbook -i inventory/mycluster/hosts.ini --become --become-user=root cluster.yml
Deploy Build the application within a Docker container and then load balance the application
# Get a simple hello world application with python
1. mkdir -p /apps ;git clone https://github.com/henshitou/py-dck-hello.git

2. Check below files app.py(python hello world), Dockerfile(python docker image) and py-deployment.yaml(deploy files).

3. Build and deploy python hello world
# Build the container on your local machine
  docker build -t henshitou/helloworld-python .
# Push the container to docker registry
 docker push henshitou/helloworld-python

4. Deploy the app into your cluster
kubectl apply --f py-deployment.yaml

5. Find the IP address for your service 
kubectl get svc --all-namespaces

6. To find the URL for your service, use
kubectl get svc helloworld-python  --output=custom-columns=NAME:.metadata.name,DOMAIN:.status.domain

7. Request to the app to see the result. the IP_ADDRESS refer to step5 in this part
curl -H "Host: helloworld-python.default.example.com" http://{IP_ADDRESS}
