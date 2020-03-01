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

## GCP Metadata API
Similair to AWS, there are two versions of the metadata API, v0.1 and v1. v1 requires a header be set, which protects against many SSRF vulnerabilities.

### Bypassing the GCP Protections in Cloud Functions
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

### Fixing DNS Rebind attacks
Google paid 1337 for the report, and added Host header validation to protect against future DNS rebind attacks. 
<img src="https://i.imgur.com/EcwsR0y.png" width="400">

