---
layout: default
deployDoc: true
---

## Sock Shop on Kubernetes + Weave 

This demo demonstrates running the Sock Shop on a Kubernetes cluster using 
Weave Net and Weave Scope.

### Pre-requisites
* *Optional* [Terraform](https://www.terraform.io/downloads.html)
* *Optional* [AWS Account](https://aws.amazon.com/)
* *Optional* [awscli](http://docs.aws.amazon.com/cli/latest/userguide/installing.html)

<!-- deploy-test require-env AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_DEFAULT_REGION -->
<!-- deploy-test-start pre-install -->

    apt-get install -yq jq python-pip curl unzip
    pip install awscli
    curl -o /tmp/terraform.zip https://releases.hashicorp.com/terraform/0.7.11/terraform_0.7.11_linux_amd64.zip 
    unzip /tmp/terraform.zip -d /usr/bin

<!-- deploy-test-end -->

```
git clone https://github.com/microservices-demo/microservices-demo 
```
<!-- deploy-test-hidden pre-install 

    cat > /root/healthcheck.sh <<-EOF
#!/usr/bin/env bash
kubectl run -\-namespace=sock-shop healthcheck -\-image=andrius/alpine-ruby sleep 10000
sleep 90
kube_id=\$(kubectl get pods -\-namespace=sock-shop | grep healthcheck | awk '{print \$1}')
kubectl exec -\-namespace=sock-shop \$kube_id -\- sh -c "curl -o healthcheck.rb \"https://raw.githubusercontent.com/microservices-demo/microservices-demo/master/deploy/healthcheck.rb\"; chmod +x ./healthcheck.rb; ./healthcheck.rb -s user,catalogue,queue-master,cart,shipping,payment,orders"

EOF

    mkdir -p ~/.ssh/
    aws ec2 describe-key-pairs -\-key-name deploy-docs-k8s &>/dev/null
    if [ $? -eq 0 ]; then aws ec2 delete-key-pair -\-key-name deploy-docs-k8s; fi
-->
### Setup Kubernetes

Begin by setting the appropriate AWS environment variables.
```
export AWS_ACCESS_KEY_ID=[YOURACCESSKEYID]
export AWS_SECRET_ACCESS_KEY=[YOURSECRETACCESSKEY]
export AWS_DEFAULT_REGION=[YOURDEFAULTREGION]
```

Next we'll create a private key for use during this demo.    

<!-- deploy-test-start create-infrastructure -->

    aws ec2 create-key-pair --key-name deploy-docs-k8s --query 'KeyMaterial' --output text > ~/.ssh/deploy-docs-k8s.pem
    chmod 600 ~/.ssh/deploy-docs-k8s.pem

<!-- deploy-test-end -->

Finally run terraform.

<!-- deploy-test-start create-infrastructure -->

    terraform apply deploy/kubernetes/terraform/

<!-- deploy-test-end -->

Our master node makes use of some of the files in this repo so lets securely copy those over.

<!-- deploy-test-start create-infrastructure -->

    master_ip=$(terraform output -json | jq -r '.master_address.value') 
    scp -i ~/.ssh/deploy-docs-k8s.pem -o StrictHostKeyChecking=no deploy/kubernetes/weavescope.yaml ubuntu@$master_ip:/tmp/
    scp -i ~/.ssh/deploy-docs-k8s.pem -rp deploy/kubernetes/manifests ubuntu@$master_ip:/tmp/

<!-- deploy-test-end -->

### <a name="weavenet"></a>Setup Weave Net
* Run the following commands to setup Kubernetes and Weave Net on the master instance

<!-- deploy-test-start create-infrastructure -->

    master_ip=$(terraform output -json | jq -r '.master_address.value') 
    ssh -i ~/.ssh/deploy-docs-k8s.pem ubuntu@$master_ip sudo kubeadm init > k8s-init.log
    grep -e --token k8s-init.log > join.cmd
    ssh -i ~/.ssh/deploy-docs-k8s.pem ubuntu@$master_ip kubectl apply -f https://git.io/weave-kube

<!-- deploy-test-end -->

### Time for the nodes to join the master 
* Run the following commands to SSH into each node\_addresses listed when you ran the ```terraform output``` command and run the ```kubeadm join --token <token> <master-ip>``` command from before.

<!-- deploy-test-start create-infrastructure -->

    node_addresses=$(terraform output -json | jq -r '.node_addresses.value|@sh' | sed -e "s/'//g" ) 
    for node in $node_addresses; do
        ssh -i ~/.ssh/deploy-docs-k8s.pem -o StrictHostKeyChecking=no ubuntu@$node sudo `cat join.cmd`
    done

<!-- deploy-test-end -->

### Setup Weave Scope
* SSH into the master node
* Start weave scope on the cluster

<!-- deploy-test-start create-infrastructure -->

    master_ip=$(terraform output -json | jq -r '.master_address.value') 
    ssh -i ~/.ssh/deploy-docs-k8s.pem ubuntu@$master_ip kubectl apply -f /tmp/weavescope.yaml -\-validate=false

<!-- deploy-test-end -->

### Deploy Sock Shop
* SSH into the master node
* Deploy the sock shop

<!-- deploy-test-start create-infrastructure -->

    master_ip=$(terraform output -json | jq -r '.master_address.value') 
    ssh -i ~/.ssh/deploy-docs-k8s.pem ubuntu@$master_ip kubectl apply -f /tmp/manifests/sock-shop-ns.yml -f /tmp/manifests

<!-- deploy-test-end -->

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

<!-- deploy-test-start run-tests -->

    elb_url=$(terraform output -json | jq -r '.sock_shop_address.value') 
    docker run --rm weaveworksdemos/load-test -d 300 -h $elb_url -c 3 -r 10

<!-- deploy-test-end -->

<!-- deploy-test-hidden run-tests

    master_ip=$(terraform output -json | jq -r '.master_address.value') 
    scp -i ~/.ssh/deploy-docs-k8s.pem -rp /root/healthcheck.sh ubuntu@$master_ip:/home/ubuntu
    ssh -i ~/.ssh/deploy-docs-k8s.pem ubuntu@$master_ip "chmod +x /home/ubuntu/healthcheck.sh; ./healthcheck.sh"

    if [ $? -ne 0 ]; then
        exit 1;
    fi

-->

### Uninstall App

Remove all deployments (will also remove pods)
```
ssh -i ~/.ssh/deploy-docs-k8s.pem ubuntu@$master_ip kubectl delete deployments --all
```
Remove all services, except kubernetes
```
ssh -i ~/.ssh/deploy-docs-k8s.pem ubuntu@$master_ip kubectl delete service $(kubectl get services | cut -d" " -f1 | grep -v NAME | grep -v kubernetes)
```

Destroying the entire infrastructure

<!-- deploy-test-start destroy-infrastructure -->

    terraform destroy -force deploy/kubernetes/terraform/
    aws ec2 delete-key-pair -\-key-name deploy-docs-k8s
    rm ~/.ssh/deploy-docs-k8s.pem
    rm terraform.tfstate
    rm terraform.tfstate.backup
    rm k8s-init.log
    rm join.cmd

<!-- deploy-test-end -->
