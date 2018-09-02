---
layout: post
title: Concouse with Credhub on Google Cloud Platform via BOSH Boot Loader
date: 2018-08-31
summary: Integrated credential management with a CI/CD pipeline automation engine
logo: industry
categories: Concourse
comments: true
---

[Concourse 4.1](https://concourse-ci.org/download.html#v410) was recently released so I figured I'd take [bbl](https://github.com/cloudfoundry/bosh-bootloader) for a spin and stand up a secure cluster integrated with [Credhub](https://github.com/cloudfoundry-incubator/credhub) on Google's [cloud](https://cloud.google.com).  

Thinking about the level of effort it takes to stand up and operate such a cluster, whether in an on-premise data center or in the cloud, and the tool choices I have as a DevOps practitioner, I gravitated toward [BOSH](https://bosh.io/docs/). BOSH is amazing at provisioning and deploying software. It's lifecycle responsibilities doesn't stop there, it also performs monitoring, failure recovery, and software updates with zero to minimal downtime.

Why would one use **bbl**?  Well, it simplifies standing up BOSH on AWS, Azure, GCP, vSphere, or Openstack. You'll see just how it does that in a few moments.

## Getting Started

While package managers are subject to [supply-chain breaches](https://nakedsecurity.sophos.com/2018/08/10/how-one-man-could-have-hacked-every-mac-developer-73-of-them-anyway/), I still *cautiously* rely on [Homebrew](https://brew.sh) to install and upgrade software on my Mac.  Here's a short list of packages I installed:

```
brew tap cloudfoundry/tap
brew install bosh-cli bbl terraform credhub-cli git google-cloud-sdk wget
``` 

Public cloud providers typically want you to authenticate via an identity and access management API, then it's generally good practice to setup and authorize a service account which will be allowed to create and destroy resources within your account.  This can be accomplished in about 5 commands on Google Cloud

```
gcloud init
gcloud auth login
gcloud iam service-accounts create <service account name>
gcloud iam service-accounts keys create --iam-account='<service account name>@<project id>.iam.gserviceaccount.com' <service account name>.key.json
gcloud projects add-iam-policy-binding <project id> --member='serviceAccount:<service account name>@<project id>.iam.gserviceaccount.com' --role='roles/editor'
```

Next, I set some enviromment variables 

```
export PARENT_FOLDER=$HOME/Documents/development/projects/pivotal/concourse
export BBL_IAAS=gcp
export BBL_GCP_REGION=us-west1
export BBL_GCP_SERVICE_ACCOUNT_KEY=$HOME/.ssh/<service account name>.key.json
export BBL_STATE_DIRECTORY=$PARENT_FOLDER/.bbl-state/$BBL_IAAS
export CREDHUB_REPO=$PARENT_FOLDER/concourse-credhub
export CONCOURSE_BOSH_DEPLOYMENT=$PARENT_FOLDER/concourse-bosh-deployment
mkdir -p $PARENT_FOLDER
mkdir -p $BBL_STATE_DIRECTORY
```

The `BBL_` prefixed variables above set the IaaS, region, and service account key location on my file system.  The last two variables are for convenience; they represent where I cloned a couple of useful Github repositories

```
git clone https://github.com/concourse/concourse-bosh-deployment
git clone https://github.com/pivotalservices/concourse-credhub
```

The first one facilitates deployment of Concourse and the second is used to [plan patch](https://github.com/cloudfoundry/bosh-bootloader/tree/master/plan-patches) the deployment by integrating Credhub.  

## Setting up a Jump host and BOSH Director

To create [Jump host](https://en.wikipedia.org/wiki/Jump_server) and [BOSH Director](https://bosh.io/docs/bosh-components/#director) VMs, I executed the following

```
cd concourse-credhub
bbl plan --lb-type=concourse
cp bbl-terraform/$BBL_IAAS/*.tf $BBL_STATE_DIRECTORY/terraform/
bbl up
```

After a few minutes I saw 2 VMs in my Google Cloud console. (I also got a VPC network, subnets, firewall rules, and a load balancer provisioned). Coolio!

## Interacting with BOSH Director

To SSH to either of these VMs I had a look at these [instructions](https://github.com/cloudfoundry/bosh-bootloader/blob/master/docs/howto-ssh.md).

To login to BOSH Director and check out what'd been deployed so far, I executed

```
eval "$(bbl print-env)"
bosh login
bosh deployments
```

## Deploying Concourse

Feeling confident I was ready to take the next few steps. I needed to capture the load balancer IP address that had also been created, so I set a few more environment variables

```
export BOSH_NON_INTERACTIVE=true
export CONCOURSE_HOST=$(bbl lbs | cut -d : -f 2- | tr -d '[:space:]')
CONCOURSE_URL=https://$CONCOURSE_HOST 
```

The `concourse-credhub` repository contains a `deploy-concourse.sh` script.  The [prerequisites](https://github.com/pivotalservices/concourse-credhub#prerequisites) mentioned that I should upload the appropriate Xenial [stemcell](https://bosh.io/stemcells#ubuntu-xenial) before attempting to execute it, so I did that with

```
bosh upload-stemcell https://bosh.io/d/stemcells/bosh-google-kvm-ubuntu-xenial-go_agent?v=97.15 --sha1 a19718ac6fd64b1f3624d5769565369cb593dbbd
```

The prerequisites also mentioned that I should generate credentials for a local user on Concourse 

```
credhub login
credhub generate -t user -z admin -n /bosh-$(bbl env-id)/concourse/local_user
```

Of course I'd need those credentials when I wanted to login to Concourse, so here's how I got them

```
credhub get -n bosh-$(bbl env-id)/concourse/local_user
```

Then I was ready to run the script (and grab a cup of coffee)

```
./deploy-concourse.sh
```

When I came back in 10-15 minutes I saw 3 additional VMs in my Google Cloud console. Yay!

## Interacting with Concourse and Credhub

I had a choice to make.  I could connect to Concourse via (a) a browser or (b) the [fly](https://concourse-ci.org/fly.html) client by logging in with local user credentails.

Quite happy to stay at the command-line, I fetched the fly client, installed it, and logged in with

```
wget $CONCOURSE_URL/api/v1/cli?arch=amd64&platform=darwin
mv fly /usr/local/bin
chmod +x fly
fly -t <alias> login -c $CONCOURSE_URL -k -u <username> -p <password>
```

Next I took a look [here](https://github.com/pivotal-cf/pcf-pipelines/blob/master/docs/credhub-integration.md#-adding-secrets-to-credhub) and [here](https://github.com/pivotal-cf/pcf-pipelines/blob/master/docs/credhub-integration.md#sample-pipeline) for how to set up a simple pipeline that consumed secrets from Credhub.  

The beautiful result of all my prior efforts was that I actually had two instances of Credhub, one integrated with BOSH Director and the other with Concourse. But how do I connect to the latter?  That's where the `concourse-credhub` repository had me covered with another script, `target-concourse-cedhub.sh`.

```
source target-concourse-credhub.sh
```

Having logged in, I could create a secret named `hello` with

```
credhub set --type value --name '/concourse/<team name>/hello' --value 'World'
```

## Tearing it all down

Likely you'll want to save some money and not have this running 24x7.  To reap your cluster, execute

```
bbl destroy --gcp-service-account-key $BBL_GCP_SERVICE_ACCOUNT_KEY
```

## Final thoughts

I don't know about you but I am sold on how useful **bbl** is. I'm also confident that I could replicate this cluster on other public clouds like [AWS](https://github.com/cloudfoundry/bosh-bootloader/blob/master/docs/getting-started-aws.md) and [Azure](https://github.com/cloudfoundry/bosh-bootloader/blob/master/docs/getting-started-azure.md).  

There's some additional topics like (a) [CA rotation](https://github.com/pivotal-cf/credhub-release/blob/master/docs/ca-rotation.md), (b) the [Credhub service broker for PCF](https://docs.pivotal.io/credhub-service-broker/), and (c) [Spring integration](https://github.com/cloudfoundry-incubator/credhub/blob/master/docs/spring-java-credhub-integration.md) with Credhub that I'll explore in a subsequent post.

Lastly, I've created a [gist](https://gist.github.com/pacphi/1b0f5b5535ea8f006eb284ce2d3d3d89#file-install-concourse-with-credhub-on-gcp-sh) that coalesces all the above.
