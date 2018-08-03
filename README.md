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
5. Check the state of cluster using:

```
kubectl get node
```
6. Create a EFS volume on AWS and copy its name. Following [this](https://docs.aws.amazon.com/efs/latest/ug/gs-step-two-create-efs-resources.html) for creating EFS volume. Copy its name once its done. The name will be added to the **wordpress-web** file which is explained in the next section.

Another way is to setup EFS is using **aws-cli**. Steps required are:

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
        server: us-west-1a.<efs_vol_name>.efs.us-west-1.amazonaws.com
```
#### :six: wordpress-web-service.yml

- Service definition file for wordpress appication service discovery for wordpress-web. 
- It basically maps the service with the yml file *wordpress-web* yml file and open the wordpress application for DNS service discovery
- Additionally, a load balancer configuration is alos provided for effictive load balancing using AWS ELB (Elastic Load Balancer)

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
![alt text](https://github.com/grv231/Vagrant-RedisSentinel-Consul-Clustering/blob/master/Images/SetupCompletion.png "Cluster_Deployment")

### Stage 2: Cluster Validation

![alt text](https://github.com/grv231/Vagrant-RedisSentinel-Consul-Clustering/blob/master/Images/SetupCompletion.png "Cluster_Deployment")

![alt text](https://github.com/grv231/Vagrant-RedisSentinel-Consul-Clustering/blob/master/Images/SetupCompletion.png "Cluster_Deployment")

### Stage 3: AWS-EFS Volume mount

![alt text](https://github.com/grv231/Vagrant-RedisSentinel-Consul-Clustering/blob/master/Images/SetupCompletion.png "Cluster_Deployment")

![alt text](https://github.com/grv231/Vagrant-RedisSentinel-Consul-Clustering/blob/master/Images/SetupCompletion.png "Cluster_Deployment")

### Stage 4: File Deployment

![alt text](https://github.com/grv231/Vagrant-RedisSentinel-Consul-Clustering/blob/master/Images/SetupCompletion.png "Cluster_Deployment")

### Stage 5: Wordpress Access via URL

![alt text](https://github.com/grv231/Vagrant-RedisSentinel-Consul-Clustering/blob/master/Images/SetupCompletion.png "Cluster_Deployment")

### Stage 6: Successful static images upload in Wordpress and storage in EFS

![alt text](https://github.com/grv231/Vagrant-RedisSentinel-Consul-Clustering/blob/master/Images/SetupCompletion.png "Cluster_Deployment")


## Running the tests
For checking cluster related information, node health status and wordpress image access, please refer to commands and screenshots below:

1. **RedisConsulSmokeTest.sh**
   This file is used for running smoke tests on Redis (redismaster and Slaves) and Consul Clusters. In the script, it has been              specifically written for the server *redismaster*. It can be used for running on the slaves as well, however, the server names and      commands would need to be changed accordingly.

   Make sure that you are not logged into any servers provisioned (redismaster, slaves or sentinel) before running the script. The          script needs to be run where the *Vagrantfile* is present from CLI. Run the following command:
```
vagrant ssh redismaster -c ‘/vagrant/TestScripts/RedisConsulSmokeTest.sh; /bin/bash’
```
**Test Output**
![alt text](https://github.com/grv231/Vagrant-RedisSentinel-Consul-Clustering/blob/master/Images/RedisConsulTest.png "RedisConsulSmokeTest")


2. **SentinelSmoketest.sh**
   This file is used for running smoke tests on only for checking Redis Clustering. Since the Redis Sentinel servers were put up on a      different server, we can check the Redis Cluster status from Redis Sentinels (example **redissentinel01**). Elaborate information can    be gathered from these tests using Redis Sentinels servers.

   Make sure that you are not logged into any servers provisioned (redismaster, slaves or sentinel) before running the script. The          script needs to be run where the *Vagrantfile* is present. For running the script, use the following command on the command line:
```
vagrant ssh redissentinel01 -c ‘/vagrant/TestScripts/SentinelSmoketest.sh; /bin/bash’
```
**Test Output**
![alt text](https://github.com/grv231/Vagrant-RedisSentinel-Consul-Clustering/blob/master/Images/RedisSentinelTest.png "RedisSentinelSmokeTest")

3. **Healthcheck for Redis service using Consul**
   Healthcheck information has been configured in the *consulmasterscript.sh* and *consulslavescript.sh*. This healthcheck gives the        information about Redis service running on port 6379. If the service is on, consul shows **service sync** successful.
   
   The healthcheck was setup in the JSON file using the following entries
```json
"service": {
	"name": "redis6379",
	"tags": ["master"],
	"port": 6379,
	"check": {
		"args": ["nc", "-zv", "127.0.0.1", "6379"],
		"interval": "5s"
	}
}
```

The information can be gathered by running the following command on any node (Master or Slave):
```
consul monitor
```
**Healthcheck**
![alt text](https://github.com/grv231/Vagrant-RedisSentinel-Consul-Clustering/blob/master/Images/HealthCheck.png "Healthcheck")

## Important Information
 - Cluster can be resized by adding/removing nodes in the *Vagrantfile* **BOXES** variable.
 - Additionally, add the master,slave,sentinel nodes in the **hosts** file for correct *Ansible* provisioning
 - Quorum can be increased/decresed for Sentinel Clusters using the file *ansible-redis/defaults/main.yml*. Values can be                  increased/decresed in this file.
```yaml
redis_sentinel_monitors:
  - name: master01
    host: localhost
    port: 6379
    quorum: 2
    auth_pass: ant1r3z
    down_after_milliseconds: 30000
    parallel_syncs: 1
    failover_timeout: 180000
    notification_script: false
    client_reconfig_script: false
```
## Architecture Diagram

![alt text](https://github.com/grv231/Vagrant-RedisSentinel-Consul-Clustering/blob/master/Images/Arch_RedisConsul.jpg "Design")

## Built With

* [Vagrant](https://www.vagrantup.com/docs/) - Managing virtual machines
* [Vagrant Hostmanager](https://github.com/devopsgroup-io/vagrant-hostmanager) - Managing multiple machine environment
* [Ansible](https://docs.ansible.com/) - Provisioning redis and redis-sentinel on servers
* [Redis](https://redis.io/documentation) - Redis server (Master-Slave setup)
* [Redis Sentinel](https://redis.io/topics/sentinel) - Sentinel for monitoring Master-Slaves Redis cluster
* [Consul](https://www.consul.io/docs/index.html) - Consul for service discovery in a cluster setup
* [Ruby](http://tutorials.jumpstartlab.com/topics/vagrant_setup.html) - Ruby for vagrant setup




