# Attacking And Defending The GCPMetadata API
This repo gives an overview of some GCP metadata API attack and defend patterns

## Overview
A metadata API in a cloud platform is an internal API resources like VM's that run code can query to obtain information about themselves, and obtain credentails to access the instance identity attached to the resource.

![metadataAPI](https://i.imgur.com/vW1iiFh.png)


Similar API's exist in other cloud platforms, including AWS. 

## AWS Overview
In AWS for a long time simple get requests could be use to fetch instance identity credentials. This lead to a string of SSRF vulnerabilities, where when services would send simple get requests on behalf of a user, the user could proxy requests to the metadata API endpoint and fetch credentials.

This is the vulnerability that lead to the [Capital One data breach](https://edition.cnn.com/2019/07/29/business/capital-one-data-breach/index.html)

