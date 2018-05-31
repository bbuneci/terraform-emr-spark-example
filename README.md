# Spark on Amazon EMR + Terraform 
An example Terraform project that will configure a Secure and Customizable 
Spark Cluster on Amazon EMR.  Zeppelin is also installed as an interface to 
Spark, and Ganglia is also installed for monitoring.

## Architecture

This project leverages [Terraform Modules](https://www.terraform.io/docs/modules/index.html), 
and relies heavily on the [EMR Cluster](https://www.terraform.io/docs/providers/aws/r/emr_cluster.html) 
resource.

The layout of this project is as follows:

```
modules/
    bootstrap/  
        --> This module copies files to S3 so EMR can run a script right after
            the EC2 instances are provisioned
    emr/
        --> This module creates the EMR cluster and configuration files
    lb/
        --> This module creates a Load Balancer so Zepplein is accessible from
            your system
    s3/
        --> This module creates S3 buckets needed by EMR
    sec/
        --> This module creates some Security Fundamentals needed by EMR.
            NOTE: Please DO NOT USE THIS in production, its in place purely 
            for demonstrative purposes.  See "Security Module" below.
    sgs/
        --> This module creates the Security Groups needed for EMR and the Load
            Balancers.
```

## Building
Before building, ensure you're comfortable with [how terraform works](https://www.terraform.io/intro/index.html).

### Pre-Requisities
Terraform will use the AWS credentials provided in your shell environment.  You
will need an AWS user account available that has the following permissions:
 - View/Create/Update/Delete IAM, KMS, and Certificates
 - View/Create/Update/Delete S3 Buckets and Objects
 - View/Create/Update/Delete Security Groups
 - View/Create/Update/Delete Load Balancers
 - View VPC Subnets

### Initalization

Before executing your first plan, you need to initialize Terraform:
```
$> terraform init
```

### Planning

Terraform allows you to "Plan", which allows you to see what it would change
without actually making any changes.

```
$> terraform plan -var 'vpc_id=vpc-abcde123' -var 'cluster_name=my_emr_cluster_1' 
```

### Applying

Finally, affer initialization, planning, you can apply your chages, which will
actaully create or update your cluster, based on the plan:

```
$> terraform apply -var 'vpc_id=vpc-abcde123' -var 'cluster_name=my_emr_cluster_1'
```

### Destroying

If you want Terraform to clean up anything it made, you can destroy the
cluster:

```
$> terraform destroy -var 'vpc_id=vpc-abcde123' -var 'cluster_name=my_emr_cluster_1'
```

## Security Module

This project provides a [Security Module](modules/sec) that is intendended to
be replaced with the security requirements of your organization.  PLEASE DO NOT
use this Security Module in production.  It is there for demonstrative purposes
only, but can be a guide on what needs to be replaced to use this project in
production.


It provides bootstrapping for the following:
 - IAM Roles
   - Three IAM roles are created, one for EMR to create infrastructure, a
     role (instance profile) to attach to each one of the EC2 instances, and a
     role for EMR to use for autoscaling.
 - Network
   - Whitelisting: Instead of opening your EMR cluster to the world, this
     module will look up your public IP using `ifconfig.co` and adding that
     IP to a few security groups (SSH for the Nodes and HTTPS for the LB)
   - Subnets: This module will fetch all subnets in the VPC provided as a 
     variable, it will spin up EMR in the first, and it will attach the LB
     to the rest.  You will likely want to specify which subnets to use for 
     both.
  - SSH
    - It will create and save a SSH key to connect to the cluster.  The private
    SSH key is saved in `generated/ssh/`.
  - Zeppelin
    - Zeppelin is equipped with a Load Balancer for (easier) access.  It also
      is equipped with a Self-Signed Certificate for encrypted communication
      between the ELB and the Zeppelin process, and another Self-Signed 
      Certifcate for the Internet.  You should not use Self-Signed Certificates
      in Production, and switch these out with valid certificates.

## FAQs

### I've changed something in `bootstrap`, why didn't that get applied?

Terraform isn't the best at tracking changes to resources in the `bootstrap`
module.  Sometimes you have to let Terraform know you need to destroy and 
recreate the EMR cluster by executing the following command:

```
$> terraform taint -var 'vpc_id=vpc-abcde123' -var 'cluster_name=my_emr_cluster_1' -module=emr aws_emr_cluster.cluster
$> terraform apply -var 'vpc_id=vpc-abcde123' -var 'cluster_name=my_emr_cluster_1'
```

### I want this to run in another region, what do I do?

Currently this project is hard-coded to run in AWS US-West-2 (Oregon), and
there are two things you have to change to make it run in another region:
 - [Default Region](variables.tf#L4)
   - Change this to match the name of the region you'd like to use
 - [SNS Source IPs](variables.tf#L8)
   - Change this list to make it match the [Source IPs](https://forums.aws.amazon.com/ann.jspa?annID=2347)
     of Amazon SNS in the region you'd like to use.

### How do I login to Zeppelin?

After the cluster builds, it will output the DNS Name of the Load Balancer that
was created:

```
$> terraform apply...
...
Outputs:

dns_name = my_emr_cluster_1-default-1234567890.us-west-2.elb.amazonaws.com
```

You can navigate to `https://my_emr_cluster_1-default-1234567890.us-west-2.elb.amazonaws.com` 
in your browser.  You will have to ignore the certificate warning, since this
example project creates self-signed SSL certs for demonstrative purposes.

Finally you can find the Username and Password for Zepplein, hard-coded in the
[Shiro Configuration File](modules/bootstrap/templates/shiro.ini.tpl#L2-L3).

NOTE: Please DO NOT hard code your Usernames and Passwords for Zeppelin in
production, or check them into Git.  This is in place purely for demonstrative 
purposes.  Zeppelin offers a few [Authentication Options](https://zeppelin.apache.org/docs/latest/security/shiroauthentication.html).

### How do I SSH into this cluster?

For demonstrative purposes, SSH is allowed to the public IP of the system that
runs Terraform.

Also, this project will create an SSH key to connecto to the cluster.  After
Terraform is applied, the SSH key generated is placed in the `generated/` 
folder, so you can SSH into the cluster with the following command:

```
$> ssh -i generated/ssh/my_emr_cluster_1-default ip_address_of_a_node
```

AWS also provides additional SSH connection help in the [EMR Console](https://us-west-2.console.aws.amazon.com/elasticmapreduce/home)

### The Terraform state file is saved locally, how do I share that with others?

It is recommended to use [Terraform Remote State](https://www.terraform.io/docs/state/remote.html)

### The cluster failed to create, where do I find logs?

 - First, check the [EMR Console](https://us-west-2.console.aws.amazon.com/elasticmapreduce/home)
 - Second, there are logs in the S3 bucket created, under `logs/`

### How do I test that everything is working?

 - You can visit the Zeppelin UI (See the "How do I login to Zeppelin?" FAQ above)
 - Second, follow the [Zeppelin Tutorial](https://zeppelin.apache.org/docs/0.6.1/quickstart/tutorial.html)
