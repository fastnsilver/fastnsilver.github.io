---
layout: post
title: Installing Pivotal Application Service 2.1.1 on Azure using Terraform and Concourse
date: 2018-04-24
summary: How to get PAS 2.1 up and running on Azure with Terraform and Concourse and keep it up to date
logo: industry
categories: PCF
comments: true
---

Pivotal Application Service (PAS) 2.1 was [released](https://content.pivotal.io/slides/pivotal-cloud-foundry-2-1-making-transformation-real-webinar) just recently and I wanted to take it for a spin on Azure.  I started [here](https://docs.pivotal.io/pivotalcf/2-1/customizing/azure.html), and chose this [path](https://docs.pivotal.io/pivotalcf/2-1/customizing/azure-terraform.html).  I also spun up an instance of [Concourse](https://concourse-ci.org) and configured some [pipelines](https://github.com/pivotal-cf/pcf-pipelines) to  expedite installation of other tiles and to help me keep PAS up-to-date.


## Getting Started

* Signup for 
    * an [Azure account](https://signup.azure.com/signup)
    * a [Pivotal Network account](https://account.run.pivotal.io/z/uaa/sign-up)
* Download and install the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
* [Create a parent Azure DNS Zone](https://docs.microsoft.com/en-us/azure/dns/dns-getstarted-portal#create-a-dns-zone)
* [Create 'A' records](https://docs.microsoft.com/en-us/azure/dns/dns-getstarted-portal#create-a-dns-record) for Concourse, Jenkins, Sonarqube, and Artifactory
* [Spin up an Ubuntu 16.04 VM](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-portal)
* [SSH into the VM](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-portal#connect-to-virtual-machine)
* [Download](https://github.com/fastnsilver/jenky/archive/master.zip), unpack, and install [Jenky](https://github.com/fastnsilver/jenky/tree/master/compose), then exit the VM


## Installing Operations Manager

* [Install Terraform](https://www.terraform.io/downloads.html)
* Download and unpack [Terraform scripts for PCF on Azure](https://github.com/pivotal-cf/terraforming-azure/archive/v0.11.0.zip)
    * Follow the [instructions](https://github.com/pivotal-cf/terraforming-azure#how-does-one-use-this) for creating an automation account, seeding a `terraform.tfvars` file with the appropriate variables, and standing up an environment

Here's a sample `terraform.tfvars` (you would of course need to replace values)

```
subscription_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
tenant_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
client_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
client_secret = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
env_name = "pas"
env_short_name = "cloudclownas"
location = "West US"
ops_manager_image_uri = "https://opsmanagerwestus.blob.core.windows.net/images/ops-manager-2.1-build.212.vhd"
dns_suffix = "cloudclown.io"
vm_admin_username = "admin"
```

I took care to capture the output from `terraform apply plan`.

Lastly, I visited the Azure Portal and added an NS record to the parent zone `cloudclown.io` by copying the NS details from the `pas.cloudclown.io` child-zone.


## Deploying Operations Director

I stepped through the instructions [here](https://docs.pivotal.io/pivotalcf/2-1/customizing/azure-om-config-terraform.html).

I configured `Internal Authentication`, skipped configuring a proxy, accepted the terms and conditions, and logged into OM.  From there, I filled in necessary details for:

* Azure Config
* Director Config
* Create Networks 
* Assign Network

panels, while frequently referencing the output from `terraform apply plan`. 

I had to be careful with the `Networks` sub-sections in the `Create Networks` panel.  The `Azure Network Name` for each subnet should be built from `{pcf_resource_group_name}/{network_name}/{_subnet_name}` and the 

* CIDR
* Reserved IP Ranges
* Gateway 

field values should not be taken straight from the screenshots in the documentation.

Before clicking `Apply Changes`, I adjusted the `Resource Config` to create only two instances of `Master Compilation Job`.


## Installing PAS

Here's where I cheated a bit.  

I downloaded the release version of PCF Pipelines from the [Pivotal Network](https://network.pivotal.io/products/pcf-automation/)

I configured the `upgrade-ert` pipeline, editing `params.yml` to be (something like):

```
enable_errands: 
iaas_type: azure
opsman_admin_username: admin
opsman_admin_password: xxxxx
opsman_client_id: 
opsman_client_secret: 
opsman_domain_or_ip_address: pcf.pas.cloudclown.io
pivnet_token: xxxxxxxxxxxxxxx
product_globs: "cf*pivotal"
product_version_regex: ^2\.1\..*$
```

(To install small footprint instead, replace `cf*pivotal` above with `srt*pivotal`).

Then I ran 

```
fly -t cloudclown login --concourse-url https://ci.cloudclown.io
fly -t cloudclown set-pipeline -p upgrade-pas-on-azure -c pipeline.yml -l params.yml
```

to login into my Concourse instance and configure an upgrade pipeline for PAS.  

Next, I logged in to Concourse, clicked on the pipeline and paused the `apply-changes` step, then unpaused the pipeline.  After the `upload-and-stage-ert` step completed, I revisited OM and began configuring PAS as per [Deploying PAS on Azure](https://docs.pivotal.io/pivotalcf/2-0/customizing/azure-er-config.html#assign-networks), completing steps 2 through 23.  

Finally, revisiting Concourse, I unpaused the `upload-and-stage-ert` pipeline to effectively `Apply Changes`.