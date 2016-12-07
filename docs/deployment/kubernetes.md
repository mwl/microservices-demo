---
layout: default
deployDoc: true
---

## Sock Shop on Kubernetes + Weave 

This demo demonstrates running the Sock Shop on a Kubernetes cluster using
Weave Net and Weave Scope. In addition to the basic Kubernetes demo this 
demo demonstrates using Weave Flux to automate the deployment workflow.

### Pre-requisites
* *Optional* [Terraform](https://www.terraform.io/downloads.html)
* *Optional* [AWS Account](https://aws.amazon.com/)
* *Optional* [awscli](http://docs.aws.amazon.com/cli/latest/userguide/installing.html)

<!-- deploy-doc require-env AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_DEFAULT_REGION SLACK_WEBHOOK -->
<!-- deploy-doc-start pre-install -->

    curl -sSL https://get.docker.com/ | sh
    apt-get install -yq jq python-pip curl unzip build-essential python-dev

    pip install awscli

    curl -o /tmp/terraform.zip https://releases.hashicorp.com/terraform/0.7.11/terraform_0.7.11_linux_amd64.zip
    unzip /tmp/terraform.zip -d /usr/bin

    curl -o /usr/bin/fluxctl -sSL https://github.com/weaveworks/flux/releases/download/master-ccb9a99/fluxctl-linux-amd64
    chmod +x /usr/bin/fluxctl

<!-- deploy-doc-end -->

For the purposes of this demo you'll need to fork the `microservices-demo` repo <a href="https://github.com/microservices-demo/microservices-demo" target="_blank">here</a>

<!-- deploy-doc-hidden pre-install

    cat > /root/healthcheck.sh <<-EOF
#!/usr/bin/env bash
kubectl delete -\-namespace=sock-shop deployment healthcheck
kubectl run -\-namespace=sock-shop healthcheck -\-image=andrius/alpine-ruby sleep 10000
sleep 90
kube_id=\$(kubectl get pods -\-namespace=sock-shop | grep healthcheck | awk '{print \$}')
kubectl exec -\-namespace=sock-shop \$kube_id -\- sh -c "curl -o healthcheck.rb \"https://raw.githubusercontent.com/microservices-demo/microservices-demo/master/deploy/healthcheck.rb\"; chmod +x ./healthcheck.rb; ./healthcheck.rb -s user,catalogue,queue-master,cart,shipping,payment,orders"
EOF

    mkdir -p ~/.ssh/
    cp $PWD/microservices-demo_id_rsa ~/.ssh/
    ssh-keygen -y -f ~/.ssh/microservices-demo_id_rsa > ~/.ssh/microservices-demo_id_rsa.pub
    aws ec2 describe-key-pairs -\-key-name microservices-demo-flux
    if [ $? -eq 0 ]; then
        touch /root/deploy-exists
    else
        ./deploy/kubernetes/terraform/cleanup.sh
    fi
-->
### Setup Kubernetes

Begin by setting the appropriate AWS environment variables.
```
export AWS_ACCESS_KEY_ID=[YOURACCESSKEYID]
export AWS_SECRET_ACCESS_KEY=[YOURSECRETACCESSKEY]
export AWS_DEFAULT_REGION=[YOURDEFAULTREGION]
```

First we'll create a repo key and upload it to AWS. Next we'll upload it to github for the repo we just forked above.

```
ssh-keygen -t rsa -b 4096 -C "your_email@example.com" -f ~/.ssh/microservices-demo_id_rsa
chmod 600 ~/.ssh/microservices-demo_id_rsa
aws ec2 import-key-pair -\-key-name microservices-demo-flux -\-public-key-material file://$HOME/.ssh/microservices-demo_id_rsa.pub
```

<!-- deploy-doc-hidden create-infrastructure

    if [ ! -f /root/deploy-exists ]; then
        aws ec2 import-key-pair -\-key-name microservices-demo-flux -\-public-key-material file://$HOME/.ssh/microservices-demo_id_rsa.pub
    else
        echo "Flux deployment exists. Skipping. Delete the aws key microservices-demo-flux to redeploy."
    fi

-->

You may opt to take advantage of the command below to add the public key to your forked repo or perfer to do it manually through the github web interface
```
curl -u <YOUR_GITHUB_USERNAME> -d '{"title": "microservices-demo-flux", "key": "'"`cat ~/.ssh/microservices-demo_id_rsa.pub`"'", "read_only": false}' https://api.github.com/repos/<YOUR_GITHUB_USERNAME>/microservices-demo/keys
```

Finally run terraform.

```
export TF_VAR_private_key_path="~/.ssh/microservices-demo_id_rsa"
export TF_VAR_key_name="microservices-demo-flux"
terraform apply deploy/kubernetes/terraform/
```
<!-- deploy-doc-hidden create-infrastructure

    if [ ! -f /root/deploy-exists ]; then
        export TF_VAR_private_key_path="~/.ssh/microservices-demo_id_rsa"
        export TF_VAR_key_name="microservices-demo-flux"
        terraform apply deploy/kubernetes/terraform/
    else
        echo "Flux deployment exists. Skipping. Delete the aws key microservices-demo-flux to redeploy."
    fi

-->

Our master node makes use of some of the files in this repo so lets securely copy those over.

```
master_ip=$(terraform output -json | jq -r '.master_address.value')
scp -i ~/.ssh/microservices-demo_id_rsa -o StrictHostKeyChecking=no deploy/kubernetes/weavescope.yaml ubuntu@$master_ip:/tmp/
scp -i ~/.ssh/microservices-demo_id_rsa -rp deploy/kubernetes/manifests ubuntu@$master_ip:/tmp/
```
<!-- deploy-doc-hidden create-infrastructure

    if [ ! -f /root/deploy-exists ]; then
        master_ip=$(terraform output -json | jq -r '.master_address.value')
        scp -i ~/.ssh/microservices-demo_id_rsa -o StrictHostKeyChecking=no deploy/kubernetes/weavescope.yaml ubuntu@$master_ip:/tmp/
        scp -i ~/.ssh/microservices-demo_id_rsa -rp deploy/kubernetes/manifests ubuntu@$master_ip:/tmp/
    else
        echo "Flux deployment exists. Skipping. Delete the aws key microservices-demo-flux to redeploy."
    fi

-->

### <a name="weavenet"></a>Setup Weave Net
* Run the following commands to setup Kubernetes and Weave Net on the master instance

```
master_ip=$(terraform output -json | jq -r '.master_address.value')
ssh -i ~/.ssh/microservices-demo_id_rsa ubuntu@$master_ip sudo kubeadm init > k8s-init.log
grep -e --token k8s-init.log > join.cmd
ssh -i ~/.ssh/microservices-demo_id_rsa ubuntu@$master_ip kubectl apply -f https://git.io/weave-kube
```

<!-- deploy-doc-hidden create-infrastructure

    if [ ! -f /root/deploy-exists ]; then
        master_ip=$(terraform output -json | jq -r '.master_address.value')
        ssh -i ~/.ssh/microservices-demo_id_rsa ubuntu@$master_ip sudo kubeadm init > k8s-init.log
        grep -e -\-token k8s-init.log > join.cmd
        ssh -i ~/.ssh/microservices-demo_id_rsa ubuntu@$master_ip kubectl apply -f https://git.io/weave-kube
    else
        echo "Flux deployment exists. Skipping. Delete the aws key microservices-demo-flux to redeploy."
    fi

-->

### Time for the nodes to join the master
* Run the following commands to SSH into each node\_addresses listed when you ran the ```terraform output``` command and run the ```kubeadm join --token <token> <master-ip>``` command from before.

```
node_addresses=$(terraform output -json | jq -r '.node_addresses.value|@sh' | sed -e "s/'//g" )
for node in $node_addresses; do
    ssh -i ~/.ssh/microservices-demo_id_rsa -o StrictHostKeyChecking=no ubuntu@$node sudo `cat join.cmd`
done
```

<!-- deploy-doc-hidden create-infrastructure

    if [ ! -f /root/deploy-exists ]; then
        node_addresses=$(terraform output -json | jq -r '.node_addresses.value|@sh' | sed -e "s/'//g" )
        for node in $node_addresses; do
            ssh -i ~/.ssh/microservices-demo_id_rsa -o StrictHostKeyChecking=no ubuntu@$node sudo `cat join.cmd`
        done
    else
        echo "Flux deployment exists. Skipping. Delete the aws key microservices-demo-flux to redeploy."
    fi

-->

### Setup Weave Scope
* SSH into the master node
* Start weave scope on the cluster

```
master_ip=$(terraform output -json | jq -r '.master_address.value')
ssh -i ~/.ssh/microservices-demo_id_rsa ubuntu@$master_ip kubectl apply -f /tmp/weavescope.yaml -\-validate=false
```

<!-- deploy-doc-hidden create-infrastructure

    if [ ! -f /root/deploy-exists ]; then
        master_ip=$(terraform output -json | jq -r '.master_address.value')
        ssh -i ~/.ssh/microservices-demo_id_rsa ubuntu@$master_ip kubectl apply -f /tmp/weavescope.yaml -\-validate=false
    else
        echo "Flux deployment exists. Skipping. Delete the aws key microservices-demo-flux to redeploy."
    fi

-->

### Deploy Sock Shop
* SSH into the master node
* Deploy the sock shop

```
master_ip=$(terraform output -json | jq -r '.master_address.value')
ssh -i ~/.ssh/microservices-demo_id_rsa ubuntu@$master_ip kubectl apply -f /tmp/manifests/sock-shop-ns.yml -f /tmp/manifests
```

<!-- deploy-doc-hidden create-infrastructure

    if [ ! -f /root/deploy-exists ]; then
        master_ip=$(terraform output -json | jq -r '.master_address.value')
        ssh -i ~/.ssh/microservices-demo_id_rsa ubuntu@$master_ip kubectl apply -f /tmp/manifests/sock-shop-ns.yml -f /tmp/manifests
    else
        echo "Flux deployment exists. Skipping. Delete the aws key microservices-demo-flux to redeploy."
    fi

-->

### Setup Weave Flux

* Create a file called flux.conf similar to the one displayed below
* Fill in values that are applicable to your deployment

```
git:
  URL: git@github.com:<YOUR_GITHUB_USERNAME>/microservices-demo
  path: deploy/kubernetes/manifests
  branch: master
  key: |
        -----BEGIN RSA PRIVATE KEY-----
        nTooXXGagxg5a3vqsGPgoHH1KvqE5my+v7uYhRxbHi5uaTNEWnD46ci06PyBz
        zSS6I+zgkdsQk7Pj2DNNzBS6n08gl8OJX073JgKPqlfqDSxmZ37XWdGMlkeIuS21
        nwli0jsXVMKO7LYl+b5a0N5ia9cqUDEut1eeKN+hwDbZeYdT/oGBsNFgBRTvgQhK
        ... contents of deploy-docs-k8s.pem file from above ...
        -----END RSA PRIVATE KEY-----
slack:
  hookURL: ""
  username: ""
registry:
  auths: {}
```

<!-- deploy-doc-hidden pre-install

    if [ ! -f /root/deploy-exists ]; then
        mkdir -p /home/ubuntu/
        cat > /home/ubuntu/flux.conf <<-EOF
git:
  URL: git@github.com:microservices-demo/microservices-demo
  path: deploy/kubernetes/manifests
  branch: master
  key: |
$(cat microservices-demo_id_rsa | sed -e 's/^/    /g')
slack:
  hookURL: $SLACK_WEBHOOK
  username: "deploydocs-flux"
registry:
  auths: {}
EOF
    else
        echo "Flux deployment exists. Skipping. Delete the aws key microservices-demo-flux to redeploy."
    fi

-->

* SSH into the master node
* Start the flux deployment and service
* Set fluxctl to use flux.conf
* Set FLUX_URL
* Iterate over the services setting them to update automatically


```
master_ip=$(terraform output -json | jq -r '.master_address.value')

ssh -i ~/.ssh/microservices-demo_id_rsa ubuntu@$master_ip kubectl create -f 'https://raw.githubusercontent.com/weaveworks/flux/master/deploy/standalone/flux-deployment.yaml'
ssh -i ~/.ssh/microservices-demo_id_rsa ubuntu@$master_ip kubectl create -f 'https://raw.githubusercontent.com/weaveworks/flux/master/deploy/standalone/flux-service.yaml'

echo "Sleeping for 120s.."
sleep 120

flux_port=$(ssh -i ~/.ssh/microservices-demo_id_rsa ubuntu@$master_ip kubectl get service fluxsvc -\-template \'{{ index .spec.ports 0 \"nodePort\" }}\')
aws ec2 authorize-security-group-ingress -\-group-name MD-k8s-security-group -\-protocol tcp -\-port $flux_port -\-cidr 0.0.0.0/0
export FLUX_URL=http://$master_ip:$flux_port

fluxctl set-config -\-file=/home/ubuntu/flux.conf
for svc in front-end catalogue orders queue-master user cart catalogue user-db catalogue-db payment shipping; do
    fluxctl automate -\-service=sock-shop/$svc
done
```

<!-- deploy-doc-hidden create-infrastructure

    if [ ! -f /root/deploy-exists ]; then
        master_ip=$(terraform output -json | jq -r '.master_address.value')

        ssh -i ~/.ssh/microservices-demo_id_rsa ubuntu@$master_ip kubectl create -f 'https://raw.githubusercontent.com/weaveworks/flux/master/deploy/standalone/flux-deployment.yaml'
        ssh -i ~/.ssh/microservices-demo_id_rsa ubuntu@$master_ip kubectl create -f 'https://raw.githubusercontent.com/weaveworks/flux/master/deploy/standalone/flux-service.yaml'

        echo "Sleeping for 120s.."
        sleep 120

        flux_port=$(ssh -i ~/.ssh/microservices-demo_id_rsa ubuntu@$master_ip kubectl get service fluxsvc -\-template \'{{ index .spec.ports 0 \"nodePort\" }}\')
        aws ec2 authorize-security-group-ingress -\-group-name MD-k8s-security-group -\-protocol tcp -\-port $flux_port -\-cidr 0.0.0.0/0
        export FLUX_URL=http://$master_ip:$flux_port

        fluxctl set-config -\-file=/home/ubuntu/flux.conf
        for svc in front-end catalogue orders queue-master user cart catalogue user-db catalogue-db payment shipping; do
            fluxctl automate -\-service=sock-shop/$svc
        done
    else
        echo "Flux deployment exists. Skipping. Delete the aws key microservices-demo-flux to redeploy."
    fi

-->

If all went successfully then your demo is set up to automatically deploy changes made to the sock shop microservices app by monitoring the docker registry for changes. If you would like more information or to run your own version head on over to <a href="https://www.weave.works/guides/cloud-guide-part-2-deploy-continuous-delivery/" target="_blank">this</a> handy guide provided by the wonderful people at Weaveworks.

### View the results
Each app is behind a load balancer which delivers the app from one of 3+ nodes

Run `terraform output` again to see the load balancer URLs
```
Outputs:

master_address = ec2-52-49-12-162.eu-west-1.compute.amazonaws.com
node_addresses = [
    ec2-52-212-222-97.eu-west-1.compute.amazonaws.com,
    ec2-52-51-218-57.eu-west-1.compute.amazonaws.com,
    ec2-52-18-214-169.eu-west-1.compute.amazonaws.com
]
scope_address = elb-scope-927708071.eu-west-1.elb.amazonaws.com
sock_shop_address = elb-sock-shop-363710645.eu-west-1.elb.amazonaws.com
```

Open any of the links listed in scope_address and sock_shop_address to see the apps in action.
It may take a few moments for the apps to get running.

### Run tests

There is a seperate load-test available to simulate user traffic to the application. For more information see [Load Test](#loadtest).
This will send some traffic to the application, which will form the connection graph that you can view in Scope or Weave Cloud.

```
    elb_url=$(aws elb describe-load-balancers --load-balancer-name MD-k8s-elb-sock-shop | jq -r '.LoadBalancerDescriptions[0].DNSName')
    docker run --rm weaveworksdemos/load-test -d 300 -h $elb_url -c 3 -r 10
```

<!-- deploy-doc-hidden run-tests
    master_ip=$(aws ec2 describe-instances -\-filter "Name=tag:Name,Values=MD-k8s-master" "Name=instance-state-name,Values=running"| jq -r '.Reservations[0].Instances[0].PublicDnsName')
    elb_url=$(aws elb describe-load-balancers -\-load-balancer-name MD-k8s-elb-sock-shop | jq -r '.LoadBalancerDescriptions[0].DNSName')

    docker run -\-rm weaveworksdemos/load-test -d 300 -h $elb_url -c 3 -r 10

    scp -i ~/.ssh/microservices-demo_id_rsa -o StrictHostKeyChecking=no /root/healthcheck.sh ubuntu@$master_ip:/home/ubuntu
    ssh -i ~/.ssh/microservices-demo_id_rsa ubuntu@$master_ip "chmod +x /home/ubuntu/healthcheck.sh; ./healthcheck.sh"

    if [ $? -ne 0 ]; then
        exit 1;
    fi

-->

### Uninstall App

Remove all deployments (will also remove pods)
```
ssh -i ~/.ssh/microservices-demo_id_rsa ubuntu@$master_ip kubectl delete deployments --all
```
Remove all services, except kubernetes
```
ssh -i ~/.ssh/microservices-demo_id_rsa ubuntu@$master_ip kubectl delete service $(kubectl get services | cut -d" " -f1 | grep -v NAME | grep -v kubernetes)
```

Destroying the entire infrastructure
    
```
terraform destroy -force deploy/kubernetes/terraform/
rm terraform.tfstate
rm terraform.tfstate.backup
rm ~/.ssh/microservices-demo_id_rsa
rm k8s-init.log
rm join.cmd
```
<!-- deploy-doc-hidden destroy-infrastructure

    if [ ! -f /root/deploy-exists ]; then
        rm ~/.ssh/microservices-demo_id_rsa
        rm k8s-init.log
        rm join.cmd
    else
        echo "Flux deployment exists. Skipping. Delete the aws key microservices-demo-flux to redeploy."
    fi

-->
