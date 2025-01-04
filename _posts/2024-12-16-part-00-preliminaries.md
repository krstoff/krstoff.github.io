---
layout: post
title:  "Writing an Orchestrator - Preliminaries"
---

Most datacenter applications are now structured as message-passing services, micro- or otherwise, delivered in containerized format. Those who aren't using developer-centric platforms like [Heroku](http://heroku.com), [Render](http://render.com), or [Fly.io](http://Fly.io), frequently employ a full-featured container orchestrator such as Kubernetes or Nomad to deploy, scale, and manage their workloads. Kubernetes in particular has sizable mindshare among cloud-native developers, in part because of its industry backing and a standardization effect, resulting in a thriving ecosystem around the technology.

But Kubernetes is big, and it's big because it was designed to accommodate a wide array of use-cases. The flexibility that made it so popular also makes it daunting for some people to learn. Immersed in its API, one might get the impression that this complexity is essential, but the core of container orchestration itself isn't really that complicated, and building one is a tractable mid-sized exercise for those with some time to kill.

Well, I have some time. I'll be writing a container orchestrator, starting from containerd’s API, and summarizing my experience along the way, providing fully reproducible code. 

My goal is:

- a platform that covers the major aspects of container orchestration, such as resource-based scheduling, networking, service discovery, monitoring, configuration, and scaling
- extensible and cloud-agnostic, suitable for both a datacenter cluster and a bundle of raspberry pis
- reasonable for managing a cluster of twenty-five or more services, or a system with fifty to a hundred different developers
- flexible enough to be the foundational layer of a bespoke, internal platform with specific security and networking requirements

As an operations novice I expect to bang my head a lot, but I'll make and explain design choices along the way, and consult state of the art. I'll also make simplifying assumptions, like only utilizing Linux hosts, using IPv6 for networking, or using specific virtual machine capabilities that make VPC routing easy.

This project will make use of many different technologies and those will be introduced over time. The course of development overall, however, will make use of several technologies immediately, and so if you're following along, you should set up your development environment accordingly. You’ve probably used many of these tools already but they’re listed here just in case you haven’t. 

## Amazon Web Services

You’re going to want to run your cluster somewhere and this is as good a place as any. Unfortunately, we will be using resources beyond those enumerated in the free tier, but as long as computing resources are instantiated only during active development, you should be able to keep the bill very low. Particularly important is that T2.micro instances don’t support the networking features we will be using. 

- Set up an AWS account if you don’t have one already, get your private key, and install the aws cli tool

## Terraform

We’ll follow good practice and embrace Infrastructure-As-Code fully. To have full trust in the configuration of our infrastructure we need to be able to fearlessly destroy and recreate it, over and over again. We could try things out quickly by using the AWS console, ssh-ing into our instances, scp-ing updated software, etc., but we may end up missing something when we commit this configuration to code. We can save ourselves the headache by making sure our set-up is reproducible, immediately, each and every time.

- Download terraform
- Configure it to use the credentials associated with your AWS account
- Get comfortable bringing a variety of resources up and down. Create instances, subnets, security-groups, whatever.

Some people prefer something like Pulumi to Terraform so that they can use their favorite programming language. Use whatever you like - the code presented here will be in Terraform because it’s what I know and it’s strictly more common.

## Rust

Rust is a systems programming language like C or C++. Not only is it memory-safe and thread-safe by default, but it is also superb at describing and enforcing system invariants overall, such as protocols, exhaustive case-handling, and resource destruction. Used judiciously, macros also enable the creation of excellent Domain Specific Languages, or generate tedious boilerplate for you. 

The de facto default language for back-end networked services in the container orchestration world is Go. Plugins for Kubernetes and containerd are written in Go. CNCF projects frequently support Go first. Go is a first-class language at Google and the crossover between its software developers and contributors to CNCF projects is very high. But the truth is I just don’t like Go. My personal opinion is that it's an unpleasant programming language on an amazing runtime. More importantly I am not good enough at Go to feel secure in the code that I write. Its type system lacks in certain regards where I don’t always know if I’ve handled all the various errors that can arise from an API. Error-handling is verbose and I have returned a nil in the wrong place more than one time. I feel safer with the code that I write in Rust. 

- Download rustup if you don’t already have it.
- Work through the tutorial if you’ve never used Rust before
- Make sure your environment is set correctly so that you can actually build crates

## Packer

Packer is a tool for creating Amazon Machine Images, which are the templates for EC2 instances. Our instances will be running identical set-ups.

- Download packer
- Complete the tutorial
- Upload it to Amazon S3
- Launch an EC2 instance based on your AMI.

Git can’t handle version controlling our AMIs. Luckily AWS S3 supports differential storage of volumes. The cost of S3 for EBS is about five cents per gigabyte per month, so, not bad, but be mindful that this will cost money, however minuscule. 

## Docker

Naturally we will be building container images, so I'm reaching straight for Docker. If you want to use Podman or Buildah, feel free. The important part is that you can build and upload OCI Images that will be pulled by our cluster.

## POSIX

I use a variety of *nix tools and there will be some shell scripting. I develop primarily on WSL2 - most of the tools I use are either also present on OSX or have some easily discovered analogue.
