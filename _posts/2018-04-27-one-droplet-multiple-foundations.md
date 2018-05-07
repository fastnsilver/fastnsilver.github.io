---
layout: post
title: One Droplet, Multiple Foundations
date: 2018-04-27
summary: Thoughts on droplet management across multiple foundations
logo: tint
categories: PCF
---

I had an occasion to play with [cf local](https://github.com/cloudfoundry-incubator/cflocal). 

Why you might want to take a look at it yourself is described well enough [here](https://pivotal.io/cf-local).  But there's perhaps one more reason you might want to use it - for managing the creation and deployment of an immutable artifact (i.e., droplet) across multiple foundations.

## What's in a Droplet?

A droplet is basically a tarball of compiled application code stored in a blob store. 

How does it get created? 

When you execute a `cf push` the CLI tells the Cloud Controller to create an application record.  Next it stores the application metadata (information typically drawn from a `manifest.yml` and/or CLI parameters). Before uploading the application files it does a resource match, and omits files that exist in the resource cache.  The uploaded application files are combined with files from the resource cache to create an application package.  This package, known as a droplet, is stored in the blobstore.

## Why not just cf push?

A blobstore is provisioned for use by a single foundation. So, if you are an organization maintaining multiple foundations, does it make sense to create separate droplets for the same code release? Why wouldn't you simply manage and deploy from a single repository of versioned artifacts? 

## A way forward?

As I was looking at what capabilities `cf local` provided, it struck me that I could script a workflow to create a droplet, then reuse it to push to multiple foundations. 

Step 1:  Push application to Foundation A

```
cf api ...
cf login
cf push ...
```

Step 2:  Pull droplet and export a Docker image
```
cf local pull ...
cf local export ...
```

(Optional) Step:  Publish Docker image to a Private Registry

```
docker tag ...
docker push ...
```

Step 3: Push application to Foundation B

```
cf api ...
cf login
cf local push ...
```

(Of course, you could rinse-and-repeat Step 3 to push to other foundations).