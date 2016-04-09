Inspired from https://github.com/docker/swarm-microservice-demo-v1

=========================
  AWS Setup
=========================

#
# Pre-requisites
#

1.  Create a VPC and a subnet within it

  ```
  VPC Name:  SwarmCluster
  VPC Network:  192.168.0.0/16
  Subnet:  "PublicSubnet", 192.168.33.0/24
  ```
  (after create, make sure to enable "Auto-Assign Public IP")
  
2.  Run CloudFormation template in this repo.  

	- Use defaults for IPs
	- Select the VPC and subnet created in step 1 from the drop downs
	- Use a keypair you have the private key for, in case you need to ssh into a machine directly.

3.  You will end up with these machines:

  ```
  manager: t2.micro / 192.168.33.11
  node01: t2.micro / 192.168.33.20
  node02: t2.micro / 192.168.33.21
  node03: t2.micro / 192.168.33.200
  ```

  AMI for all:  ami-a113c6c2 in region ap-southeast-1 only
  SG for all:  SG-WideOpen
  All have public IPs.

4.  Now ssh into master using its public IP.  We will do all cluster setup from this machine.  

  Note:  to ssh into any machines:
  
	- find the machine's public IP in EC2 dashboard
	- ssh -i ~/.ssh/id_rsa_aws.pem ubuntu@[public_IP]
	
Replace ~/.ssh/id_rsa_aws.pem with the private key corresponding to the public key you entered in the CloudFormation setup.


**ALL STEPS AFTER THIS POINT DONE FROM MASTER / manager**


5. Build the Discovery Service:

  ```
  docker -H=tcp://192.168.33.11:2375 run --restart=unless-stopped -d -h consul --name consul -v /mnt:/data \
  -p 192.168.33.11:8300:8300 \
  -p 192.168.33.11:8301:8301 \
  -p 192.168.33.11:8301:8301/udp \
  -p 192.168.33.11:8302:8302 \
  -p 192.168.33.11:8302:8302/udp \
  -p 192.168.33.11:8400:8400 \
  -p 192.168.33.11:8500:8500 \
  -p 172.17.0.1:53:53/udp \
  progrium/consul -server -advertise 192.168.33.11 -bootstrap
  ```
  
  ```
  docker -H=tcp://192.168.33.11:2375 run -d --name registrator -h registrator -v /var/run/docker.sock:/tmp/docker.sock gliderlabs/registrator:latest consul://192.168.33.11:8500/
  ```
  
  
  High-availability:
  
  ```
  #docker -H=tcp://192.168.33.11:2375 run --restart=unless-stopped -d \
  -p 192.168.33.11:8300:8300 \
  -p 192.168.33.11:8301:8301 \
  -p 192.168.33.11:8301:8301/udp \
  -p 192.168.33.11:8302:8302 \
  -p 192.168.33.11:8302:8302/udp \
  -p 192.168.33.11:8400:8400 \
  -p 192.168.33.11:8500:8500 \
  -p 172.17.0.1:53:53/udp \
  -h consul1 --name consul1 progrium/consul -server -advertise 192.168.33.11 -bootstrap-expect 3
  
  #docker -H=tcp://192.168.33.12:2375 run --restart=unless-stopped -d \
  -p 192.168.33.12:8300:8300 \
  -p 192.168.33.12:8301:8301 \
  -p 192.168.33.12:8301:8301/udp \
  -p 192.168.33.12:8302:8302 \
  -p 192.168.33.12:8302:8302/udp \
  -p 192.168.33.12:8400:8400 \
  -p 192.168.33.12:8500:8500 \
  -p 172.17.0.1:53:53/udp \
  -h consul2 --name consul2 progrium/consul -server -advertise 192.168.33.12 -join 192.168.33.11
  
  #docker -H=tcp://192.168.33.13:2375 run --restart=unless-stopped -d \
  -p 192.168.33.13:8300:8300 \
  -p 192.168.33.13:8301:8301 \
  -p 192.168.33.13:8301:8301/udp \
  -p 192.168.33.13:8302:8302 \
  -p 192.168.33.13:8302:8302/udp \
  -p 192.168.33.13:8400:8400 \
  -p 192.168.33.13:8500:8500 \
  -p 172.17.0.1:53:53/udp \
  -h consul3 --name consul3 progrium/consul -server -advertise 192.168.33.13 -join 192.168.33.11
  ```

5. Build Swarm Managers:

  ```
  docker -H=tcp://192.168.33.11:2375 run --restart=unless-stopped -d -p 3375:2375 --name swarmgr swarm manage consul://192.168.33.11:8500/
  ```
  
  High-availability:
  
  ```
  #docker -H=tcp://192.168.33.11:2375 run --restart=unless-stopped -h mgr1 --name swarmgr1 -d -p 3375:2375 swarm manage --replication --advertise 192.168.33.11:3375 consul://192.168.33.11:8500/
  #docker -H=tcp://192.168.33.12:2375 run --restart=unless-stopped -h mgr1 --name swarmgr2 -d -p 3375:2375 swarm manage --replication --advertise 192.168.33.12:3375 consul://192.168.33.11:8500/
  #docker -H=tcp://192.168.33.13:2375 run --restart=unless-stopped -h mgr1 --name swarmgr3 -d -p 3375:2375 swarm manage --replication --advertise 192.168.33.13:3375 consul://192.168.33.11:8500/
  ```
  
6. Add nodes to the cluster:

  **Node01:**
  
  ```
  docker -H=tcp://192.168.33.20:2375 run --restart=unless-stopped -d -h consul-agt1 --name consul-agt1 -v /mnt:/data \
  -p 8300:8300 \
	-p 8301:8301 -p 8301:8301/udp \
	-p 8302:8302 -p 8302:8302/udp \
	-p 8400:8400 \
	-p 8500:8500 \
	-p 8600:8600/udp \
  progrium/consul -rejoin -advertise 192.168.33.20 -join 192.168.33.11
  ```
  
  ```
  docker -H=tcp://192.168.33.20:2375 run -d swarm join --advertise=192.168.33.20:2375 consul://192.168.33.20:8500/
  ```
  
  ```
  docker -H=tcp://192.168.33.20:2375 run -d --name registrator -h registrator -v /var/run/docker.sock:/tmp/docker.sock gliderlabs/registrator:latest consul://192.168.33.20:8500/
  ```
  
  **Node02:**
  
  ```
  docker -H=tcp://192.168.33.21:2375 run --restart=unless-stopped -d -h consul-agt2 --name consul-agt2 -v /mnt:/data \
  -p 8300:8300 \
  -p 8301:8301 -p 8301:8301/udp \
  -p 8302:8302 -p 8302:8302/udp \
  -p 8400:8400 \
  -p 8500:8500 \
  -p 8600:8600/udp \
  progrium/consul -rejoin -advertise 192.168.33.21 -join 192.168.33.11
  ```
  
  ```
  docker -H=tcp://192.168.33.21:2375 run -d swarm join --advertise=192.168.33.21:2375 consul://192.168.33.21:8500/
  ```
  
  ```
  docker -H=tcp://192.168.33.21:2375 run -d --name registrator -h registrator -v /var/run/docker.sock:/tmp/docker.sock gliderlabs/registrator:latest consul://192.168.33.21:8500/
  ```
  
  **Node03:**
  
  ```
  docker -H=tcp://192.168.33.200:2375 run --restart=unless-stopped -d -h consul-agt3 --name consul-agt3 -v /mnt:/data \
  -p 8300:8300 \
  -p 8301:8301 -p 8301:8301/udp \
  -p 8302:8302 -p 8302:8302/udp \
  -p 8400:8400 \
  -p 8500:8500 \
  -p 8600:8600/udp \
  progrium/consul -rejoin -advertise 192.168.33.200 -join 192.168.33.11
  ```
  
  ```
  docker -H=tcp://192.168.33.200:2375 run -d swarm join --advertise=192.168.33.200:2375 consul://192.168.33.200:8500/
  ```
  
  ```
  docker -H=tcp://192.168.33.200:2375 run -d --name registrator -h registrator -v /var/run/docker.sock:/tmp/docker.sock gliderlabs/registrator:latest consul://192.168.33.200:8500/
  ```

7. Validating / Checking
   
   Use Docker Deamon (manager):
   
   ```
   export DOCKER_HOST="tcp://192.168.33.11:2375"
   docker ps
   ```
   
   ```
   docker logs consul<1/2/3>
   docker logs swarmgr<1/2/3>
   ```
   
   ```
   docker exec -it consul bash
   consul members
   ```
   
   ```
   curl http://192.168.33.11:8500/v1/catalog/nodes | python -m json.tool
   ```

   ```
   curl http://192.168.33.11:8500/v1/catalog/services | python -m json.tool
   ```

   Switch to Docker Swarm Manager:
   
   ```
   export DOCKER_HOST="tcp://192.168.33.11:3375"
   docker ps
   docker info
   ```
   
   
8. Create overlay network

   ```
   export DOCKER_HOST="tcp://192.168.33.11:3375"
   docker network create --driver overlay mynetwork
   ```

9. Set up application layer containers:

