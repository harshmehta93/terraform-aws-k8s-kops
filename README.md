# Create Kubernetes cluster with Kops in existing VPC on AWS with Terraform

- Terraform is an infrastructure as a code software used to create different cloud infrastructures on-the-go as and when needed.
- It allows users to define a datacenter infrastructure in a high-level configuration language, from which it can create an execution plan to build the infrastructure such as OpenStack or in a service provider such as AWS, IBM Cloud, Google Cloud Platform etc.
- Infrastructure is defined in a HCL Terraform syntax or JSON format.
- This project creates Kubernetes cluster with Kops on AWS cloud in existing VPC with Terraform.
- So let's get started...

### Create a VPC with Terraform
- To proceed ahead with creating Kubernetes cluster on AWS, first we need an existing VPC infrastructure to work with. We have Terraform modules(inside /modules directory) that will let us easily create a VPC with public / private subnet pairs across multiple availability zones. It will also create NAT gateways to allow outbound internet traffic for instances on the private subnets.
- In variables.tf file, we need to set the name variable which should be set to the domain name we are going to be using for this cluster.
- Optionally, we can configure the region and availability zone variables. By default, we are going to be creating a highly available cluster with Kubernetes masters in us-east-1a, us-east-1c, us-east-1d. You can also configure the env and vpc_cidr variables, if desired.
- Let’s take a look at main.tf. Here is how we define our VPC:
```
module "vpc" {
  source   = "./modules/vpc"
  name     = "${var.name}"
  env      = "${var.env}"
  vpc_cidr = "${var.vpc_cidr}"

  tags {
    Infra             = "${var.name}"
    Environment       = "${var.env}"
    Terraformed       = "true"
    KubernetesCluster = "${var.env}.${var.name}"
  }
}

module "subnet_pair" {
  source              = "./modules/subnet-pair"
  name                = "${var.name}"
  env                 = "${var.env}"
  vpc_id              = "${module.vpc.vpc_id}"
  vpc_cidr            = "${module.vpc.cidr_block}"
  internet_gateway_id = "${module.vpc.internet_gateway_id}"
  availability_zones  = "${var.azs}"

  tags {
    Infra             = "${var.name}"
    Environment       = "${var.env}"
    Terraformed       = "true"
    KubernetesCluster = "${var.env}.${var.name}"
  }
}
```
- Most of the heavy lifting is done in the vpc and subnet-pair modules. Those modules are responsible for creating the VPC, private and public subnets, NAT Gateways, routes, and security groups.
- One thing to note is that we are setting various tags on our resources. These tags are required by some of the Kubernetes AWS integration features (such as creating a LoadBalancer service that is backed by an ELB).
- Besides the networking infrastructure, we also need to create the hosted zone for our cluster domain name in Route53. If you already have your domain name registered in Route53, you can remove this resource from your local configuration.
```
resource "aws_route53_zone" "public" {
  name          = "${var.name}"
  force_destroy = true
  ...
}
```
Finally, Kops also requires an S3 bucket for storing the state of the cluster. We create this bucket as part of our Terraform configuration:
```
resource "aws_s3_bucket" "state_store" {
  bucket        = "${var.name}-state"
  acl           = "private"
  force_destroy = true

  versioning {
    enabled = true
  }
  ...
}
```
- Let’s go ahead and create our infrastructure. We will need to provide credentials for an IAM user that has sufficient privileges to create all of these resources. For simplicity, I am using a user that has the following policies associated:
  - AmazonEC2FullAccess
  - IAMFullAccess
  - AmazonS3FullAccess
  - AmazonVPCFullAccess
  - AmazonRoute53FullAccess
```
export AWS_ACCESS_KEY_ID=<access key>
export AWS_SECRET_ACCESS_KEY=<secret key>

terraform get
terraform apply
```
- The apply may take a few minutes but when it’s done, we should have a new VPC and associated resources in your AWS account.

### Deploy Kubernetes with Kops and Terraform
- Pre-requisites:
  - Kops and Kubectl must be installed before proceed ahead with deployment.
- At this point, we have our base AWS infrastructure up and running. Now, we can move on to using Kops to generate the Terraform for our Kubernetes cluster.
- First, we should export a few environment variables that we will be using in our Kops commands.
```
export NAME=$(terraform output cluster_name)
export KOPS_STATE_STORE=$(terraform output state_store)
export ZONES=us-east-1a,us-east-1c,us-east-1d
```
- The $NAME and $KOPS_STATE_STORE variables are populated by our Terraform outputs. $NAME should be set to <env>.<yourdomain.com> and $KOPS_STATE_STORE should be s3://<yourdomain.com>-state.
- The $ZONES variable should set to the same availability zones that we are using in variables.tf.
- Now we can run Kops. Here is the command we will use to create our cluster:

```
  kops create cluster \
    --master-zones $ZONES \
    --zones $ZONES \
    --topology private \
    --dns-zone $(terraform output public_zone_id) \
    --networking calico \
    --vpc $(terraform output vpc_id) \
    --target=terraform \
    --out=. \
    ${NAME}
```
- When you run this command, Kops does several things including:
1. Populating the KOPS_STATE_STORE S3 bucket with the Kubernetes cluster configuration.
2. Creating several record sets in the Route53 hosted zone for your domain (for Kubernetes APIs and etcd).
3. Creating IAM policy files, user data scripts, and an SSH key in the ./data directory.
4. Generating a Terraform configuration for all of the Kubernetes resources. This will be saved in a file called kubernetes.tf.
- The kubernetes.tf includes all of the resources required to deploy the cluster. However, we are not ready to apply this yet as it will want to create new subnets, routes, and NAT gateways. We want to deploy Kubernetes in our existing subnets.
- Before we run terraform apply, we need to edit the cluster configuration so that Kops knows about our existing network resources.
- The kops edit cluster command will open your $EDITOR with your cluster settings in YAML format. We need to replace the subnets section with our existing vpc and subnet information.
```
kops edit cluster ${NAME}
```
- Your subnets map should look something like this:
```
subnets:
- cidr: 10.20.32.0/19
  name: us-east-1a
  type: Private
  zone: us-east-1a
- cidr: 10.20.64.0/19
  name: us-east-1c
  type: Private
  zone: us-east-1c
- cidr: 10.20.96.0/19
  name: us-east-1d
  type: Private
  zone: us-east-1d
- cidr: 10.20.0.0/22
  name: utility-us-east-1a
  type: Utility
  zone: us-east-1a
- cidr: 10.20.4.0/22
  name: utility-us-east-1c
  type: Utility
  zone: us-east-1c
- cidr: 10.20.8.0/22
  name: utility-us-east-1d
  type: Utility
  zone: us-east-1d
```
- There should be one Private type subnet and one Utility (public) type subnet in each availability zone. We need to modify this section by replacing each cidr with the corresponding existing subnet ID for that region. For the Private subnets, we also need to specify our NAT gateway ID in an egress key.
- After you edit and save your cluster configuration with the updated subnets section, Kops updates the cluster configuration stored in the S3 state store. However, it will have not yet updated the kubernetes.tf file. To do that, we need to run kops update cluster:
```
kops update cluster \
  --out=. \
  --target=terraform \
  ${NAME}
```
- If you look at the updated kubernetes.tf, you will see that it references our existing VPC infrastructure instead of creating new resources.
- Now we can actually build our Kubernetes cluster. Run terraform plan to make sure everything looks sane and then run terraform apply.
- After the apply finishes, it will take another few minutes for the Kubernetes cluster to initialize and become healthy. But, eventually, we will have a working, highly-available Kubernetes cluster!
```
$ kubectl get nodes
NAME                            STATUS         AGE
ip-10-20-101-252.ec2.internal   Ready,master   7m
ip-10-20-103-232.ec2.internal   Ready,master   7m
ip-10-20-103-75.ec2.internal    Ready          5m
ip-10-20-104-127.ec2.internal   Ready,master   6m
ip-10-20-104-6.ec2.internal     Ready          5m
```

### Cleaning up
- If we want to delete all of the infrastructure created in this post, we just have to run terraform destroy.
- If you used a different S3 bucket for your $KOPS_STATE_STORE, you may also want to run kops delete cluster to remove the Kops state. Otherwise, the entire S3 bucket will be destroyed along with the rest of the infrastructure.
