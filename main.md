## Intro :
In this article, we're going to have a working setup of wordpress with Kubernetes. You're going to:

- set up a load-balanced web application that mixes stateful and stateless containers
- Learn the primitives that you'll use to configure your application on Kubernetes: Containers, Pods, Services, Deployments, ReplicaSets, DaemonSets & Volumes

## Install kubectl:

https://kubernetes.io/docs/tasks/tools/install-kubectl/

    sudo apt-get update && sudo apt-get install -y apt-transport-https
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
    sudo apt-get update
    sudo apt-get install -y kubectl
    sudo apt-get install awscli -y
    aws --version
    pip3 install awscli --upgrade --user

## Install eksctl :

    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
    brew tap weaveworks/tap
    brew install weaveworks/tap/eksctl
    brew upgrade eksctl && brew link --overwrite eksctl
    eksctl version

## AWS Credentails in local config

aws configure --profile --name of your profile
(Now type in your AWS Access Key ID. An Access Key ID can be created from the AWS Management Console)

## Download config file
    mv ~/Downloads/aws_config.yaml ~/.kube/config

## Create EKS Cluster

    eksctl create nodegroup \
    --cluster <my-cluster> \
    --region <us-west-2> \
    --name <my-mng> \
    --node-type <m5.large> \
    --nodes <3> \
    --nodes-min <2> \
    --nodes-max <4> \
    --ssh-access \
    --ssh-public-key <my-public-key.pub> \
    --managed


## Ensure cluster version and kubectl version are within 1 minor version of each other
    kubectl version

### Test it
    kubectl get nodes

## Create the secret
    kubectl apply -f secrets/wp-mysql-secrets.yaml

### Test it
    kubectl -n kube-system get secrets  

### Infrastructure Diagram:
On the left is a traditional diagram for this 3-tier web application. On the right, you see how each part of that infrastructure maps to kubernetes concepts.

Don't worry if this doesn't make sense at the beginning.

    Some config data            (k8s ConfigMaps and Secrets)

    MySQL Container             (k8s replicaset)
    MySQL Service               (k8s service)
        |
        |
    WordPress Container         (k8s deployment)
    [ apache and php-fpm ]
        |
        |
    DO Loadbalancer             (k8s service)

## 1. Mysql Setup

Secret files require base64-encoded values if you want to use them in a sane way (--from-file is hopelessly broken with .env files).

Generate new MYSQL_PASSWORD and MYSQL_ROOT_PASSWORD values like this and replace them in secrets/wp-mysql-secrets.env:

    echo && cat /dev/urandom | env LC_CTYPE=C tr -dc [:alnum:] | head -c 15 | base64 && echo


Now, Create an all-in-one secret:

    kubectl apply -f secrets/wp-mysql-secrets.yaml


Create your mysql volume and replicaset. Expose this new internal service.

    kubectl apply -f code/mysql-volume-claim.yaml
    kubectl apply -f code/mysql-replicaset.yaml
    kubectl apply -f code/mysql-service.yaml


Get a shell inside the mysql container, log into mysql, and set up the DB:

    kubectl get pods
    kubectl exec -it mysql-abcde -- bash    # replace mysql-abcde with the actual pod name

    # use the root password you created earlier (secrets/wp-mysql-secrets.yaml)
    # use the following to decode your password if needed
    # echo -n YOURBASE64PASSWORD | base64 -d
    mysql -u root -p

    # In your mysql shell:
    CREATE DATABASE wordpress;

Ctrl-d to get back out.


Check out what we just created!

    kubectl get pv
    kubectl get secrets
    kubectl get replicasets
    kubectl get pods
    kubectl describe pod $YOURPOD
    kubectl logs $YOURPOD


## 2. Wordpress Setup

Edit the config file at configs/apache.conf if you want to use a domain name for your WordPress site.

    # If you were putting a custom apache config file on your containers, this is the pattern
    # you would use:
    # kubectl create cm --from-file configs/apache.conf apache-config

    kubectl apply -f code/wordpress-datavolume-claim.yaml
    kubectl apply -f code/wordpress-deployment.yaml

Check out the pattern for getting a single config file into a container in wordpress-deployment.yaml. This is currently the best practice. Yuck!


## 3. Load Balancer Setup
It's just exposing our app to the Internet, not really load-balancing (because we're running a stateful singleton).

    kubectl apply -f code/loadbalancer.yaml


## View your work!
Grab the load balancer's external IP here:

    kubectl get services