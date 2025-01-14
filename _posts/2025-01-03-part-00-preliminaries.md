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

The sum of all the software I'll be writing that make clusters usable as a single platform will be called `hyphae` (pronounced HY-FEE). [Hyphae](https://en.wikipedia.org/wiki/Hypha) are the long, branching filaments that compose a fungi’s mycelium. It’s also a homonym for [Oakland’s most well-known hip-hop movement](https://en.wikipedia.org/wiki/Hyphy). `hyphae` essentially serves the same purpose as Kubernetes or Nomad.

As an operations novice I expect to bang my head a lot, but I'll make and explain design choices along the way, and consult state of the art. I'll also make simplifying assumptions, like only utilizing Linux hosts, using IPv6 for networking, or using specific virtual machine capabilities that make VPC routing easy. My hope is that by documenting this all, I help someone else out there and save them some time with hard questions. If you want to follow along, I'll be using the following technology:

## Amazon Web Services
You’re going to want to run your cluster somewhere and this is as good a place as any. Unfortunately, we will be using resources beyond those enumerated in the free tier, but as long as computing resources are instantiated only during active development, you should be able to keep the bill very low. Particularly important is that T2.micro instances don’t support the networking features I'll be using. 

## Terraform
To have full trust in the configuration of my infrastructure, I'm going to need to be able to fearlessly destroy and recreate it.

## Rust

Rust is a systems programming language like C++. Not only is it memory-safe and thread-safe by default, but it is also superb at describing and enforcing system invariants overall, such as protocols, exhaustive case-handling, and resource destruction. Macros generate tedious boilerplate and enable the creation of excellent DSLs.

The de facto default language for back-end networked services in the container orchestration world is Go. Plugins for Kubernetes and containerd are written in Go. CNCF projects frequently support Go first. Go is a first-class language at Google and the crossover between its software developers and contributors to CNCF projects is very high. But the truth is I just don’t like Go. More importantly I'm just not good enough at Go to feel secure in the code that I write. Its type system lacks in certain regards where I don’t always know if I’ve handled all the various errors that can arise from an API. Error-handling is verbose and I have returned a nil in the wrong place more than one time. I feel safer with the code that I write in Rust. 

## Packer

Packer is a tool for creating Amazon Machine Images, which are the templates for EC2 instances. Our instances will be running identical set-ups.

Git can’t handle version controlling our AMIs. Luckily AWS S3 supports differential storage of volumes. The cost of S3 for EBS is about five cents per gigabyte per month, so, not bad, but be mindful that this will cost money, however minuscule. 

## Docker

Naturally we will be building container images, so I'm reaching straight for Docker. If you want to use Podman or Buildah, feel free. The important part is that you can build and upload OCI Images that will be pulled by our cluster.

## POSIX

I use a variety of *nix tools and there will be some shell scripting. I develop primarily on WSL2 - most of the tools I use are either also present on OSX or have some easily discovered analogue.
