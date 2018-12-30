Deploy a Production Ready Kubernetes Cluster
Quick Start
To deploy the cluster you can use :

## Setup Kubelet Cluster by kubespray
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

## Deploy Build the application within a Docker container and then load balance the application
# Get a simple hello world application with python
1. mkdir -p /apps ;git clone https://github.com/henshitou/py-dck-hello.git

2. Check below files helloworld.py(python hello world), Dockerfile(python docker image) and py-deployment.yaml(deploy files).
   1>  helloworld.py
    import os
    from flask import Flask
    app = Flask(__name__)
    @app.route('/')
    def hello_world():
        target = os.environ.get('TARGET', 'World')
        return 'Hello {}!\n'.format(target)
    if __name__ == "__main__":
        app.run(debug=True,host='0.0.0.0',port=int(os.environ.get('PORT', 8080)))
        
   2> Dockfile 
     # Use the official Python image.
     FROM python
     # Copy python hello world to the container image.
     ENV APP_HOME /app
     WORKDIR $APP_HOME
     COPY helloworld.py /app
     # Install production dependencies.
     RUN pip install Flask
     # Configure and document the service HTTP port.
     ENV PORT 8080
     EXPOSE $PORT
     # Run the web service on container startup.
     CMD ["python", "helloworld.py"]
        
     3> py-deployment.yaml
     
     apiVersion: v1
     kind: Service
     metadata:
       name: helloworld-python
       namespace: default
     spec:
       type: NodePort
       ports:
         - port: 8080
           targetPort: 8080
           nodePort: 32001
       selector:
         app: helloworld-python
     ---
     apiVersion: extensions/v1beta1
     kind: Deployment
     metadata:
       name: helloworld-python
       namespace: default
     spec:
       replicas: 1
       template:
         metadata:
           labels:
             app: helloworld-python  # define this when service to slect it
         spec:
           containers:
             - name: helloworld-python
               image: docker.io/henshitou/helloworld-python  # the Image which you want to use
               imagePullPolicy: IfNotPresent # if the image is not in local, pull it from image respository
               ports:
                 - name: http-port
                   containerPort: 8080  # Container port
               volumeMounts:  # The volume for mounting in contianer
                 - name: apps-home
                   mountPath: /app # Mount point in container
          volumes:
            - name: apps-home # the volume you want to mount to contianer
              persistentVolumeClaim:
                claimName: apps-pv-claim
     ---
     apiVersion: v1
     kind: PersistentVolumeClaim
     metadata:
       name: apps-pv-claim
       namespace: default
     spec:
       accessModes:
         - ReadWriteOnce
     selector: nfs-share
     resources:
       requests:
       storage: 5Gi
     
     # Notice: before claim the persistentvolume, you need to create persistentvolume
     kubectl apply --f hellopy-pv.yaml (fill below contain to this yaml file)
     
     kind: PersistentVolume \
     apiVersion: v1 \
     metadata:
       name: apps-pv-volume
       namespace: default
       annotations:
         pv.beta.kubernetes/gid: "1000"
       labels:
         type: helloworld-python
     spec:
       storageClassName: manual
       capacity:
         storage: 10Gi
       accessModes:
         - ReadWriteOnce
       hostPath:
         path: "/app/home"     
     
3. Build and deploy python hello world  
Build the image on your local machine
  docker build -t henshitou/helloworld-python .
Push the container to docker registry
 docker push henshitou/helloworld-python

4. Deploy the app into your cluster
kubectl apply --f py-deployment.yaml

5. Find the IP address for your service 
kubectl get svc --all-namespaces

6. To find the URL for your service, use
kubectl get svc helloworld-python  --output=custom-columns=NAME:.metadata.name,DOMAIN:.status.domain

7. Request to the app to see the result. the IP_ADDRESS refer to step5 in this part
curl -H "Host: helloworld-python.default.example.com" http://{IP_ADDRESS}

