# #Three Tier Application

## Overview
This is a Three-Tier Notes Web Application using `ReactJS`, `NodeJS`, and `MongoDB`, with deployment on AWS EKS.

## Prerequisites
- Create a user from IAM with `AdministratorAccess`.
- Generate Security Credentials: `Access Key` and `Secret Access Key`
- Launch an Ubuntu instance as a bastion host(eg. region `ap-south-1`).
- Setup these 3 things `AWS CLI`, `kubectl`, `eksctl`,	`Docker`. refer below


### Setup 1 : AWS CLI v2
``` shell
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install -i /usr/local/aws-cli -b /usr/local/bin --update
aws configure
```

### Setup 2 : Install kubectl {v1.19.6}
``` 
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
```
```
chmod +x ./kubectl
```

```
sudo mv ./kubectl /usr/local/bin
```

```
kubectl version --short --client
```


### Setup 3 : eksctl {0.176.0}
``` shell
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```

```
sudo mv /tmp/eksctl /usr/local/bin
```

```
eksctl version
```

### Setup 4 Install Docker
``` shell
sudo apt-get update -y
sudo apt install docker.io -y
sudo chmod 777 /var/run/docker.sock
docker ps
```



### Phase 1: Containerize the application Using Docker into containers 

Step 1 : Write Docker file of the application, create image from the dockerfile, check the container are up and running  

DB  : no need of dockerfile just pull the image frim dockerhub 
```
docker pull mongo:4.4.6
docker tag mongo:4.4.6 sohailshaikh23/3-tier_mongo:4.4.6
```

db container (check whetehr the DB  is up and running)
```
docker run -dit --name mongo --network notes-webapp -p 27017:27017 -e MONGO_USERNAME="admin" -e MONGO_PASSWORD="admin"  sohailshaikh23/3-tier_mongo:4.4.6 
```

```
docker logs mongodb
```


BACKEND

backend Dockerfile
```
vi Dockerfile
```

backend image
```
docker build -t sohailshaikh23/3-tier_backend:latest .
```

backend container (check whether the backend is up and running)
```
docker run -dit --name api --network notes-webapp -e MONGO_CONN_STR=mongodb://mongo:27017/todo -e MONGO_USERNAME="admin" -e MONGO_PASSWORD="admin" -p 8080:8080 sohailshaikh23/3-tier_backend:latest
```

```
docker logs api
```


FRONTEND

frontend Dockerfile 
```
vi Dockerfile
```

frontend image 
```
docker build -t sohailshaikh23/3-tier_frontend:latest .
```

frontend container (check whether the frontend is up and running)
```
docker run -d --name frontend --network notes-webapp -e REACT_APP_BACKEND_URL=http://15.207.108.244:8080/api/tasks -p 3000:3000 sohailshaikh23/3-tier_frontend:latest
```

```
docker logs frontend
```


Step 2: Push images to DockerHub and Test your containerize Application

```
docker push sohailshaikh23/3-tier_frontend:latest
```

```
docker push sohailshaikh23/3-tier_backend:latest
```

```
docker push sohailshaikh23/3-tier_mongo:4.4.6
```


deploy the containerize application using docker-compose tool 

```
docker compose up -d
```
Test your Application's `frontend`

```
<PUBLIC_IP>:3000
```

Test your Application's `backendnd`
```
127.0.0.1:8080/ok
```

```
127.0.0.1:8080/api/tasks
```


Check your mongodb
```
docker exec -it mongo bash
```

go to mongo shell
```
mongo
```

check Databases
```
show dbs
```

```
use todo
```

View your database content
```
db.tasks.find() 
```



### Phase 2: EKS Cluster for deployment

Step 1 : - Create a new cluster using eksctl CLI tool 
``` 
eksctl create cluster --name notes-webapp --region ap-south-1 --node-type t2.medium --nodes-min 2 --nodes-max 2
```
or create manually
         - Create EKS cluster with IAM Role `myAmazonEKSClusterRole`, along with it add `EBS CSI Driver` from add ons section {this Enable Amazon         Elastic Block Storage (EBS) within your cluster and it is responsible to create persistent volumes automatically}
         - Create NodeGroup (2 nodes of t2.medium instance type) with IAM Role `myAmazonEKSNodeRole` 


Step 2 : - Use you Ubuntu instance as bastion host (t2.micro). attach an `IAM role with admin Access`/or/ use the below policy /or/ use `aws configure credentials` and provide your `access key` and `secret key`


Once the Cluster is ready run the command to connect EKS cluster with your kubectl tool:
```
aws eks update-kubeconfig --name notes-webapp --region ap-south-1
```

check whether your kubectl is connected to your EKS cluster or not
```
kubectl get nodes
```


### Phase 3: Prerequisites for EKS cluster
 
create IAM role and serviceAccount for AWS Load Balancer

You can skip these 2 step (commands) {if done previously }

I - Download the policy 
``` 
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```
II - Create a policy in IAM
```
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicyForEKS --policy-document file://iam_policy.json
````

Associate OpenIDconnect provider for the cluster
```
eksctl utils associate-iam-oidc-provider --region=ap-south-1 --cluster=notes-webapp --approve
```

Create a service account 
```
eksctl create iamserviceaccount --cluster=notes-webapp --namespace=kube-system --name=aws-load-balancer-controller --role-name AmazonEKSLoadBalancerControllerRoleForEKS --attach-policy-arn=arn:aws:iam::743542589032:policy/AWSLoadBalancerControllerIAMPolicyForEKS --approve --region=ap-south-1
```

Now Deploy AWS Load Balancer Controller to control the AWS loadbalancer
``` 
sudo snap install helm --classic
```

Add the Github repo for the ALB controller
```
helm repo add eks https://aws.github.io/eks-charts
```

```
helm repo update eks
```

Install the Load Balancer controller inside your cluster
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=notes-webapp --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
```
aws
Check whether the controller in properly install or not 
```
kubectl get deployment -n kube-system aws-load-balancer-controller
```



### Phase 4 : kubernetes Manifests creation for notes-webapp 

**Create a custom Namespace as notes-webapp **
``` 
kubectl create namespace notes-webapp
```

** set this namespace as default **
```
kubectl config set-context --current --namespace notes-webapp
```

**MONGO Database deployment**


Create and apply Secrets files
```
kubectl apply -f mongo_secret.yml 
```

create Mongo statefulset with Persistent volumes :
```
kubectl apply -f mongo_deployment.yml   
```

create Mongo Service : 
```
kubectl apply -f mongo-svc.yaml
```

Check whether EBS is created, PV/PVC is created or not
```
kubectl get pv
```

```
kubectl get pvc
```


Go inside mongo-0 pod and set replication
```
kubectl exec -it mongo-0 -- mongo  
```

Run these commands while you inside to set replication (main) and secondary

```
rs.initiate(); 
sleep(2000);
rs.add("mongo-1.mongo:27017");
sleep(2000);
rs.add("mongo-2.mongo:27017");
sleep(2000);
cfg = rs.conf();
cfg.members[0].host = "mongo-0.mongo:27017";
rs.reconfig(cfg, {force: true});
sleep(5000);
```

Now check the status
```
rs.status()
``` 

create a database with name `todo`
```
use todo;
```

CHECK YOUR  DATA 

```
db.tasks.find().pretty();
```


**Backend/API deployment Setup**


Create and apply backend deployment files
```
kubectl apply -f deployment.yml
```

Create and apply backend service files
```
kubectl apply -f service.yml
```

Check whether Backend is connected to mono or not
```
docker logs api
```


frontend deployment NOTE :: for frontend deployment you can simply use service type as `LoadBalancer` and access from browser, but it is not good practice beause
I. It is costly to maintain seperate load balancer for each service
II. There is no URL based routing (path or host based routing)

so, the best practice is to use ingress controller for internal routing. 



**Frontend Deployment Setup**


Create and apply frontend deployment files
```
kubectl apply -f deployment.yml
```

Create and apply frontend service files 
```
kubectl apply -f service.yml       
```


Check all the applied services
```
kubectl get all 
```

Check the logs of frontend
```
kubectl logs frontend
```


Create and apply Ingress controller manifest and apply to you cluster for internal routing
```
kubectl apply -f ingress.yaml
```

Check you ingress controller
```
kubectl get ingress # fail build model,address field is blank
```




### Cleanup
- To delete the EKS cluster:
``` 
eksctl delete cluster --name notes-webapp --region ap-south-1
```
