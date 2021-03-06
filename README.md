# Vault AWS Module

This repo contains a module for how to deploy a [Vault](https://www.vaultproject.io/) cluster on [AWS](https://aws.amazon.com/) using [Terraform](https://www.terraform.io/). It follows the patterns laid out in the [Vault Reference Architecture](https://learn.hashicorp.com/vault/operations/ops-reference-architecture) and the [Vault Deployment Guide](https://www.vaultproject.io/guides/operations/deployment-guide.html).

## Contents

* A Terraform module to install Vault into AWS.
* A packer directory containing the packer code to build Vault and Consul AMIs
* The module can be used with the Packer built AMIs or user data to configure up to the standard listed in the [Deployment Guide](https://www.vaultproject.io/guides/operations/deployment-guide.html).

This module is specifically designed to deliver the [Reference Architecture](https://learn.hashicorp.com/vault/operations/ops-reference-architecture) and as such adheres to that pattern. This means that it has these specifics that are not configurable:
* A Vault cluster using TLS
* The Vault cluster is backed by a Consul cluster for storage
* The Vault cluster is intended to be deployed into a _private_ subnet(s)
* The security groups attached to the Vault and Consul cluster members are non-configurable and allow for full functionality (see below)

The module has these specifics that are configurable:
* The Vault cluster can be set up as _n_ standalone instances or inside an ASG depending on your preference.
* The Vault cluster can be fronted by an _internal_ or _external_ ELB or not, depending on your preference.  
* The number of Vault nodes can be configured, though 3 is the default and recommended number
* The number of Consul nodes can be configured though 5 is the default, 5 as per the recommended architecture.
* The recommended architecture suggests that the 3 Vault nodes be spread across 3 separate availability zones, but this module will take fewer than that.
* While this module manages the security groups to allow for the correct function of the cluster, it is possible to add further security groups to the instances if other connectivity is desired.
* The module can be set to install and configure Vault and Consul via user data or this can be turned off.
* The module contains packer scripts to build Vault and Consul AMIs as per the deployment guide if required so that you have the option of using user data, using the module AMI or building your own AMI.
* The Vault cluster can be set up to use AWS KMS key for auto unseal (something that is recommended if using an ASG)

## Versions
This module is written for Terraform 11.10 and has been tested with this version only.
This module has been tested with Ubuntu 18.04 and Centos 7.x OS
This has been tested with Vault 1.x
This has been tested with Consul 1.3.x. The configuration and user data setup will not work for Consul 1.4.x currently

## Setup
This module is to deliver a Vault cluster on Linux with systemd on AWS.

## Usage
```hcl
module vault_cluster {

  # This source should be the tag for this module if pulling from git
  source                 = "../terraform-aws-vault/vault-cluster"

  instance_type          = "${var.instance_type}"
  ssh_key_name           = "${var.ssh_key_name}"
  cluster_name           = "${var.cluster_name}"
  vault_ami_id           = "${var.ami_id}"
  consul_ami_id          = "${var.ami_id}"
  private_subnets        = "${module.vpc.private_subnets}"
  public_subnets         = "${module.vpc.public_subnets}"
  vpc_id                 = "${module.vpc.vpc_id}"
  availability_zones     = "${var.availability_zones}"
  cluster_name           = "${var.cluster_name}"
  vault_ami_id           = "${var.ami_id}"
  consul_ami_id          = "${var.ami_id}"
  use_asg                = true
  use_elb                = true
  internal_elb           = false
  use_auto_unseal        = true
  aws_region             = "${var.global_region}"
}
```
## Variables
### Required Input Variables
* cluster_name      - name of your cluster
* vault_ami_id      - The AMI id for the Vault cluster server instances. This can be the same as the consul_ami_id if you are using the user data install method.
* consul_ami_id     - The AMI id for the Consul cluster server instances
* instance_type - The AWS instance type you wish for the Vault and Consul servers
* ssh_key_name      - The AWS key-pair name to use for the instances
* private_subnets   - a list of the private subnets the cluster will be installed to. This can be from 1 to `var.consul_cluster_size`. This defaults to 3
* public_subnets    - - a list of the public subnets the ELB will be installed to.
* availability_zones - The availability zones that the cluster will be installed in. This should match up with the private_subnets list so that there is at least 1 subnet in each AZ.
* vpc_id            - The AWS VPC id

### Optional Input Variables
These are listed below with their defined defaults. If you wish to change the variable value then define it in the code block for the module. see the `variables.tf` file for descriptions

* use_asg (false)
* use_elb (false)
* use_userdata (false)
* use_auto_unseal (false)
* internal_elb (true)
* vault_cluster_size (3)
* consul_cluster_size (5)
* health_check_grace_period (300)
* wait_for_capacity_timeout (10m)
* enabled_metrics ([])
* termination_policies (Default)
* cross_zone_load_balancing (true)
* idle_timeout (60)
* connection_draining (true)
* connection_draining_timeout (300)
* lb_port (8200)
* vault_api_port (8200)
* health_check_protocol (HTTPS)
* health_check_path (/v1/sys/health)
* health_check_interval (15)
* health_check_healthy_threshold (2)
* health_check_unhealthy_threshold (2)
* health_check_timeout (5)
* install_bucket (my_bucket)
* vault_bin (vault.zip)
* key_pem (key.pem)
* cert_pem (cert.pem)
* consul_version (1.3.1)

### Output Variables
* vault_cluster_instance_ids - The instance IDs for the Vault instances.
* vault_cluster_instance_ips - The instance IPs for the Vault instances.
* consul_cluster_instance_ids - The instance IDs for the Consul instances.
* consul_cluster_instance_ips - The instance IPs for the Consul instances.
* elb_dns - the DNS name for the internal ELB
* cluster_server_role - The role name for the IAM role assigned to the cluster instances for use with attaching policies.

## Infrastructure Options
This module will install Vault as `$var.vault_cluster_size` individual instances or as `$var.vault_cluster_size` instances inside an ASG
This behaviour is controlled by the use of a boolean variable `use_asg`
The default is false
```hcl
  /* This variable is a boolean that determines whether the Vault cluster is
  provisioned inside an ASG. */

  use_asg           = false
```
This module will install Vault with or without an _internal_ ELB/
This behaviour is controlled by the use of a boolean variable `use_elb`
The default is false
```hcl
  /* This variable is a boolean that determines whether the Vault cluster is
  provisioned behind an ELB */

  use_elb           = false
```
If an ELB is used then you have the option for this to be an internal (recommended) or external by use of the internal_elb variable.

```hcl
/* this variable is a boolean that determines if the ELB is internal or
 external. If use_elb is set to false then it will have no effect*/

internal_elb = false
```

## Use of awskms autounseal
This module will configure and deploy a AWS KMS key and set the cluster to use this for auto unsealing. This behaviour is controlled by the use of a boolean variable `use_auto_unseal` The default is false
```hcl
/* This variable controls the creation of a KMS key for use with awskms
seal operations */

use_auto_unseal   = false
```
## User Data Install

If the `use_userdata` variable is set to `false` then no post install user data configuration will take place and it is assumed that the cluster configuration will be done via some other method. This module also provides Packer scripts to perform this. See below *Packer Install*

If the `use_userdata` variable is set to `true` then the user data scripts will be used and the user data install files must be set up prior to deployment. The steps for doing this are below.

```hcl
  /* User Data. This sets up the configuration of the cluster.
  If use use_userdata variable is set to false then none of these need be
  configured as they will not be used and they are set to default variables in
  the module variables.tf. These defaults are shown below */

  install_bucket = "my_bucket"
  vault_bin      = "vault.zip"
  key_pem        = "key.pem"
  cert_pem       = "cert.pem"
  consul_version = "1.3.1"
  consul_cluster_size   = 5
```
This user data install will install Vault and Consul binaries either from the S3 bucket (var.install_bucket) or download them directly from the [hashicorp releases](https://releases.hashicorp.com/) page.
This behaviour is controlled by the use of the [vault|consul] version and bin variables.
```hcl
  # This will mean the install will look for the Vault binary in the S3 bucket
  vault_bin      = "vault.zip"
  # This will mean the install will download the release from releases page
  vault_version  = "1.0.1"
  # This will mean the install will look for the Consul binary in the S3 bucket
  consul_bin     = "consul.zip"
  # This will mean the install will download the release from releases page
  consul_version = "1.3.1"
```
You should use either the *bin* or the *version* for each application. If you put in both, the install  will only look in the S3 bucket.
You can have the behaviour different for each application if you wish.

This module installs the Vault and Consul servers as per the [Deployment Guide](https://www.vaultproject.io/guides/operations/deployment-guide.html). This includes full configuration of the servers. Because of this there are a few prerequisites that must be followed.

The steps for the configuration of the servers is as follows:
* Create a private S3 bucket first
* Copy the provisioning/templates/s3-access-role.json.tpl from the
Vault module to a local space and reference it in this code block.
This will create a policy allowing the instances access to the bucket.

```
resource "aws_iam_role_policy" "s3-access" {
  name   = "s3-access-policy"
  role   = "${module.vault_cluster.cluster_server_role}"
  policy = "${data.template_file.s3_iam_policy.rendered}"
}

data "template_file" "s3_iam_policy" {
  template = "${file("s3-access-role.json.tpl")}"

  vars {
    s3-bucket-name = "${var.my_bucket}"
  }
}
```
* Copy the install_files directory from the module to the install_files bucket.
`module_path/packer/install_files`
* Copy your certificate and private key for Vault TLS to the install_files bucket.
* Copy the enterprise Vault binary to the install_files bucket
* The final S3 bucket will look like this:
```
${var.my_bucket}/install_files/
  - cert.pem              # The certificate for TLS for Vault (var.cert_pem)
  - key.pem               # The private key for TLS for Vault (var.key_pem)
  - install-consul.sh     # The install script for consul
  - install-vault.sh      # The install script for Vault
  - install_final.sh      # The script used to set up ACL on Consul server
  - vault.zip             # The Vault binary as this will cater for an
                              enterprise binary already downloaded.
                              (var.vault_bin)
```
This installs the Vault and Consul servers and agents to a point. The module outputs the Consul servers and the Vault servers IPs as a list and these will need to be used to perform a final setup

## Packer Install
The packer directory contains 2 packer configurations - one to install the Vault nodes and one to install the Consul nodes. This will give you 2 AMIs that can be used in the module as:
```hcl
vault_ami_id           = "${var.vault_ami_id}"
consul_ami_id          = "${var.consul_ami_id}"
```
If you pre-build the AMIs via this method then you should also set the use_userdata variable to false so that the Vault and Consul installs are not overwritten.

A current caveat with Ubuntu installs is that they seem to require a reboot directly after install and before the final_config script is run.
```
use_userdata = false
```
## Security groups
This module sets up the correct security groups for the internal communications for the cluster between Vault and Consul, but additional SGs can be assigned to the cluster instances.
```hcl
/* This is where you can add additional security groups to allow for such
  things as SSH access and access to the API */

  additional_sg_ids = [
    "${aws_security_group.private_instances_sg.id}",
    "${aws_security_group.vault_cluster_ext.id}",
  ]
```
## Final Configuration
If the cluster has been configured either by the user data or Packer method from this module then there is one final step that is required to complete the configuration as per the deployment guide. This final step cannot (and arguably should not) be completed at the Terraform deployment stage and so a helper script is available here: `cfg/final/final_config.sh`
This script performs the following functions:
* Generate an ACL master token on one Consul server
* Generate an agent ACL token on one Consul server
* Set up that agent ACL token on all Consul servers and clients
* sets up the kms_key if used
* Generate a Vault ACL token on one Consul server
* Set up that Vault ACL token on all Vault servers
* sets the Vault api_addr to either the individual host IPs (if no ELB is used) or the supplied DNS for the ELB

There is a continue and break point in the script that allows you to check if the Consul cluster has actually come up and has returned a management token for ACL initialisation before continuing. This is in there as there are some issues with Ubuntu at the moment when the instances are not rebooted, Consul does not start correctly.
This will then complete the configuration of the Vault cluster as per the deployment guide.
For ease of use the Consul master token is output by the script and should be saved (possibly as a Vault static secret) if this may be required again.

To use this script:
copy this script to a suitable host that has ssh access to all your Vault and Consul Servers
```
Usage: final_config.sh [OPTIONS]
Options:

  --consul-ips  A comma separated string in " no spaces of Consul server IPs.
                    Required

  --vault-ips   A comma separated string in " no spaces of Vault server IPs.
                    Required

  --kms-key     The key id for the kms key if this has been set to be used

  --kms-region  The region that the kms key was created in.   

  --elb-dns     The DNS name that the Vault servers can redirect API calls to if used  

  --install-bucket  This should be used only if you are installing from a packer install as the script will need to download the install_final.sh script to each server.
```
It is arguable whether this final step should be included in any sort of deployment as this is really the domain of configuration management, however it is included here for completeness to finalise the Vault install so that it fully replicates the deployment guide.

## Next Steps
At this point if all steps are completed there will be a Vault cluster backed by a Consul storage backend. This Vault cluster will need to be initialised and unsealed.

There is a script to stop and start the cluster in the cfg directory _vault_control.sh_
