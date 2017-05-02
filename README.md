# Kubernetes on AWS care of Terraform and Ansible

A worked example to provision a Kubernetes cluster on AWS from scratch, using
Terraform and Ansible. A scripted version of the famous tutorial
[Kubernetes the hard way](https://github.com/kelseyhightower/kubernetes-the-hard-way).

See the companion article https://opencredo.com/kubernetes-aws-terraform-ansible-1/
for details about goals, design decisions and simplifications.

- AWS VPC
- 3 EC2 instances for HA Kubernetes Control Plane: Kubernetes API, Scheduler and Controller Manager
- 3 EC2 instances for *etcd* cluster
- 3 EC2 instances as Kubernetes Workers (aka Minions or Nodes)
- Kubenet Pod networking (using CNI)
- HTTPS between components and control API
- Sample *nginx* service deployed to check everything works


## Install Tools on Laptop

Your laptop will act as the control machine, you will need to install these
tools:

- Terraform (tested with Terraform 0.9.3; **NOT compatible with Terraform 0.6.x**)
- Python (tested with Python 2.7.9, may be not compatible with older versions; requires Jinja2 2.8)
- Create a [virtualenv](https://virtualenv.pypa.io/en/stable/) for your Python env
  - Python netaddr module (pip install netaddr)
  - Ansible `pip install ansible` (tested with Ansible 2.3.0.0)
  - Python boto `pip install boto`
- [*cfssl* and *cfssljson*](https://github.com/cloudflare/cfssl)
- [Kubernetes CLI](https://kubernetes.io/docs/tasks/kubectl/install/)
- SSH Agent


## AWS Credentials

You will need a EC3 key pair for SSH access to hosts and access key id and
secret access key for api access.


### AWS KeyPair

You need a valid AWS Identity (`.pem`) file and the corresponding Public Key.
Please read [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#how-to-generate-your-own-key-and-import-it-to-aws)
about supported formats. If you don't have a keypair create or import one.

Use Terraform to import the [KeyPair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html)
from your AWS account. For example with a key called k8s-machine
`terraform import aws_key_pair.default_keypair k8s-machines`

Later Ansible uses the Identity to SSH into machines.

Ansible will use the SSH identity loaded by SSH agent, add the PEM file from
creating a key pair:
```
$ ssh-add <keypair-name>.pem
```

You will need to generate the public key also which will go in the
terraform.tfvars file in the next section. [This article](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#retrieving-the-public-key)
from AWS explains how to generate the public key from private key.

### AWS API Credentials

Terraform and Ansible need to authenticate with AWS apis, so you need to setup
access key id and secret access key.

The easiest solution is environment variables:
```
$ export AWS_ACCESS_KEY_ID=<access-key-id>
$ export AWS_SECRET_ACCESS_KEY="<secret-key>"
```

If you plan to use AWS CLI you have to set `AWS_DEFAULT_REGION`.

## Configuring Terraform

Terraform requires some variables be defined:

- `control_cidr`: Instances will accept traffic from only this CDIR. e.g. `123.45.67.89/32` (mandatory). Use your laptops public IP.
- `default_keypair_public_key`: see AWS Credentials section on generating this if needed
- `default_keypair_name`: name of EC2 keypair

**Note that Instances and Kubernetes API will be accessible only from the
"control IP"**. If you fail to set it correctly, you will not be able to SSH into machines or run Ansible playbooks.

You may optionally redefine:

- `default_keypair_name`: Fill in name you used when setting up key pair above (Default: "k8s-sandbox")
- `vpc_name`: VPC Name. Must be unique in the AWS Account (Default: "kubernetes")
- `elb_name`: ELB Name for Kubernetes API. Can only contain characters valid for DNS names. Must be unique in the AWS Account (Default: "kubernetes")
- `owner`: `Owner` tag added to all AWS resources. Can be used to filter.

Create a `terraform.tfvars` [variable file](https://www.terraform.io/docs/configuration/variables.html#variable-files)
in `./terraform` directory. Terraform automatically imports it.

Sample `terraform.tfvars`:
```
default_keypair_public_key = "ssh-rsa AAA...zzz"
control_cidr = "123.45.67.89/32"
default_keypair_name = "k8s"
vpc_name = "sre-k8s vpc"
elb_name = "sre-k8s-elb"
owner = "Site Reliabilty"
```


#### Changing AWS Region

By default, the project uses `us-west-2`. To use a different AWS Region, set additional Terraform variables:

- `region`: AWS Region (default: "us-west-2").
- `zone`: AWS Availability Zone (default: "us-west-2a")
- `default_ami`: Pick the AMI for the new Region from https://cloud-images.ubuntu.com/locator/ec2/: Ubuntu 16.04 LTS (xenial), HVM:EBS-SSD

You also have to edit `./ansible/hosts/ec2.ini`, changing `regions = us-west-2` to the new Region.

## Provision infrastructure, with Terraform

Run Terraform commands from `./terraform` subdirectory.

`$ terraform plan` and if all goes well you can `$ terraform apply`


When done Terraform outputs public DNS name of Kubernetes API and Workers
public IPs, you will need these later:
```
Apply complete! Resources: 12 added, 2 changed, 0 destroyed.
  ...
Outputs:

  kubernetes_api_dns_name = k8s-sandbox-1444286147.us-west-2.elb.amazonaws.com
  kubernetes_workers_public_ip = 34.209.17.129,54.69.111.35,34.223.206.129
```

you may show them at any moment with `terraform output`.


### Generated SSH config

Terraform generates `ssh.cfg`, SSH configuration file in the project directory.
It allows using node names when SSHing:

e.g.
```
$ ssh -F ssh.cfg worker0
```

Note this file is not used by Terraform or Ansible.

## Install Kubernetes, with Ansible

Run Ansible commands from `./ansible` subdirectory.

We have multiple playbooks that will need to be run to finish setting up your
cluster.


### Install and set up Kubernetes cluster

Install Kubernetes components and *etcd* cluster.
```
$ ansible-playbook infra.yaml
```

NOTE: If Terraform complains about finding cfssl(json) you probably are using
~ rather than $HOME when defining locations in $PATH. Either use $HOME or use
the full path to the location containing cfssl(json).


### Configure Kubernetes CLI on Laptop

Configure Kubernetes CLI (`kubectl`) on your machine, setting Kubernetes API
endpoint (as returned by Terraform).
```
$ ansible-playbook kubectl.yaml --extra-vars "kubernetes_api_endpoint=<kubernetes-api-dns-name>"
```

You can use `terraform output` to get the api dns name needed above.

Now verify all components and minions (workers) are up and running, using
Kubernetes CLI (`kubectl`).

```
$ kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}

$ kubectl get nodes
NAME                                       STATUS    AGE
ip-10-43-0-30.eu-west-1.compute.internal   Ready     6m
ip-10-43-0-31.eu-west-1.compute.internal   Ready     6m
ip-10-43-0-32.eu-west-1.compute.internal   Ready     6m
```


### Setup Pod cluster routing

Set up additional routes for traffic between Pods.
```
$ ansible-playbook kubernetes-routing.yaml
```


### Smoke test: Deploy *nginx* service

Deploy a *ngnix* service inside Kubernetes.
```
$ ansible-playbook kubernetes-nginx.yaml
```

Verify pods and service are up and running.

```
$ kubectl get pods -o wide
NAME                     READY     STATUS    RESTARTS   AGE       IP           NODE
nginx-2032906785-9chju   1/1       Running   0          3m        10.200.1.2   ip-10-43-0-31.eu-west-1.compute.internal
nginx-2032906785-anu2z   1/1       Running   0          3m        10.200.2.3   ip-10-43-0-30.eu-west-1.compute.internal
nginx-2032906785-ynuhi   1/1       Running   0          3m        10.200.0.3   ip-10-43-0-32.eu-west-1.compute.internal

> kubectl get svc nginx --output=json
{
    "kind": "Service",
    "apiVersion": "v1",
    "metadata": {
        "name": "nginx",
        "namespace": "default",
...
```

Retrieve the port *nginx* has been exposed on:

```
$ kubectl get svc nginx --output=jsonpath='{range .spec.ports[0]}{.nodePort}'
```

and get the public DNS name with

```
$ ansible/hosts/ec2.py --host worker0 | grep ec2_public_dns_name
```

Now you should be able to access *nginx* default page:

```
$ curl http://<worker-0-public-ip>:<exposed-port>
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

The service is exposed on all Workers using the same port (see Workers public IPs in Terraform output).


# Known simplifications

There are many known simplifications, compared to a production-ready solution:

- Networking setup is very simple: ALL instances have a public IP (though only accessible from a configurable Control IP).
- Infrastructure managed by direct SSH into instances (no VPN, no Bastion).
- Very basic Service Account and Secret (to change them, modify: `./ansible/roles/controller/files/token.csv` and `./ansible/roles/worker/templates/kubeconfig.j2`)
- No actual integration between Kubernetes and AWS.
- No additional Kubernetes add-on (DNS, Dashboard, Logging...)
- Simplified Ansible lifecycle. Playbooks support changes in a simplistic way, including possibly unnecessary restarts.
- Instances use static private IP addresses
- No stable private or public DNS naming (only dynamic DNS names, generated by AWS)
