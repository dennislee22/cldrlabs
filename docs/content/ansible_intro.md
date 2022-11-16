---
id: ansible_intro
title: Introduction and User-orientation
sidebar_label: Introduction

---

- TOC
{:toc}

---

## Ansible Automation for Cloudera Products
AKA: Cloudera Deploy | Cloudera AutoProv | Cloudera Ansible Foundry | Cloudera Playbooks

## What to Expect
These docs are for new users and power users alike to leverage Cloudera’s Ansible implementation for deployment and automation of Cloudera Products within the wider Hybrid Data Cloud ecosystem.

It is generally broken into several sections:

For all users:
These introductory notes to orient new users

-- A discussion of Use Cases that the Framework supports

-- A Getting Started guide based around user goals and skill levels

-- An Overview of preparing Definitions for Deployment

For Developers:
-- A History and guiding principles

-- A breakdown of Framework Components

-- An in-depth Setup Guide

-- Logical breakdowns of key Workflows within the Playbooks

-- A Reference guide to all Definition options

## What is Cloudera Ansible Foundry
This framework is generally called the Cloudera Ansible Foundry, and it wraps software dependencies, automation code, examples, and documentation into an integrated family.

It further provides a consistent and portable toolkit for working in Python, Bash, Terraform, various CLIs, language clients, ad-hoc commands, and other commonly used tools in the Hybrid Cloud ecosystem.

##  What is Cloudera Deploy
Cloudera Deploy is the name of the reference implementation in Ansible Playbooks for automating Cloudera Products.

For most users, Cloudera-Deploy is the entrypoint and only component they need interact with, but underneath it there is an extensible framework of components for handling various scenarios for power-users.

To that end, you can use Cloudera-Deploy itself directly, or use it as a starting point for your own implementation, or simply use the components within for your own purposes.

## Licensing, Warranties and Restrictions
Cloudera-Deploy and the underlying framework are all open source, mostly under the Apache 2.0 license or compatible licenses.

The software is provided without Warranty or Guarantee of Support - it therefore differs from, and is complementary to, Software provided by Cloudera or other Vendors under commercial agreements.

However, while we do not guarantee support, Cloudera use this software along with our partners and customers, and thus strive to maintain it to the same high standards of our other products in the best spirit of community and partnership.

## Use Cases
### Simple Use Cases
These simple use cases can be run from the Cloudera-Deploy Quickstart without requiring additional skills in advance beyond use of cmdline, docker, and cloud credentials

-- Deploy Reference Architectures for CDP Public on AWS, GCP, or Azure

-- Deploying from OSX, Windows10, or Linux machines

-- Leverage Ansible or Terraform for Cloud Infrastructure

-- Deploy CDP Base clusters to most supported OS targets

-- Deploy Applications on various CDP implementations

-- Teardown and other lifecycling task

### Intermediate Use Cases
In these cases, we expect that you are a more experienced DevOps user with particular goals in mind and at a good working knowledge of the technologies in play

-- Deploy CDP Private Cloud on a Kubernetes Variant

-- Deploy Applications across multiple CDP Hybrid Cloud Environments

-- Integrate a 3rd party component into the Framework

-- Create a mirror of archive.cloudera.com or custom offline repo of parcels

-- Use the framework within Ansible Tower, or via CICD integration

-- Handle partial deployments or partial teardowns using run levels

-- Automate deployment to adopted infrastructure

-- Extend the Ansible Collections to handle additional cases

-- Force remove cloud infrastructure within a given namespace

### Framework Components
Here we will give more of a sense of what is in each of the repositories within the framework

### cloudera-deploy
This is the standard entry-point for most users, most of the time.

It contains the main Ansible Playbooks, default User Profile, Definition handling, and Run initialization.

It also usually contains other ease-of-use features for new users, like the current Dynamic Inventory implementation, until such time as they are matured into their own separate feature.

### cldr-runner
A Container image as a common Runner with all dependencies suitable for use locally, with Ansible Tower, against an IDE, or various other circumstances

It is based in a centos8 image produced by the Ansible Community, known as 'Ansible Runner'. We then resolve the difficulty of conflicting dependencies to layer in the various clients, python modules, utilities, and other bits and pieces that make it a useful shell or remote execution environment for working with Hybrid Cloud.

You can also use this as a template for your own specific implementation, but we ask Users to adopt it as their default if they don’t have a reason not too for the simple benefit of not reinventing something that works well and reduces the burden of reproducing errors.

### cdpy
A Python client for the CDP Control Plane, both Public and Private.

This is essentially a convenience wrapper for CDPCLI, which itself is based in a fork of AWSCLI and fully written in Python.

cdpy contains a large number of helper methods which are reused throughout the cloudera.cloud modules, as well as well-structured Ansible-friendly error handling.

### Cloudera Ansible Collections
cloudera.cloud
This Collection is primarily un-opinionated modules for the CDP Control Plane in Public or Private Cloud.

The point of keeping the modules unopinionated and solely covering the CDP Control Plane interactions is to minimise the dependencies and attack surface for users who aren’t using Public Cloud.

cloudera.exe
This Collection is highly opinionated Roles for most task sequences around achieving some run-level or dependency satisfaction.

The concept of run-level deployments in cloudera.exe is explained later in the Architecture.

cloudera.cluster
This Collection is focused on Deploying & Configuring Clusters via Cloudera Manager - usually traditional Cloudera clusters.

It is backwards compatible with the Cloudera Playbooks written for Ansible <2.9, whereas this Collection is for >2.10

### Example Deployment Definitions
Example Definitions for use with Cloudera-Deploy, presently bundled here

We plan on publishing more examples soon.

## Getting Started
Cloudera-Deploy is designed for use both internally and externally by Cloudera Engineering, Testing, Education, Support, Field, Sales, Marketing and other staff, but also the same code and artefacts are used by Customers, Partners, Resellers, and our Community.

As such, we have set it up so a new user can get started in as little as 3 steps, while not limiting Power Users from leveraging the full suite of capabilities.

## Prerequisites
While Cloudera-Deploy aims to remove as many complexities for the new user, there are still a few dependencies like authentication and runtimes required for it to work. The quickstart guide contains extra prerequisite steps that you are advised to follow if this is your first time using new credentials or tools such as AWS CLI.

## Understanding the Skills Gap
We recognise there is a significant jump in the skills and knowledge required to go from using the pre-configured push-button examples in the 'Simple' section to more complex cases the 'Intermediate' section and beyond.

There are several good introductory courses online for Ansible and Terraform use in Hybrid Cloud Architectures. Cloudera also offers a training program tailored to these tools to interested Customers and Partners.

You can also reach out to your Cloudera contacts for assistance in these deployments at any time.

### Personas
New to Automation
Using Cloudera-Deploy without modifying it only requires a couple of steps and some cloud credentials. It is designed to be extremely accessible to new users.

If this is your first time, we suggest running the Cloudera-Deploy Quickstart using one of the prebuilt Definitions before diving in further.

Skills required:
commandline, editing text files, cloud credential management

Create your own Definitions
Crafting Definitions is just editing text files to declare what the Deployment should look like, and allowing Cloudera-Deploy to interpret and produce what you have described. The files are written in YAML with a simple structure.

Once you have worked out how to run a basic Deployment via the Quickstart, you may wish to customise it to meet your requirements. A summary of editing Definitions is included in the Getting Started sections for CDP Public Cloud and CDP Private Cloud.

There is also a detailed explanation of how Definitions work in the Deployments Reference in the Developer’s Guide, though reading the rest of these Docs and examining the other Definition examples may also be helpful.

Additional skills required:
YAML editing and formatting, Cloudera-Deploy Definitions management

Create your own Playbooks
Creating your own Playbooks requires a working understanding of authoring Ansible tasks and how to use Modules and Roles from Ansible Collections. It goes a step beyond editing YAML declarations into understanding the sequences of steps necessary to achieve a given outcome in a hopefully robust manner.

We will soon be publishing more examples of Application Playbooks on top of Cloudera-Deploy, but you can review the Playbooks already included as a starting point.

We suggest you start by fully reading this documentation including the Developer sections and how the existing Workflows are structured.

Additional skills required:
Ansible Task development, use of Ansible Collections, advanced use of Ansible Tags, experience with multi-tier automation abstractions

Extend the Framework
If you are familiar with Ansible and Cloudera Products, and are considering adding Ansible modules or Roles to the Collections, then you may wish to follow the Developer Setup to fork and checkout the Framework components so you can make (and perhaps contribute back) your own changes.

Additional skills required:
Python development, Ansible test frameworks and debugging, Docker Containers
