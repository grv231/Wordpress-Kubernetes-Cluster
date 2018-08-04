# Wordpress-Kubernetes-Cluster
Deploying a Wordpress image on Kubernetes cluster backed up by MySQL Database and AWS - EFS for persistent volume

## Getting Started

These instructions will get you a copy of the project up and running on your AWS account and local machine for development purposes.

### Prerequisites

Following softwares and tools needs to be setup before proceeding with the project setup

1. The projects runs well on Linux/MacOS machines. If using Windows, you will need to create VIrtual machine. Look below for setup:
  - [Windows 10](https://www.youtube.com/watch?v=63_kPIQUPp8)
  - [Virtualbox](https://www.virtualbox.org/)

2. **Kubernetes** needs to be installed on machine. Installation can be done using following ways:
 - [Local-Setup: Minikube](https://kubernetes.io/docs/setup/minikube/) Setup
 - [Docker-Client: Windows](https://store.docker.com/editions/community/docker-ce-desktop-windows) Installation
 - [Docker-Client: MacOS](https://store.docker.com/editions/community/docker-ce-desktop-mac) Installation
 - [AWS KOPS](https://github.com/kubernetes/kops) Installation (Used in this project)

3. **IP Name** can be setup for running the cluster using one IP addresses. For cheaper options, use [Minikube](https://kubernetes.io/docs/setup/minikube)
 - [Namecheap](https://www.namecheap.com) was used to purchase IP addresses
 - [AWS-Route53 : DNS](https://aws.amazon.com/route53/) for routing the requests to AWS resources
 
4. **Git** installation for cloning the project.
- [Debian](https://www.liquidweb.com/kb/install-git-ubuntu-16-04-lts/) operating systems
- Mac OS has in-built git installation. 
<br>


### Project Setup
After checking/setting up the prerequisites, we setup the project by following the steps below in the *same order*:

1. Kubernetes Cluster needs to be created FOr this we need **kubectl** to be setup first:
<br>

**MAC-OS**
```
brew install kubernetes-cli
```

**Ubuntu/Debian**
```
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl

sudo cp kubernetes/platforms/linux/amd64/kubectl /usr/local/bin/kubectl

sudo chmod +x /usr/local/bin/kubectl
```

2. Install AWS CLI (Command line tools) for interacting with AWS cluster:
```
pip install awscli --upgrade --user
```
3. Login into the AWS account and create a S3 Bucket or simply use AWS CLI to create a bucket. *Note the name of bucket*

4. Open command terminal of your local Host OS and generate SSH Keys & move KOPS to bin folder 
```
ssh-keygen -f .ssh/id_rsa
cat .ssh/id_rsa.pub
sudo mv /usr/local/bin/kops-xx-xxxx /usr/local/bin/kops
```
5. Create the cluster in kubernetes:
```
kops create cluster --name=kubernetes.<your cluster name> --state=s3://<your-bucket-name> --zones=<zone for awscli> --node-count=2 --node-size=t2.micro --master-size=t2.micro --dns-zone=<name configured in namecheap/route53>

kops update cluster kubernetes.<your cluster name>
```
6. Check the state of cluster using:

```
kubectl get node
```
<br>

### Setting up EFS Volume
EFS volume is setup using **aws-cli**. The name will be added to the **wordpress-web** file which is explained in the next section. Steps required are:

1. Create an EFS volume using aws-cli tools. Issue the following command and copy the **FileSystemID** parameter value:
```
aws efs create-file-system --creation-token 1
```
2. Run the following command to get the **subnet-id** of the Kubernetes cluster which was launched before:
```
aws ec2 describe-instances
```
3. Get the **subnet-id** and **security groups** of either *master* or *slave* nodes from the description of following command. Now issue the following:
```
aws efs create-mount-target file-system-id <id_from step_1> --subnet-id <id_from_step_2> --security-groups <id_from_step_2>
```
<br>

### Setup files for Kubernetes cluster

We use a bunch of YAML files to setup our wordpress application which has MySQL as backend database and AWS Elastic File system (EFS) as volume mount for persistent storage for images (when we upload things to our wordpress application). Files are:

#### :one: storage.yml

This file is used to create autoprovisioned volumes on the region provided with volume type (gp2) in our case. Its signature is:

```yml
kind: StorageClass
provisioner: kubernetes.io/aws-ebs
```
#### :two: pv-claim.yml

This file is claim the storage with 8 GB specified in the signature. Its signature is:

```yml
kind: PersistenceVolumeClaim
spec:
  resources:
    requests:
      storage: 8Gi
```
#### :three: wordpress-db.yml

 - This file is used to deploy a MySQL image in pod with replication controller enabled to generate 1 prelica only.
 - Moreover,  **selector** configuration is being used to map it with our original *wordpress image*. 
 - The passwordto log into the wordpress image uses **secrets** file and stores it in MySQL database
 - Finally, the persistent volume claim of 8 GB's is mapped with **persistentvolumeclaim**

```yml
kind: ReplicationController
selector:
    app: wordpress-db
env:
- name: MYSQL_ROOT_PASSWORD
  valueFrom:
  	secretKeyRef:
           name: wordpress-secrets
           key: db-password
volumes:
- name: mysql-storage
  persistentVolumeClaim:
  	claimName: db-storage
```
#### :four: wordpress-db-service.yml

- Service definition file for database service discovery for wordpress-mysql. 
- It basically maps the service with the yml file *wordpress-db* yml file and open the wordpress db for DNS service discovery

```yml
kind: Service
spec:
  selector:
    app: wordpress-db
```
#### :five: wordpress-web.yml

- This file consists of the actual wordpress image to be setup on Pods.
- It refers to **secrets** file for password matching
- Additionally, it launches the containers in **deployment** mode, which helps in *rolling updates* to the cluster
- Finally, we also create an EFS persistent volume to account for the static images on our blog post and the respective mount path on the wordpress image itself */var/www/html/wp-content/uploads*

```yml
kind: Deployment
env:
   - name: WORDPRESS_DB_PASSWORD
     valueFrom:
        secretKeyRef:
           name: wordpress-secrets
              key: db-password
volumes:
   - name: uploads
     nfs:
        server: us-west-1b.<efs_vol_name>.efs.us-west-1.amazonaws.com
```
#### :six: wordpress-web-service.yml

- Service definition file for wordpress appication service discovery for wordpress-web. 
- It basically maps the service with the yml file *wordpress-web* yml file and open the wordpress application for DNS service discovery
- Additionally, a  classic load balancer configuration is also provided for effective load balancing using AWS ELB (Elastic Load Balancer)

```yml
kind: Service
selector:
    app: wordpress
type: LoadBalancer
```
#### :seven: wordpress-secrets.yml

- This file is used for passing secrets (*passwords* and other application information) to other configuration files. 

```yml
kind: Secret
data:
  db-password: cGFzc3dvcmQ=
```

### Deploying Wordpress application on AWS environment

After completing the above steps, run the following commands to deploy the pods in kubernetes cluster:

```bash
kubectl create -f storage.yml
kubectl create -f pv-claim.yml
kubectl create -f wordpress-db.yml
kubectl create -f wordpress-db-service.yml
kubectl create -f wordpress-web.yml
kubectl create -f wordpress-web-service.yml
kubectl create -f wordpress-secrets.yml
```

## Project Successful completion (Stages)

### Stage 1: Cluster Deployment

![alt text](https://github.com/grv231/Wordpress-Kubernetes-Cluster/blob/master/img/Clusterdeploy.jpeg "Cluster_Deployment")
<br>

### Stage 2: Cluster Validation

![alt text](https://github.com/grv231/Wordpress-Kubernetes-Cluster/blob/master/img/ValidateCluster.jpeg "Validate_Cluster")

![alt text](https://github.com/grv231/Wordpress-Kubernetes-Cluster/blob/master/img/Getnodes.jpeg "Node_Information")
<br>

### Stage 3: AWS-EFS Volume mount

![alt text](https://github.com/grv231/Wordpress-Kubernetes-Cluster/blob/master/img/EFS_Mount.jpeg "EFS_mount")
<br>

### Stage 4: File Deployment for Pods

![alt text](https://github.com/grv231/Wordpress-Kubernetes-Cluster/blob/master/img/FileDeployment.jpeg "Pods_files_deployment")
<br>

### Stage 5: Route 53 configuration for Load Balancers

![alt text](https://github.com/grv231/Wordpress-Kubernetes-Cluster/blob/master/img/Route53_config.jpg "Route53_config")
<br>

### Stage 6: Wordpress Access via URL (http://wordpress.kubernetes.kubetest231.site)

![alt text](https://github.com/grv231/Wordpress-Kubernetes-Cluster/blob/master/img/Wordpress_Image.jpg "Wordpress_URL")
<br>

### Stage 7: Successful static images upload in Wordpress and storage in EFS

![alt text](https://github.com/grv231/Wordpress-Kubernetes-Cluster/blob/master/img/Wordpress_EFS.jpg "StaticImage_EFS_Wordpress")
<br>


## Running the tests

- **Testing Fault Tolerance**

Firstly, we are testing fault tolerance -HA by issuing *delete pods* commands. In this case, all running containers are terminated 	 however, new containers take its place. Firstly, to get all poid information, we issue
```bash
kubectl get pods
```
After getting the information, we issue delete commands on all Pods to check fault tolerance.
```bash
kubectl delete pods/wordpress-db-<unique-id>
kubectl delete pods/wordpress-deployment-<unique-id>
kubectl delete pods/wordpress-deployment-<unique-id>
```
When we issue the following commands, automatically old containers are terminated and new ones pop up, automatically handling High availabilaty-Fault tolderance by the Kubernetes cluster iteself.

![alt text](https://github.com/grv231/Wordpress-Kubernetes-Cluster/blob/master/img/HA.jpeg "Pods_FaultTolerance")

Addtionally, when we issue logs on the pods, we get the messages that wordpress was still present on the pods. COmmand issued is:
```bash
kubectl logs wordpress-deployment-<unique-id>
```
![alt text](https://github.com/grv231/Wordpress-Kubernetes-Cluster/blob/master/img/logs.jpeg "Pods_logs")
<br>


- **Testing Persistent volume - AWS EFS**
Now we login into one of the deployment pods to check whether our static image is still present or not (after destroying-automaticaaly recreated pods). We issue the following command to login into the pods and run bash commands:
```bash
kubectl exec wordpress-deployment-<unique-id> -it /bin/bash
ls wp-contents/uploads/2018/08
```
For this project, a music mixer image was added to the blogpost. It still persists, even when the pods were destroyed!!

![alt text](https://github.com/grv231/Wordpress-Kubernetes-Cluster/blob/master/img/Persistence.jpeg "Image_Persistence")


## Built With

* [Kubernetes](https://github.com/kubernetes) - Managing containers in a cluster for High Availabilaty
* [Wordpress](https://wordpress.com) - Application deployed on pods
* [KOPS](https://github.com/kubernetes/kops) - Provisioning Kubernetes cluster on Amazon Web Services
* [kubectl](https://rancher.com/docs/rancher/v2.x/en/k8s-in-rancher/kubectl/) - Issuing commands to kubernetes cluster
* [Amazon Web Services (AWS)](https://aws.amazon.com) - Cloud-platform for deploying kubernetes cluster
* [AWS-EC2](https://aws.amazon.com/ec2/) - Servers used for Master-Slave nodes
* [AWS-Route53](https://aws.amazon.com/route53/) - DNS service for AWS
* [AWS-ElasticFileSystem (EFS)](https://aws.amazon.com/efs/) - Service used for deploying persistent volumes
* [Namecheap](https://www.namecheap.com/) - Domain for deploying cluster




