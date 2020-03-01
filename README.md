# Attacking And Defending The GCPMetadata API
This repo gives an overview of some GCP metadata API attack and defend patterns

## Overview
A metadata API in a cloud platform is an internal API resources like VM's that run code can query to obtain information about themselves, and obtain credentails to access the instance identity attached to the resource.

<img src="https://i.imgur.com/vW1iiFh.png" width="600">

Similar API's exist in other cloud platforms, including AWS. 

## AWS Metadata API Recap/Overview
In AWS for a long time simple get requests could be use to fetch instance identity credentials. This lead to a string of SSRF vulnerabilities, where when services would send simple get requests on behalf of a user, the user could proxy requests to the metadata API endpoint and fetch credentials.

<img src="https://i.imgur.com/6PoRmKy.png" width="400">

This is the vulnerability that lead to the [Capital One data breach](https://edition.cnn.com/2019/07/29/business/capital-one-data-breach/index.html)

### AWS Protection
AWS roled out [IMDSv2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html), which protects against most SSRF vulnerabilities, by requiring users have a header set on their request.

# GCP Metadata API
Similair to AWS, there are two versions of the metadata API, v0.1 and v1. v1 requires a header be set, which protects against many SSRF vulnerabilities.

## Bypassing the GCP Protections in Cloud Functions
Cloud functions are a serverless offering in GCP that has a metadata available to them. 

Not too long ago, a blog post was released by Google, demonstrating how to run a headless browser in these cloud functions

<img src="https://i.imgur.com/Vux88f7.png" width="400">

Because browsers can set headers, untrusted HTML can potentially access the metadata API. They are beholdent to the Same Origin Policy, but [DNS rebinding](https://github.com/nccgroup/singularity) tricks [allow](https://www.youtube.com/watch?v=Q0JG_eKLcws) us to [bypass](https://www.youtube.com/watch?v=HfpnloZM61I) this restriction.

This means customers of Google that stood up headless browsers in the cloud function and allowed untrusted HTML to be rendered exposed the instance credentials to the untrusted HTML.

### Identifying Vulnerable Customers
HTTP Triggered Cloud functions expose most of their naming in their naming in their subdomain. They also default to being on the open internet with no authentication. They use the following naming convention:

```
https://<region>-<GCP-project-name>.cloudfunctions.net/<function name>
```

Where the function names default to function-1, function-2, etc... the blog post also suggested naming the function `screenshot`.

Using this information and a passiveDNS vendor, we can enumerate cloud functions:

<img src="https://i.imgur.com/CI4Pwt2.png" width="400">

At this point to get the last peice, we can guess the path via either trying `/function-X` or `/screenshot` and see if we're presented with a page that matches the one from the blog post.

Using this technique I was able to identify vulnerable customers.

_Because in GCP instances have default identities with default permissions, this leads to a compromise of a signifigent portion of the customer's GCP project. More on this later..._

### Fixing DNS Rebind Attacks
Google paid 1337 for the report, and added Host header validation to protect against future DNS rebind attacks. 
<img src="https://i.imgur.com/EcwsR0y.png" width="400">

## Google Created Identities (Service Accounts) In GCP
In GCP, Service Accounts are used to provide instances identity and give them privlige. When you enable all the API's in GCP, identities with default permissions bound to your project are created on your behalf. Some of them are attached to instances you control, some of them are attached to instances you do not control. Here's a list of them:

<img src="https://i.imgur.com/OrEPx7W.png" width="800">

These service accounts fall into two buckets, __Google Managed__ service accounts and __User Managed__ service accounts.

### User Managed Service Accounts
The main difference between these two is __User Managed__ service accounts, though created on your behalf, have permissions that are easily visible to you, and are mostly attached to resources you can see and have control over. There are two __User Managed__ service accounts, and 47 __Google Managed__ ones for comparison.

Here's what the user managed ones look like:
```
<projectID>-compute@developer.gserviceaccount.com
<projectID>@appspot.gserviceaccount.com
```

Both user managed Service Accounts have a project level Editor role binding. The editor role binding has thousands of permissions and can do things which include, access all the bigquery in your project, access all the buckets in your project, access all the databases in your project, access all the spanner in your project, etc...

<img src="https://i.imgur.com/3pWAQiU.png" width="400">

These two service accounts are attached to just about all of your resources by default. This means the cloud function example above likely meant the compromise of a lot of data and many resources, because by default the metadata API would have returned a credential that has over 2000 permissions.


### Attacking The Metadata API in GKE
In GKE, the GCP Kubernetes offering, the nodes that power your cluster are just standard GCP VM's you can view in your project. This means, again they are given the default service account with project editor.

Unlike Cloud Functions, VM's have something called _scopes_ that limit which API's the service account can access, regardless
of the service account's permissons. By default this is the scope applied to VM's:

<img src="https://i.imgur.com/g3moLzt.png" width="400">

Note: this leaves read access to storage open, meaning VM's by default can read data out of all buckets in the project.

Because workloads in GKE all run on underlying VM's that have storage open, all workloads in GKE by default can hit the metadata API, and fetch these credentials.

Note, nodes likely need storage enabled, so they can fetch from the GCR (Google Container Regeistry) which is powered by storage.

#### GKE Metadata Protections
GCP offeres a wide range of offerings to protect against both the K8's and the GCP metadata API:
<img src="https://i.imgur.com/yX3ZXOt.png" width="400">

You can read about them here:
https://cloud.google.com/kubernetes-engine/docs/how-to/protecting-cluster-metadata#concealment
https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity
https://cloud.google.com/kubernetes-engine/docs/how-to/shielded-gke-nodes


Note none are enabled by defualt, and the only offering that blocks the VM's GCP credentials from being fetched is Workload Identity, and it's incompatible with Metadata Concealment.

### GKE Example
Here's a demo where we enabled both Concealment and Shielded Nodes, and where able to access the node's credentials: https://drive.google.com/file/d/1JLNzBjixe_iqPSmOZfR8oE9spdnOVCp8/view
 
 


In GKE, the GCP Kubernetes offering, the nodes that power your cluster are just standard GCP VM's you can view in your project. This means, again they are given the default service account with project editor.
