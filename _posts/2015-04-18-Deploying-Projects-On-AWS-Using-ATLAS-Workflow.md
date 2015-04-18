---
layout: post
title: Deploying Projects On AWS Using ATLAS Workflow
date: 2015-04-18 04:41:38 UTC
comments: true
categories: [Infrastructure Automation Terraform Packer Consul Atlas Wordpress Travis SSH Docker Discourse]
---

In the [last](http://pankajmalhotra.com/Infrastructure-Automation-Using-Terraform-Packer-Consul-Atlas/) post I talked about Infrastructure automation and gave a brief introduction about tools like Terraform, Packer, Consul and also talked about ATLAS and the workflow for deploying deploying infrastructure using it.

Today I am gonna get into details on how to actually deploy projects like a LAMP server, Wordpress, Discourse, Docker on AWS using Atlas workflow and how to generate and associate RSA keys and security group to SSH into any of your machines.

Most of the informtion is in linked guides for each of the projects, I'll concentrate on main parts and things to pay special attention to.

The [getting-started](https://atlas.hashicorp.com/help/getting-started/getting-started-overview) guide gives a very good idea about how things work and practically explains the complete ATLAS workflow with a very easy example.

Deploying Wordpress on AWS
----
Main thing to note on this tutorial is that you have to deploy database on a separate node since the aws instance gets rebuild everytime you push a new version of you application or make any configuration changes, so to preserve the application data its advised to deploy database on a separate node. We are using Consul here as a service discovery tool so that we get to get information about our application and databse server. So you don't have to hardcode any ip address or any other other resource attached to a particular service.

Since we are using Consul here, we will have to allow `udp` traffic on the port which Consul is running. As mentioned consul uses a `gossip` protocol to establish communication between services.

[Link to tutorial] (https://github.com/hashicorp/atlas-examples/tree/master/wordpress)

Using Travis to push Application code to ATLAS
----
As the name suggests this is an example demonstarting how to push application code to ATLAS from Travis. Here we use [ATLAS Upload CLI](https://github.com/hashicorp/atlas-upload-cli), a simple utility to push application code to Atlas.

[Link to tutorial](https://github.com/hashicorp/atlas-examples/tree/master/TravisCI)

SSH in an AWS Instance
----
If you are setting up a production infrastructure, its advised not to allow any ports open or leave any authentication keys in aws instance but surely for debugging purpose you need to sometimes SSH in the instance you created using ATLAS workflow.

[Link to tutorial](https://github.com/hashicorp/atlas-examples/blob/master/AWS-SSH-Setup/ssh.md)

Deploy Docker Container on AWS
----
Support for Docker provider is released in recent version of terraform `0.4`, it still lacks some of requirements but the work is under progress and more features are expected to be released in upcoming versions.

[Link to tutorial](https://github.com/bitgeeky/atlas-examples/tree/docker/Docker)

Deploy Discourse on AWS
----
Discourse uses docker and does a lot of bootstraping to the instance before deploying the actual application code. So its advised to follow their official guide and treat the bootstraping part as a black box to avoid breaking anything.

[Link to tutorial] (https://github.com/bitgeeky/atlas-examples/tree/discourse/Discourse)

There are many other examples in official HashiCorp's [ATLAS Examples](https://github.com/hashicorp/atlas-examples) respository on GitHub. I would recommend to go through some of them to get a good grip of how things work. 

If you think any project is missing feel free to raise an issue or You Can Add One Too !
