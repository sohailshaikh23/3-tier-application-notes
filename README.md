# #Three Tier Application

## Overview
This is a Three-Tier Web Application using `ReactJS`, `NodeJS`, and `MongoDB`, with deployment on AWS EKS.

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

```

### Setup 2 : kubectl
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


### Setup 3 : eksctl
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
sudo chown 777 /var/run/docker.sock
docker ps
```




### Step 1 Containerize the application 
FRONTEND

frontend Dockerfile 
```
vi Dockerfile
```

frontend image 
```
docker build -t sohailshaikh23/3-tier_frontend:latest
```

frontend container (check whether the frontend is up and running)
```
docker run -d --name frontend --network 3-tier -e REACT_APP_BACKEND_URL=http://15.207.108.244:3500/api/tasks  -p 3000:3000 front:latest
```

```
docker logs frontend
```


BACKEND

backend Dockerfile
```
vi Dockerfile
```

backend image
```
docker build -t sohailshaikh23/3-tier_backend:latest
```

backend container (check whether the backend is up and running)
```
docker run -dit --name api --network 3-tier -e MONGO_CONN_STR=mongodb://mongo1:27017/todo -e MONGO_USERNAME="admin" -e MONGO_PASSWORD="admin" -p 3500:3500 backend:latest
```

```
docker logs api
```




DB container : no need of dockerfile just pull the image frim dockerhub 
```
docker pull mongo:4.4.6
docker tag mongo:4.4.6 sohailshaikh23/3-tier_mongo:4.4.6
```

db container (check whetehr the DB  is up and running)
```
 docker run -dit --name mongodb --network 3-tier -p 27017:27017 -e MONGO_USERNAME="admin" -e MONGO_PASSWORD="admin"  sohailshaikh23/3-tier_mongo:4.4.6 
```

```
docker logs mongodb
```


### Step 2: Push images to DockerHub and Test your containerize Application
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
docker-compose up -d 
```

Test your Application's `frontend`

```
<PUBLIC_IP>:3000
```

Test your Application's `backendnd`
```
127.0.0.1:3500/ok
```

```
127.0.0.1:3500/api/tasks
```


Check your mongodb
```
docker exec -it mongo1 bash
```

```
show dbs
```

```
use todo
```

```
db.tasks.find() 
```





### Step 3: Setup EKS Cluster for deployment

Create a new cluster using eksctl CLI tool 
``` 
eksctl create cluster --name three-tier --region ap-south-1 --node-type t2.medium --nodes-min 2 --nodes-max 2
```


connect you kubectl with you EKS cluster

```
aws eks update-kubeconfig --region ap-south-1 --name three-tier
```

check whether your kubectl is connected to your EKS cluster or not
```
kubectl get nodes
```



### Step 4: kubernetes Manifests creation for three-tier namespace

create a custom namespace for your 3-tier Application
``` 
kubectl create namespace three-tier
```

MongoDB deployment : 
```
kubectl apply -f mongo_secret.yml 
```

```
kubectl apply -f mongo_deployment.yml
```

```
kubectl apply -f mongo_svc.yml
```



Backend deployment : 
```
kubectl apply -f backend_deployment.yml
```

```
kubectl apply -f backend_svc.yml
```


frontend deployment : NOTE :: for frontend deployment you can simply use service type as `LoadBalancer` and access from browser, but it is not good practice beause
I. It is costly to maintain seperate load balancer for each service
II. There is no URL based routing (path or host based routing)

so, the best practice is to use ingress controller for internal routing. 

```
kubectl apply -f frontend_deployment.yml
```


```
kubectl apply -f frontend_svc.yml       
```


```
kubectl get all -n three-tier
```






### Step 5: create IAM role and serviceAccount for AWS Load Balancer

You can skip these 2 step if done previously 
``` 
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```

```
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicyForEKS --policy-document file://iam_policy.json
````



```
eksctl utils associate-iam-oidc-provider --region=ap-south-1 --cluster=three-tier --approve
```

```
eksctl create iamserviceaccount --cluster=three-tier --namespace=kube-system --name=aws-load-balancer-controller --role-name AmazonEKSLoadBalancerControllerRoleForEKS --attach-policy-arn=arn:aws:iam::743542589032:policy/AWSLoadBalancerControllerIAMPolicyForEKS --approve --region=ap-south-1
```






### Step 6: Deploy AWS Load Balancer Controller
``` 
sudo snap install helm --classic
```

```
helm repo add eks https://aws.github.io/eks-charts
```

```
helm repo update eks
```

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=three-tier --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
```

```
kubectl get deployment -n kube-system aws-load-balancer-controller
```

```
kubectl apply -f ingress.yaml
```

### Cleanup
- To delete the EKS cluster:
``` 
eksctl delete cluster --name three-tier --region ap-south-1
```