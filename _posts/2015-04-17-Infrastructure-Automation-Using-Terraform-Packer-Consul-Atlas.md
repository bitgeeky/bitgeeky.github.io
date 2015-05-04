---
layout: post
title: Infrastructure Automation Using Terraform, Packer, Consul and ATLAS Workflow
date: 2015-04-17 03:35:19 UTC
updated: 2015-04-17 03:35:19 UTC
comments: true
categories: infrastructure automation terraform packer consul atlas homepage
---

I have been working as a software contractor for HashiCorp for over a month now. My main work is to write examples for HashiCorp [ATLAS](https://atlas.hashicorp.com/) which is an upcoming product from HashiCorp and is under tech review at the moment. During this time I got to work on tools like Terraform, Packer and Consul. These tools are all from HashiCorp. In this blog post I'll describe briefly what each of these tools is used for and about the examples I wrote for Atlas.

What is Infrastructure Automation ?
----
As the name suggests its automating the process of creating vm's, managing load balancers, security groups, ssh-keys, ip-address and all the other resources that are associated with the machine or container.
When the size of data centres is large and it consists of hundreds or thousands of nodes and servers its impossible to handle these resources manually. So you need an automation tool that would do the work for you and to preserve all the infrastructure as a code, configuration files to be more precise. Based on the infrastructure configuration and the commands you pass every thing in cloud can be controlled using the automation tool.

What is Terraform ?
----
[Terraform](https://www.terraform.io/) is an Infrastructure automation tool used to store infrastructure as code. Its written in `go` and has its own syntax for writing infrastructure configuration using `*.tf` file. A sample terraform file looks like:

```
provider "aws" {
    access_key = "ACCESS_KEY_HERE"
    secret_key = "SECRET_KEY_HERE"
    region = "us-east-1"
}

resource "aws_instance" "web" {
    instance_type = "t1.micro"
    ami = "ami-408c7f28"

    # This will create 1 instances
    count = 1
}
```

So you can specify the priver AWS in this case and specify the resources you want to create on that provider like `aws_instance` in this case. On running `$ terraform plan` you can check the validity of your terraform configuration, it will give an error if there is any syntax issue, resource dependency issues and performs validation of resources, credentials etc but the actual output you get on running `$ terraform apply` - which actually deploys these resources might differ from the output of `$ terraform plan` in some cases.

So running `$ terraform apply` on this will create an aws instance with specified configuration. It actually creates a graph on backend to check for resource dependencies and you can get the graph using the commnd `$ terraform graph`. If anything goes wrong you can run `$ terraform destroy` to destroy the complete infrastructure.

It comes with support from a lot of service providers AWS, Azure, Digital Ocean, Heroku, Google Cloud, Docker etc. All these service providers actually provide api's to communicate with their services and terraform provides an abstarction layer over these api's along with other very useful features.

This is a very small example to just give an idea what terraform is all about, you should see the [official docmentation](https://www.terraform.io/docs/index.html) for many other features that it offers. It has developed a very good community also you can visit `#terraform-tool` on freenode to ask any doubts etc.

What is Packer ?
----
Writing terraform files can get a lot complex sometimes when you need to install some packages on your machine or make a particular setup on the machine you just created. Its always advised to machine a machine image for whatever you want to do. In the machine image you can specify the packages you want to install on the machine, configuration you need to make before the machine actually gets created or scripts you want to execute after you create the machine.

[Packer](https://www.packer.io/) allows you to deal with this task of creating machine images so that they can be deployed cross platform on multiple resource providers parallely. Its also written in `go` and all the image configuration can be written in a simple `json` format. A sample configuration looks like:

```
{
  "variables": {
    "aws_access_key": "ACCESS_KEY_HERE",
    "aws_secret_key": "SECRET_KEY_HERE"
  },
  "builders": [{
    "type": "amazon-ebs",
    "access_key": "{{user `aws_access_key`}}",
    "secret_key": "{{user `aws_secret_key`}}",
    "region": "us-east-1",
    "source_ami": "ami-9eaa1cf6",
    "instance_type": "t2.micro",
    "ssh_username": "ubuntu",
    "ami_name": "packer-example {{timestamp}}"
  }]
}
```

Similar to `$ terraform plan` you can validate the configuration using `$ packer validate example.json` and the run `$ packer build example.json` to actually create the machine image.

What is ATLAS ?
----
So now you have the machine images and your terraform deployment configurations. Lets say, you want to share your infrastructure with one of the team members or you need a place to host the machine images, develop your infrastructure further and then deploy it immediately. This is what ATLAS is designed for. It provides a platform to perform all these operations at a single place.

I would describe [ATLAS](http://atlas.hashicorp.com/) as a utility to store and share your infrastrucutre. Like you can share and develop your application code on GitHub, ATLAS provides a platform to develop and share you infrastructure while providing instant deployment option.

The workflow of ATLAS is like:

1. Create machine images using Packer and push them to Atlas.
2. Push the application code to Atlas. You can use a vagrant box for this.
3. Link application code to infrastructure images.
4. Deploy Atlas artifacts using Terraform on cloud.

What is Consul ?
----
Now that the infrastructure is up and running, how do we find that all nodes in our infrastrucure are behaving as expected and are up and running ? In another case say you deployed database on one of the nodes and application on the other one, now how does one nodes comes to know about other ? How can the application and database nodes communicate with each other ?

[Consul](https://consul.io/) provides a solution to these problems. Its basically a service discovery and health monitoring tool. Apart from this it can also be used as a Key/Value Storage and it can work across multi datacenters. So the nodes monitored by consul need not be present in the same datacenter. Its based on the `gossip` protocol which itself utilises `udp` for data transfer.


So these are the interesting things I am working on. Apart from these I also worked on Docker containers and Vagrant boxes but those are quite common now.

What examples I wrote ?
----
I wrote a couple of examples on how to deploy Wordpress, link Travis, deploy Docker, Discourse and tutorials on getting SSH access to the machines in infractructure using the ATLAS workflow. I will give a brief introduction and link for each of these in the next blog post.
