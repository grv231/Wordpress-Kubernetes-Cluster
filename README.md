# Wordpress-Kubernetes-Cluster
Deploying a Wordpress image on Kubernetes cluster backed up by MySQL Database and AWS - EFS for persistence volume

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
 
3. **Git** installation for cloning the project.
- [Debian](https://www.liquidweb.com/kb/install-git-ubuntu-16-04-lts/) operating systems
- Mac OS has in-built git installation. 

### Project Setup
After checking/setting up the prerequisites, we setup the project by following the steps below in the *same order*:

1. Kubernetes Cluster needs to be created FOr this we need **kubectl** to be setup first:
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
6. Check the state of cluster using:

**Project Successful completion** should look like this:

![alt text](https://github.com/grv231/Vagrant-RedisSentinel-Consul-Clustering/blob/master/Images/SetupCompletion.png "ProjectSetupCompletion")

## Running the tests
Navigate to the folder **TestScripts** for running the tests. There are two scripts in the folder namely:

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




