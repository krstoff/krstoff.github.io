---
layout: post
title:  "Part 00: Preliminaries"
---

This project will make use of a lot of different technologies and those will be introduced over time. The course of development overall, however, will make use of several technologies immediately, and so we have to set up our development environment accordingly. You’ve probably used many of these tools already but they’re listed here just in case you haven’t. 

## Amazon Web Services

You’re going to want to run your cluster somewhere and this is as good a place as any. Unfortunately, we will be using resources beyond those enumerated in the free tier, but as long as computing resources are instantiated only during active development, you should be able to keep the bill very low. Particularly important is that T2.micro instances don’t support the networking features we will be using. 

- Set up an AWS account if you don’t have one already, get your private key, and install the aws cli tool

## Terraform

We’ll embrace Infrastructure-As-Code fully for two reasons. The first is that this is just good practice. The second is that to have full trust in the configuration of our infrastructure we need to be able to fearlessly destroy and recreate it, over and over again. We can try things out quickly by using the AWS console, ssh-ing into our instances, scp-ing updated software, etc., but we may end up missing something when we commit this configuration to code. We can save ourselves the headache by making sure our set-up is reproducible, immediately, each and every time.

- Download terraform
- Configure it to use the credentials associated with your AWS account
- Get comfortable bringing a variety of resources up and down. Create instances, subnets, security-groups, whatever.

Some people prefer something like Pulumi to Terraform so that they can use their favorite programming language. Use whatever you like - the code presented here will be in Terraform because it’s what I know and it’s strictly more common.

## Rust

Rust is a systems programming language like C or C++. Not only is it memory-safe and thread-safe by default, but it is also superb at describing and enforcing system invariants overall, such as protocols, exhaustive case-handling, and resource destruction. Used judiciously, macros also enable the creation of excellent Domain Specific Languages, or generate enormous amounts of boilerplate for you, allowing you to focus on the actual task instead of doing tedious bookkeeping. 

The de facto default language for back-end networked services in the container orchestration world is Go. Plugins for Kubernetes and containerd are written in Go. CNCF projects frequently support Go first. Go is a first-class language at Google and the crossover between its software developers and contributors to CNCF projects is very high. But the truth is I just don’t like Go. It is a mediocre programming language on an amazing runtime. More importantly I am not good enough at Go to feel secure in the code that I write. Its type system lacks in certain regards where I don’t always know if I’ve handled all the various errors that can arise from an API. Error-handling is verbose and I have returned a nil in the wrong place more than one time. I feel safer with the code that I write in Rust. 

- Download rustup if you don’t already have it.
- Work through the tutorial if you’ve never used Rust before
- Make sure your environment is set correctly so that you can actually build crates

## Git

I don’t even need to explain this one.

## Packer

Packer is a tool for creating Amazon Machine Images, which are the templates for EC2 instances. Our instances will be running identical set-ups.

- Download packer
- Complete the tutorial
- Upload it to Amazon S3
- Launch an EC2 instance based on your AMI.

Git can’t handle version controlling our AMIs. Luckily AWS S3 supports differential storage of volumes. The cost of S3 for EBS is about five cents per gigabyte per month, so, not bad, but be mindful that this will cost money, however minuscule. 

## Docker

Naturally we will be building container images so I will be using docker to do this. If you want to use Podman or Buildah then by all means, do that. The important part is that you can build and upload OCI Images that will be pulled by our cluster.

## POSIX

I use a variety of *nix tools and there will be some shell scripting. I develop primarily on WSL2 - most of the tools I use are either also present on OSX or have some easily discovered analogue.
