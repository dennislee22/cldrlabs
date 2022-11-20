---
id: ansible_info
title: Information-oriented References
sidebar_label: References

---

- TOC
{:toc}

## Background

In this section of the documentation we will attempt to collate answers to commonly asked questions about the Why and How things are built the way they are.

In general, we try to include comments directly in the code to explain why some particular thing may be done a particular way. Please raise an [Issue](https://github.com/cloudera-labs/cloudera-labs.github.io/issues) if there is some step in the tasks which you want more explanation on, or if you want some workflow to have an explanation in these docs.

### Guiding Principles

1. Good Defaults don't produce Bad Surprises
2. Least-worst is usually better than best
3. The needs of the many generally outweigh the needs of the few
4. It should work out of the box
5. It should be secure by default
6. Provide examples and as much freedom as possible
7. Fix it upstream if you can

### History

This codebase is the 4th or 5th generation tooling for automating the Platform that, in the current incarnation, is called CDP or the Cloudera Data Platform.

When Ambari and Cloudera Manager (amongst others) where early orchestrators-of-clusters, there were quickly deployers-of-orchestrators being published, such as [Ambari-Bootstrap](https://github.com/seanorama/ambari-bootstrap) and many others. These early efforts were simple and effective, and typically in Bash or Perl.

Later on, when management of deployment to IaaS was being developed in Director and Cloudbreak, Procedural management tooling like Saltstack, Puppet, Ansible and Chef were battling for adoption and dominance in deployment and configuration management.

As cloud infrastructure APIs became more deterministic, and cloud standards more codified, Terraform rose to prominence with the Declarative IaC movement. Now you could combine declarative infrastructure with procedural deployments, and people started to appreciate the combination of Terraform and Ansible as powerful system tools.

Around this time Cloudera first published our [Ansible Playbooks](https://github.com/cloudera/cloudera-playbook) for deploying Cloudera Clusters. While we still also use other tools within our systems, we settled on Ansible as the most broadly adoptable and adaptable tool to work with our Customers when integrating our Platform with their extremely varied Environments.

With the rise of Container Orchestration like Kubernetes, yet another maturity level in systems management became apparent - a stack building up to declarative applications with self-healing and auto-scaling possibilities. Now maturing valuable APIs for determinism and observability became more critical, while still offering suitable integration for users of varying modernity and maturity stages at the edge of the zone of control.

So, today, Cloudera offers many entrypoints into integrating with our Hybrid Data Platform, from deploying legacy versions of our traditional cluster management software, to deploying real-time data streaming analytics on managed kubernetes in various cloud-native providers.

We continue to provide Ansible as our general purpose Procedural automation framework, as it can generally be adapted to most situations as a good starting point. We wrap in Terraform as an option for Infrastructure, as the widely recognised industry standard, and provide the usual collection of SDKs and API specifications underneath.

### Declarative vs Procedural

We have taken the approach of implementing a much more Declarative approach to Ansible in this framework. It is not particularly traditional to use Ansible in this way prior to ~v2.10, and you might say why not use a built-for-purpose Declarative tool like Terraform instead.

The main answer to this could be summarised as follows: Ansible is better at discovering and modifying deployments, whereas Terraform prefers to own and modify them. The difference is important as, in designing these automations, we cannot know whether a given user has full control of the deployment over time. It is quite easy to get your Terraform state into a bad place if people can unexpectedly edit the deployment directly (as often happens in complex tiered applications with shared-responsibility models), whereas Ansible is comfortable with taking things as it finds them.

Secondly, we cannot know or support how 3rd party APIs and software versions may change over time with respect to a users' deployment. There are too many permutations of possible deployments to support, and so we again need a framework that is more flexible and may be adapted by any given user to their circumstances.

Thirdly, we find that the broad set of APIs and systems that CDP is integrated with have better coverage in Ansible Modules and Collections than Terraform Providers, and when that coverage isn't available, it's relatively easier to fall back to a CLI or API request implemented in Ansible.

That is not to say Terraform is somehow bad or inappropriate - actually we strongly recommend that, if you have good authorization controls over your Infrastructure, Terraform is the best way to manage it. Our point is more that, if you have this you probably already have good DevOps skills and can gracefully adopt this Framework, whereas Ansible is going to be lower friction for everyone earlier in their journey.

All that being said however, the more of your deployment you can manage declaratively _probably_ the easier your life will be - the art of the science here is knowing when to switch tools and why.

NOTE: Ansible and Terraform - Better Together

### Implicit with defaults vs explicit

This framework strives to have a clear and singular default value for as many of the variables in the platform as possible, and those default values should represent the balance between best practice secured deployments and working easily out-of-the-box for new users.

The defaults should not produce results that surprise the User with Bad Things, like accidentally deleting data or secrets.

All defaults should be able to be overridden without difficulty by users who know what they want.

The purpose of this approach is primarily to enable new users to be productive with a minimum of front-loaded learning, through the means of deployments requiring a minimum of configuration, but allowing a maximum of explicit configurability as user skill and confidence grows.

So, you can have a deployment from defaults with almost no fore-knowledge of how CDP works (like our Trial), but you can also use a 400 line declarative YAML file to describe your whole deployment (like our QE) and _both options are perfectly valid and use exactly the same tooling_

### Multi-tier abstraction

In order to deliver on this more complex implementation, it is necessary to pay the cost of several layers of abstraction.

Some of these are implemented directly here, but where possible we use upstream vendor implementations, e.g. cloud provider maintained Terraform Providers or Ansible Collections.

1. Base clients (CDPCLI, Azure CLI, AWS CLI, etc.)
2. Procedural clients (cdpy)
3. Declarative Modules (`cloudera.cloud`, `amazon.community`, etc.)
4. Sequential Tasklists (`cloudera.exe`, `geerlingguy.postgresql`)
5. Playbooks (cloudera-deploy)
6. Declarative Definitions (definitions)

Generally we aim to push some complex task or piece of logic towards the best implementation point, for example, it is much easier to do a complex map or filter operation in Python within an Ansible Module than to express it within an Ansible Jinja template in a task within a Role.

Most users will not need to bother themselves with any of this; they will craft Definitions, apply them with the Playbooks and satisfy their requirements. But for Developers, you will need to develop a sense of when something should be upstream in the Base Clients, put into cdpy or the Modules as reusable Python code, or be expressed in some Ansible Task within a particular sequence.

Feel free to raise an Issue on the repositories for guidance in these cases.

### Ansible Collections and the Handling of Variables

Possibly the single most frustrating thing in Ansible is when you can't figure out where a variable was defined, or why it isn't the value you expect to be because someone has squashed it somewhere else.

In this Framework, unless you can't for reasons of backwards compatibility, all variable names should be prefixed.

If the variable is local, and only not intended to be reused, then we follow the Python standard of a single or double underscore. This is particularly important in naming trivial variables such as those used in loops. They must be unique, and it is lazy not to do this, e.g. `__bag_of_holding_item`

If the variable is likely to be referenced elsewhere within that specific Role, assign the whole Role some short but obvious prefix followed by a double underscore to indicate ownership, and then uniquely name variables e.g. `hg__wingardium_leviosaaaa`

If the user defined variable is needed across multiple Roles in a Collection, then make a `common` Role in the Collection and have the other Roles pull the variables from the defaults part of this common Role. `cloudera.exe` makes extensive use of this, with the common defaults [here](https://github.com/cloudera-labs/cloudera.exe/blob/main/roles/common/defaults/main.yml), and the infrastructure Role importing them [here](https://github.com/cloudera-labs/cloudera.exe/blob/main/roles/infrastructure/defaults/main.yml).

If the variable is discovered for a Role, such as checking whether a resource exists on a given cloud provider, then it should be discovered anew for each Role and variables from other Roles should not be assumed to exist. This allows the Roles to be used separately if desired, or skipped using `--skip-tags` for brevity of Runs.

All significant variables in a Role that the end-user is likely to care about should be defined in defaults for that Role, using the prefixed names, and imported from common where necessary. In this way 99.9% of stupidly annoying variable issues may be avoided, and the rest can probably be blamed on Azure consistency errors.

Full credit to Webster Mudge for this robust design.

### Cloud Infrastructure and the Naming of Objects

Naming things is the second-hardest problem in DevOps.

In this Framework we rely extensively on procedurally naming objects and complying with the various and varying restrictions on uniqueness, length, character sets, immutability, and other weird requirements that emerge from the world of Hybrid Cloud Infrastructure.

In practical terms, we try to stick to a known-good intersection of requirements that work.

As such, we have settled on assigning default `label` strings to most conceptual components, and `suffix` for most classes of objects to be created. When combined with the `name_prefix` for that Deployment, and some other sources of uniqueness like the `infra_type`, we find we can generate meaningful names for almost anything while still allowing the user to modify as they see fit.

You can find most of these in the [cloudera.exe.common](https://github.com/cloudera-labs/cloudera.exe/blob/main/roles/common/defaults/main.yml) Role defaults.

You can also statically assign names to almost anything, but then you are responsible for investigating the impact that might have.

Generally shorter names are better, particularly when something might be used to construct a FQDN, which must generally be <63 chars to be safe.

Generally avoid punctuation, particularly at the start or end of a name, as we have found that different cloud providers and their subsystems will fail unexpectedly when they encounter a double underscore or double hyphen. In most cases a single hyphen or underscore that is not the first or last character are allowed.

Starting a name with a number can also sometimes cause strange subsystem errors.

Generally stick to basic UTF8 characters - while some systems will allow you to explore the exciting depths of UTF32, many others will fail terribly.

Our guidance in these notes is there to help you avoid such difficult errors, please heed it.

## Terminology
Some terms used herein have specific meaning, and are usually Capitalised in the document to indicate they are being used in that specific context.

**Playbooks**
Are referring to Ansible Playbooks, and are generally the main entrypoint into Runs

**Run**
Some execution of a Playbook

NOTE: When using the Runner, Runs are automatically logged back to the user profile and use a collection of default settings known to be good in most situations

**Controller**
The Controller, or Ansible Controller, refers to the machine where the `ansible` or `ansible-playbook` etc. commands are being executed, as distinct from the 'Ansible Inventory' which are the hosts that Ansible is connecting to.

**Runner**
Refers to [cldr-runner](https://github.com/cloudera-labs/cldr-runner), a common Execution Environment. This is built from a Dockerfile maintained by the Ansible community, to which the various dependencies for CDP and Hybrid Cloud Architectures are added.

**Definition**
A directory containing files expected by the Playbooks which describe the Deployment. The Definition directory also doubles as a working directory for artefacts produced in the Run. The files and details around composability will be explained later.

**Deployment Definitions**
Refers to one or more of these Definition directories that may be provided by default with Cloudera-Deploy, created by the user, or some other process

**Tags**
Tags is an overloaded term, and may be referring to:

* 'Ansible Tags', which control what actions will be executed within the Run based on the Definition. At the simplest level this is things like ‘deploy’ or ‘teardown’ but can provide a great deal of control with sophisticated use
* 'Tags' applied to cloud infrastructure, which is strongly recommended for all users

**Profile**
Profile is an overloaded term and may refer to any of the following depending on context:

* A profile specifically for Cloudera-Deploy which provides the lowest precedence of user defaults for things like Passwords. Usually found in `~/.config/cloudera-deploy/profiles` on Linux machines
* A profile used with some external API, such as AWS, Azure, GCP or CDP, which usually specifies things like credentials, endpoints, regions, etc. Usually found in `~/.aws` or similar on Linux machines
* Sometimes used to refer to your user home directory on your machine. When using the Runner, it mounts key User home directories such as your .ssh and .config folders in provide you access to those files with the various tools. Usually `/home/<user>` on Linux machines.

**Subsystems**

Modern Platforms are layers upon layers of different bits of Software and Infrastructure working together.

When we refer to Subsystems, we may mean a Cloud Provider in general, or some specific API, or an Apache Project. It is not necessarily Software provided or written by Cloudera, but we will attempt to give you good information about it.

**Workflows**

There are many Processes within the Framework which touch on many levels of automation abstraction and may operate end-to-end over many files and subsystems. We refer to these as Workflows as a convenient label to indicate it's a bit more complex than a simple Ansible Task list or Playbook, and Workflows have their own section in the Developer's Guide.


## Deployments Reference

A ‘Definition’ is a directory containing one or more files, and may contain additional assets required by the user. The intention is that a single directory may contain all the necessary components to describe a Deployment suitable for configuration-as-code practices using version control.

This also means that pre-defined Definitions are easy to create, and we include several for new users. See explanations in the sections for CDP Public Cloud and CDP Private Cloud.

### Definition Structure

_**The definition_path**_

Cloudera-Deploy will recognise and compose together several files during Initialization in order to determine what should be done in a run.

The files have defaults set in `cloudera-deploy/roles/cloudera_deploy/defaults/main.yml` and the various Collections, which may be overridden by the Definition or with extra vars at runtime.

The Definition directory also acts as a working directory during the run, and is the location where Definition-specific artefacts, such as dynamic inventory files, will be written back to. This differs from persistent records for the Runner like Logs and Profiles which are kept in `~/.config/cloudera-deploy` and thus written back to the Filesystem hosting the Runner.

As such, the `definition_path` extra var is one of the few mandatory values required by the Playbook, and one of the first things checked for in a Run.

It is expected that the user will have any persistent information from their profile, like ssh keys and cloud credentials and logs, mounted via bash expansion in `~/`, and any Code and Projects mounted for persistence to /runner/project. Pretty much all other areas of the Runner are presumed to be ephemeral storage.

_**Explaining the various Definition Files and Profiles**_

As said above, Cloudera-Deploy will read in and compose the facts for the run from a set of files found in the Definition path, then from an optional Profile file, and then from the Collections which hold defaults for the different kinds of deployments available.

The various files and their override values are set during [cloudera-deploy initialization](https://github.com/cloudera-labs/cloudera-deploy/blob/main/roles/cloudera_deploy/tasks/init.yml). We pull these values into multiple different files to give the user control of composing their deployments if desired - you may wish to recombine different profiles, definitions, and cluster specs according to your needs, but you can also just glob it all into a single `definition.yaml` file in most cases if you want.

The Profile is drawn from the directory `~/.config/cloudera-deploy/profiles`. The default file collected is simply called `default` in this directory, and is automatically created from the `profile.yml` template in the cloudera-deploy repo.

Then the user may choose to put all the rest of their definition details into the `definition.yml` file in their `definition_path`, they may also choose to split it across `definition.yml` and `cluster.yml`, for backwards compatibility with legacy Cloudera Playbooks.

The dynamic_inventory files are explained in a later section.

_**Required Definition Files**_

There are 2 files that must be present in the definition path:

* A Definition for the deployment, usually cluster.yml or definition.yml
* a post-run playbook, with the name application.yml

You may optionally supply other files recognised by the Definition parser, such as an inventory template or static ini file.

_**cluster.yml or definition.yml, or split between both**_

This file contains the variables that define:

* The CDP Cluster(s) - which services, service configuration parameters, host templates & Role allocation, external databases, runtime repositories, security
* Cloudera Manager - cm repository, management services, CM configs
* Host configurations - e.g. values set via CM -> All Hosts -> Configuration
* Auth backends - e.g. ldap/ad
* Any other vars required for the Plays in-scope (e.g. kdc, haproxy settings)

_**Application.yml**_

Used to describe additional deployment steps. Although this file must be present, it doesn’t have to do anything and you can simply copy the sample no-op Playbook from the examples/sandbox in Cloudera-Deploy.

_**Your Profile file(s)**_

If you take a look in the Profile.yml Template, you will see it is a collection of keys in YAML format, these are the keys most commonly personalised to a given user, such as passwords and naming prefixes.

The main cloudera-deploy https://github.com/cloudera-labs/cloudera-deploy/blob/main/readme.adoc[readme] does a good job of explaining the different values in this file, and note that you can also set these keys in your definition or cluster files if you wish, but be aware of precedence if you have them set in multiple files.

_**definition.yml file**_

definition.yml is the main file the user is anticipated to modify in order to define what their Deployment should look like. This is then combined with the Ansible tags used at runtime in order to produce the expected result.

For example, you could define a CML deployment in your definition.yml but use the ‘infra’ tag with Ansible to only create the Infrastructure dependencies for CML, but not actually deploy CML itself. This gives the user a great deal of flexibility and operational control.

If you wish you may set all of your YAML into the definition.yml file and skip profile, cluster, etc. This may be useful for CICD deployments.

_**cluster.yml file**_

Although cluster.yml is usually provided externally to the Definition, it can also be included in the definition for two main reasons:

Firstly, the v2 edition of the Cloudera Playbooks expects a cluster.yml definition file, so this provides convenient backwards compatibility and a point of comfortable continuity for existing users migrating to the v3 / Collections based approach.

Secondarily, certain complex cluster definitions typically used with the v2 playbook depended upon advanced Ansible lazy-evaluation tricks in order to replace values within the configs, such as hostnames and service descriptors mid-deployment. cluster.yml is specifically lazy evaluated (unlike definition.yml, which is evaluated almost immediately) to allow these deployments to continue to work. We do not expect new users to need these tricks, so this feature is primarily for backwards compatibility.

There is a basic cluster definition included in the cloudera-deploy https://github.com/cloudera-labs/cloudera-deploy/blob/main/roles/cloudera_deploy/defaults/basic_cluster.yml[defaults], you may instruct it to be included by setting the value `use_default_cluster_definition: yes` in your definition.yml, or you could copy it into your `definition.yml` or `cluster.yml` file in your Definition directory and then use it as a starting point to get going with customising your deployment.

_**Static_inventory.ini**_

You may also include a typical Ansible static inventory file within the Definition to be automatically loaded and used for the run, this is in addition to any inventory files in the default Ansible Inventory directory (which are typically empty).

There is an example of a Static Inventory https://github.com/cloudera-labs/cloudera-deploy/blob/main/examples/sandbox/inventory_static.example[here], it is notable that the Dynamic Inventory option will generate a static inventory file for you as a part of the process, and tear it down along with the rest of the infrastructure if you use the ‘teardown’ tag.

You may also wish to use traditional Ansible dynamic inventory implementations, or the built-in dynamic inventory generator for AWS documented xref:cdDynamicInventory[Dynamic Inventory]

### Crafting your Definition

In all simple cases, Cloudera-Deploy aims to have defaults for practically everything but your administrator password, allowing you to then only override any parameters that provide a specific change you want.

We recommend, but do not require, that you also set the values in your default user profile as explained in the readme, because otherwise your infrastructure is unlikely to be uniquely tagged and named within the cloud infrastructure account (assuming you are using these features).

However, once you have set those defaults, you will likely want to describe the actual deployment you require.

There are several examples given within cloudera-deploy, such as the Sandbox, a CML Workshop, or all the Cloudera DataFlow services. These are all primarily public cloud examples, although the Sandbox does include a basic private cloud cluster as well.

Generally, CDP Public cloud parameters follow a simple structure of some top-level key triggering a particular component to be deployed, along with whatever dependencies it needs, and then any child keys under that top-level key controlling some override of some default. These keys are explained in xref:cdSchemaReference[CDP Public Definition Keys].

Generally, CDP Private Cloud is a more haphazard structure, simply because it is extremely configurable with nearly a decade of history and backwards compatibility, and therefore trying to constrain it to a pretty structure causes more problems than it fixes within the automation.


### CDP Public Definitions

**Summary**

We aim to keep a full dictionary of all the YAML structure in the [cloudera.exe docs](https://github.com/cloudera-labs/cloudera.exe/blob/main/docs/configuration.yml), there are literally hundreds of parameters that are defaulted and may be overridden, so in this section we will attempt to explain the key values to achieve common goals.

Generally the structure is in dot notation within a collected set of default files, one to each Role. If you can find the parameter you are interested in, do a global search within that Role (preferably in your IDE, but also in the repo), and you will likely be able to quickly backtrace how it was generated and find the default.

Defaults are usually in the Role which requires them, and will otherwise be moved up to `cloudera.exe.common` if more than one Role needs to use the same values. For example, we create the subnets in `cloudera.exe.infra`, but need to supply them to the `cloudera.exe.platform` and `cloudera.exe.runtime` Roles as well, so they are sourced from the common defaults.

Defaults in `cloudera.exe.common` may also be overridden by values from whatever called it, via the `globals` mechanism. This can be observed within Cloudera-Deploy, which contains its own defaults, then reads in the User supplied Definition, then passes the values set down to the Collections.

As another example, the key `infra.teardown.delete_data` occurs within the Infrastructure defaults in the `cloudera.exe.infrastructure` Role, and it is defaulted to `False`. If you set it to `True`, when you run a `teardown` cloudera-deploy will also clean the targeted storage. You include it in your definition by converting it to YAML:

```yaml
infra:
  teardown:
    delete_data: yes
```

The second structure we typically use is nested dictionaries keyed by some common variable like Cloud Infrastructure Type, and contained in Ansible vars within Roles. This allows us to set defaults for each infrastructure provider and various types of things we might need, such as virtual machine sizes and disk types. A good example of this may be found in the [cloudera.exe.infrastructure](https://github.com/cloudera-labs/cloudera.exe/blob/main/roles/infrastructure/vars/main.yml) Role. Again, global searching for the parameters will show you how they are used within the deployment structure.

What is important to note here is that we attempt to allow the user to define the least amount of information necessary to produce the target Deployment, something sometimes refered to as Goal Based Declarative Deployment. You can set as many overrides as you like, but you then take on responsibility for knowing how they will interact, whereas if you use the defaults they are already known to be best-practice.

**Infrastructure**

Controlled under the ‘infra’ key in all the underlying cloud infrastructure services you could need, most defaults are either in the [cloudera.exe.common](https://github.com/cloudera-labs/cloudera.exe/blob/main/roles/common/defaults/main.yml) or [cloudera.exe.infra](https://github.com/cloudera-labs/cloudera.exe/blob/main/roles/infrastructure/defaults/main.yml) defaults files. It is worth examining these defaults if you are interested in which subnet CIDRs, storage types, and other values are used in deployment.

If you include the `infra` key in your definition without anything else, cloudera-deploy will create a base set of infrastructure (network, storage, security groups, etc.) on your chosen provider when you use one of the deployment triggering tags.

.e.g. To create Infrastructure for CDP Public on Azure:

```yaml
infra_type: azure
infra:
```

**CDP Public Environment & Datalake**

This is the core CDP Public ‘platform’ to which the `plat` Ansible Tag refers, and is controlled by the top-level YAML key `env`. Apart from the common defaults mentioned earlier, there are platform specific defaults as well in https://github.com/cloudera-labs/cloudera.exe/blob/main/roles/platform/defaults/main.yml[cloudera.exe.platform].

As with other top level keys, if you include `env:` on its own, then cloudera-deploy will generate an infrastructure suitable for CDP Public and Private based on your Definition, Profile, and other defaults.

There are a large number of defaults in this Role, you are advised to scroll through the defaults file above - we resist listing them all here to avoid additional maintenance burden.

.e.g. To create a CDP Public Infrastructure on GCP with a Datalake pinned to a particular version:
```yaml
infra_type: gcp
env:
  datalake:
    version: 7.2.12
```

**Datahubs**

`datahub:` is the next key we look at, and the first that requires some child keys to work.

As with env, if you have datahub in your definition file cloudera-deploy will ensure that the requisite Infrastructure and CDP Public Platform are deployed.

For datahubs, you are required to submit a list of configurations, one for each datahub you want to be present. You have three options:

* You may give the name of a definition to deploy it from defaults
* You may refer to a jinja template
* You may explicitly set the necessary values

.e.g. To Create an OpDB Datahub, and a Kafka Datahub on AWS (default):

```yaml
datahub:
  definitions:
    - definition: ‘Operational Database with SQL for AWS’
      suffix: cod-dhub
    - include: "datahub_streams_messaging_light.j2"
```

**dw**
You can include the top-level key `dw:` in your definition to deploy a default CDW cluster

More functionality for this service is coming soon

**ml**
CML has a nice [example definition](https://github.com/cloudera-labs/cloudera-deploy/blob/main/examples/cml/definition.yml), which is again a top level key `ml:` with a child key ‘definitions’ which is a list of workspaces to create, either from defaults or with specific parameters.

If you include ‘ml’ without any child keys a single default workspace will be created.

**df**
Like CML, CDF also has an [example definition](https://github.com/cloudera-labs/cloudera-deploy/blob/main/examples/cdf/definition.yml) which works in a similar fashion.

**de**
Same as above, here is the [example definition](https://github.com/cloudera-labs/cloudera-deploy/blob/main/examples/cde/definition.yml).

**opdb**
Same as above, the example is coming soon.

## Workflow Reference Guide

Many of these Architecture notes explain the sequence of steps in various parts of the codebase. It is not necessary to understand these processes in order to use Cloudera-Deploy for Deployments, but if you want to make your own processes they may be quite informative.

#### quickstart.sh

[Quickstart.sh](https://github.com/cloudera-labs/cloudera-deploy/blob/main/quickstart.sh) is provided in the root of the Cloudera-Deploy repository, itself being the reference entrypoint to the tooling. It is intended to prepare and launch you into a shell running inside the Docker execution container where all the necessary software is already prepared for.

You'll know you are in this custom shell by the Orange coloured `cldr <version> #>` prompt.

When you invoke quickstart.sh in Cloudera-Deploy, here is the general sequence of steps

1. It determines the parent directory of quickstart.sh, and sets that as the Project directory unless the user has passed in a variable for this purpose
2. Checks if Docker is running
3. Runs ‘docker pull’ against the image if there is an updated image available
4. Creates the default local User Profile mount paths on the local machine if not present
5. Creates a default Cloudera Deploy Profile in the default path if not present
6. Checks if various development helper options are set:
    * If `CLDR_COLLECTION_PATH` is set, that path is put into the ANSIBLE_COLLECTIONS_PATH, and the user is instructed to use /runner/project as the base path instead of /opt/
    * If `CLDR_PYTHON_PATH` is set, then that path is added to the PYTHON_PATH in the internal container
7. Checks if ssh-agent and ssh-auth-sock are running / available
8. Then:
    * If the Runner container is present and running, a new bash session is started
    * If the Runner container is present but stopped, it is removed
    * If the Runner container is not present or removed by the previous step, a new one is instantiated and a bash session started
9. The current release version of cloudera-deploy is cloned into /opt/cloudera-deploy alongside the other Ansible Collection pieces
10. The user is given several notices and dropped into an orange coloured bash prompt in the /runner directory

IMPORTANT: The only way to change the /runner/project directory mounted into the container is to stop the container and rerun the quickstart with a different path as the argument.

NOTE: The Ansible Log is pointed at ~/.config/cloudera-deploy/log by the quickstart container launcher

### Main Playbook in Cloudera-Deploy

The [main.yml](https://github.com/cloudera-labs/cloudera-deploy/blob/main/main.yml) Playbook is also in the root of Cloudera-Deploy and is intended as the entrypoint that you would call with the `ansible-playbook` command in the orange bash prompt we provide you in the Quickstart.

IMPORTANT: If you invoke Ansible in a directory other than /runner within cldr-runner, behavior will change as Ansible checks for particular directories relative to the execution path and uses that to find some defaults. Yes this is often annoying, they seem resistant to changing it.

The main Playbook is generally invoked from one of two locations:

`/opt/cloudera-deploy/main.yml`

This is the current release version of Cloudera-Deploy baked into the Runner itself

`/runner/project/cloudera-deploy/main.yml`

If you have launched quickstart.sh following the usual process, then this is the version of Cloudera-Deploy you have checked out from Github on your local machine, as your Project directory is mounted to /runner/project, and thus cloudera-deploy will be in that project directory.

If you are editing the Playbooks or Collections, you should launch from /runner/project to pick up your changes.

Here is the general sequence of steps:

1. Ansible initialisation
    * /runner/env/envvars are loaded
    * Any Ansible Inventory in /runner/inventory is read

2. The init Role is run
    * Run Metadata is immediately logged
    * The Definition Path and Files are validated
    * The User Profile is included, making it the lowest in Definition precedence
    * We validate that an Admin Password has been supplied
    * We examine the presence and controls around definition.yml and cluster.yml
    * definition.yml is included, making it the second in precedence
    * The selected cluster.yml is included (if present), making it third and highest in precedence for Definition files
    * ‘globals’ is then generated from general top level facts found in files included thus far
    * ‘globals’ is then updated by any values explicitly set as ‘globals’ from definition.yml, making this the highest definition authority only trumped by extra vars following the usual Ansible hierarchy
    * The Namespace is then validated and set
        * Note that this is not a namespace in the sense of Kubernetes, but the name_prefix that is used for all resources generated by Cloudera-Deploy
    * SSH keys are then validated, if necessary created, and then set
    * If a Dynamic Inventory is found, it is then validated and parsed
    * If the Download Mirror has been requested and requirements are met, the prepare_download_mirror Role is then included
    * Temp directories, various Environment Variables for the Run, and other details are finally set before continuing
    * The Cloud Playbook is then imported
    * If Cloud actions are indicated in this Run, the Role `cloudera.exe.sequence` is imported
    * Any Dynamic Inventory result is persisted to Static Inventory artefact for later use
    * If the Download Mirror is in Play, it is then populated and the cached files then persisted to the Download Mirror cache file for later use
    * If a Teardown has been requested, the clean_dynamic_inventory Role is run
    * Then the preparation steps for a Cluster deployment are run if a Cluster definition was found during init’s parsing of the Definition
        * The static inventory artefact is loaded, if found (including when just created)
        * The Download Mirror URLs are injected from the cache artefact, if requested
        * Necessary facts for the Cluster deployment are distributed to the Inventory based on the Run thus far
    * If a Cluster is defined, the Cluster Playbook is then imported
    * The Application Playbook is imported from the Definition directory for any post-run tasks

### Cloud Deployment Sequencing

The [cloud.yml](https://github.com/cloudera-labs/cloudera-deploy/blob/main/cloud.yml) Playbook in Cloudera-Deploy acts as a wrapper for steps that are taken against Public Cloud Infrastructure.

NOTE: cloud.yml is expected be called within the main.yml workflow, and therefore expects that certain Ansible Variables are available. If you wish to modify or re-use behavior, you are advised to trace the Tasks from the start of main.yml to understand the process.

The following steps are included:

1. The `cloudera.exe.sequence` Role is imported if Cloud Infra or CDP Public Cloud included in the Definition
2. If Dynamic Inventory has been created in the cloud steps, the details about it are persisted
3. If the Download Mirror is in the Definition, it is prepared and the target files are downloaded to it
4. The Download Mirror cache is then updated if any new downloads have been completed

### Cluster Deployment Sequencing

The [cluster.yml](https://github.com/cloudera-labs/cloudera-deploy/blob/main/cluster.yml) Playbook controls the sequence of Plays which deploy a CDP Base cluster, or legacy CDH cluster.

NOTE: cluster.yml is expected be called within the main.yml workflow, and therefore expects that certain Ansible Variables are available. If you wish to modify or re-use behavior, you are advised to trace the Tasks from the start of main.yml to understand the process.

It is a long Playbook at ~580 lines, and has many tags to activate or skip plays. There are Tags which control full sequences of Plays, specifically those which trigger a `full_cluster` build, a `default_cluster` build, or `teardown_all` to remove the deployed Clusters (and differs from the very top level `teardown` tag for Cloudera Deploy)

Tags to trigger individual Plays are best sourced from reading the individual Plays in cluster.yml

The main reason it is broken up into so many plays is that different tasks must happen in a particular sequence on a particular host or set of hosts at any given point. You are advised not to reorder the Plays unless you are quite familiar with the process.

There are a number of 'blocks' of Plays in the Playbook to help the user orient themselves, they are annotated in comments.

The general Deployment sequence is:
1. `Init Cluster Deployment Tasks`
    * Prepare any local directories or other items needed on the Controller
    * Generally required where files from the nodes need to come back to the Controller for the user to do something with, like handling certificate signing requests
2. `Check Inventory Connectivity`
    * Validate that the Inventory given to the Ansible Controller can actually be connected to
    * Then collects the Ansible Facts from this inventory for use later
3. `Verify Inventory and Definition`
    * We run various validation steps on the Inventory and the Cluster Definition to attempt to fail early if known misconfigurations are present
    * If a node is designated to host a `custom_repo`, then the Play `Install custom parcel repository` is run at this point so that the Repo may be populated with the various files and then validated against the provided Definition
4. `Prepare Nodes`
    * Having validated our Definition and prepared our Run, we now move onto preparing the nodes that will form the cluster
    * We first apply the OS prerequisites such as huge pages and reconfiguring selinux
    * We then create any local user accounts needed on the nodes
    * The JDK is installed on the relevant nodes
    * Depending on which Database is Defined for Cloudera Manager, the clients and other prequisites for that Database will be prepared
5. `Create Cluster Service Infrastructure`
    * If you've designated nodes for KDC, Certificate Authority or HAProxy, those nodes will be prepared
6. `Prepare TLS`
    * Having prepared the ca_server for hosting the Certificate Authority above, this section then prepares the TLS keystores and truststores from the certificates if necessary
7. `Install Cluster Service Infrastructure`
    * Having completed the necessary OS and other prereqs, we are now ready to start installing services
    * First the selected RDBMS is installed on the `db_server` node if required by the Definition
    * Then we setup the specific TLS setup for NiFi if it is required
8. `Install Cloudera Manager`
    * First the Cloudera Manager Daemons are installed
    * Then the Cloudera Manager Server is installed
    * If a Cloudera License is included in the Definition, it will be applied to the server, otherwise it will remain in Trial mode
    * Then the Cloudera Manager agents are installed
    * If TLS is enabled, Cloudera Manager is configured for TLS using the previously prepared configs
    * Finally in the CM install, the various agent, server and auth configurations are layered in
9. `Prepare Security`
    * At this stage, if Auto-TLS is enabled in the Definition, it is applied to Cloudera Manager
    * We then complete the Kerberos configuration for Cloudera Manager and the Cluster if necessary
10. `Install Cluster`
    * Next we move onto installing the Cluster within Cloudera Manager
    * Firstly, we restart the CM Agents and ensure they are up and heartbeating
    * We then install and start the Cloudera Manager Management Service
    * If a `custom_repo` is in the Inventory, we then check for any parcels which could be preloaded to Cloudera Manager to speed up launch time
    * Finally in this section, we deploy the Clusters listed in the Definition
    * This involves importing the Cluster Template, which can take a long time
11. `Setup HDFS Encryption`
    * If KTS and or KMS are in the Inventory ahd Definition, they will now be configured
    * Finally Client Configs are refreshed to fix up any stale entries

The general Teardown sequence is:
    * Note that the Teardown Plays follow the Deployment plays in the Playbook, so you'll see a lot of skipped plays when calling these specific teardowns
        ** If you have Tagged a `ca_server` teardown, it is the first to be run
        ** Then, if TLS is in your Definition, it will be cleaned up
        ** Finally, if you have Tagged to teardown one or more or all of the Clusters in Cloudera Manager, they will be removed

NOTE: There are no Teardown options for the stages preceding cluster deployment, as it is assumed that you would simply reinitialise the nodes and rerun a deployment to ensure a clean build.

### Dynamic Inventory

The Dynamic Inventory implementation in cloudera-deploy is primarily intended as an aid to automated testing or trial deployments. It allows the user to provide a template inventory file, which is then converted into a number of VMs on the infra platform, which are in turn used within the same run as inventory for the CDP Private Cloud Base deployment, without any further interaction from the user.

Note that this is not necessarily a typical approach, where a user may prefer to create infrastructure using Terraform and then supply Ansible with a static inventory file. Or a user may wish to use a traditional Ansible dynamic inventory plugin. This functionality within Cloudera-Deploy is designed to make it abundantly easy for users new to these technologies to get going with minimal initial learning.

Presently this feature is only implemented for AWS, though extending each of the code sections for any other infrastructure provider is quite possible if the process below is understood. It is one of the more complex workflows in cloudera-deploy simply because it touches on several Roles and collections in order to avoid repetition and each segment of the work involved being executed with other similar workflows.

The process is deliberately broken up into sections with artefact files at key points to allow the user to substitute their own process or inventory at any given stage. It also makes testing changes a lot easier, and it may then be replaced using Terraform or some other provider at your convenience.

NOTE: Dynamic Inventory uses AMIs from the AWS Marketplace, it is necessary for someone to have subscribed to the selected image at least once within the account for it to work. If you are the first person to run this in an account you will get an error and a link to activate the subscription, then you can re-run to complete the deployment. Yay idempotence, as this can only be done in the GUI.

The meta structure specific to this feature is as follows:

1. Include an inventory_template.ini in your Definition path, which describes the shape of the infrastructure required for your CDP Base Cluster.
2. Use a supported infra provider, currently this is only AWS
3. When the main.yml playbook is run, the cloudera-deploy/init Role will pick up and process the template to determine validity and the number of hosts needed for the deployment.
    * Note that the name of the template file may be modified
    * Then existence of the template file is checked
    * We then check if a static inventory has been supplied with the -i switch, and do not process the dynamic inventory if it is set
    * If the above condition is satisfied, we then use the Ansible ‘refresh inventory’ command with the template included in the inventory dir, this is a tasklist called ‘refresh_inventory’.
    * Note that this is a minor hack as traditional Ansible usage would recommend against dynamic inventory refresh mid run, but we use it here in the name of NUX.
    * We then check some simple compliance details within the template, and if it passes we count the number of hosts required in the template and add it to the ‘globals’ variables as ‘vm.count’, which is in turn used later to generate the required host infrastructure.
    * We then remove the dynamic inventory template from the Ansible Inventory, and refresh the inventory again.
    * Note: We use this process because Ansible already has the logic to process the inventory template and give us a host count from it.
4. Next, the main.yml playbook calls on the cloud.yml playbook, which in turn calls the sequence Role in the `cloudera.exe` Collection to run all of the Public Cloud tasks for creating the necessary infrastructure for the deployment.
    * The defaults used by `cloudera.exe` for creating the Dynamic Inventory are [here in the role defaults](https://github.com/cloudera-labs/cloudera.exe/blob/main/roles/infrastructure/defaults/main.yml)
    * Note: that `globals.vm.count` from earlier is consumed here, along with naming, ssh, storage, etc. pieces.
    * Additional defaults are set from a [vars](https://github.com/cloudera-labs/cloudera.exe/blob/main/roles/infrastructure/vars/main.yml) file in this Role, where we hold pickers for things like storage, vm, OS type, etc. particular to that Infrastructure provider. All of which may be overridden.
    * Initialization of necessary information is done in initialize_setup_aws in the same Role, here we get the correct AMI to use for VM deployment, and then the host connectivity information Ansible needs for inventory population
    * Then in the main setup.yml file within this infrastructure Role, we run the infra provider-specific ‘compute’ task list
        * The compute task list first ensures the correct number of VMs is created with the given parameters. This task does a lot of work for us using the Infra provider’s own Ansible modules.
        * It then prepares the `$$infra__dynamic_inventory_host_entries$$` variable with a list of the necessary connection information for each new host in a format that Ansible understands, which is a combination of the generic connector information in `$$infra__dynamic_inventory_connectors$$` (ansible_user, ssh private key file) and the VM-specific information (private FQDN as node name, Public FQDN as ansible_host)

        NOTE: This connectivity information is very important for Ansible to build clusters correctly on AWS because AWS uses a reverse-proxy for DNS. Other infra providers would probably only require a much simpler set of connectivity information.
5. Next, the cloud.yml playbook will run a post-processing check if dynamic inventory VMs have been created by checking the `$$infra__dynamic_inventory_host_entries$$` variable, and if it and the template are present it will merge them into a `static_inventory.ini` artefact in the Definition directory using the cloudera-deploy/persist_dynamic_inventory tasklist. This artifact is later used by the cluster.yml playbook.

     NOTE:  This essentially picks up the connection entry for each host and replaces the template entry on a 1:1 basis, look at the example above to see how it works.

6. Back in the initial main.yml, we check if a static inventory artefact is presented. If the user created their own, or used the preceding dynamic inventory process to generate a static inventory artefact, this process is then triggered, thus allowing both options. This once again uses the ‘refesh_inventory’ tasklist to inject the static_inventory artefact (regardless of how it was created) into the Ansible Inventory ready for a Cluster deployment.

7. The cluster.yml playbook is then called, which will use whatever inventory is present to run the cluster deployment according to the rest of the configuration and tags presented. In this case it will pick up the static inventory artefact, produced by thi dynamic inventory process, and deploy the cluster.

## Definition Reference Guide

In this section we will break down all the available Definition options by section and explain their usages.

The general structure of each section will start with guidance on the minimally required information for that type of Deployment, followed by guidance on the various options.

For an introduction to the basics of Definition creation and the most common options, please see the [Deployments] Section.

### Profile

The default Cloudera-Deploy user profile is instantiated by the quickstart.sh script from a template file [profile.yml](https://github.com/cloudera-labs/cloudera-deploy/blob/main/profile.yml) in the repository, and written to `~/.config/cloudera-deploy/profiles/default`

The values in the profile are among the most common to be set for a Deployment, as such the User has several options where they can be set:

* Values in the `default` profile will be picked up at the lowest priority
* if the user provides the path to an alternative `profile.yml` via the extra-vars option `abs_profile`, that will be used instead of the default profile
* The User may set the values from the Profile in definition.yml, which will override the previous stated options
* The User may pass in the values as extra-vars at runtime, which overrides almost anything else.

IMPORTANT: Many Profile values use bash expansion of the user's home directory (`~/`)to refer to files, this allows the same path to work on your local machine AND in the Runner. Using absolute paths (`/home/user`) *in the Profile* is strongly advised against, but should be fine elsewhere in the Framework.

**Default Password**

```yaml
admin_password: Mysecretpword1!
```

IMPORTANT: You must provide an `admin_password` to the Framework in some manner, it is the only mandatory value which we do not set a default for.

This password will be propagated to your CDP Public Workload User, the admin user in CDP Base, and anywhere else a security value may be required such as in a database required by the Definition you supply.

We recommend you set individual passwords for separate services in *serious deployments* using the overrides available for each of these values in the Definition. For Demo and Development a common password is a lot easier.

_**Password Complexity**_

Recommended 12-16 chars, 1 Cap, 1 num, 1 special (avoid '#', '?' and '/' for some subsystems).

This satisfies password requirements for *most* subsystems.

What we mean by this is that the Framework automates many, many subsystems for you depending on what you put in the Definition, and each of those subsystems has different security standards. We strive to balance having security out-of-the-box with an easy to set default value.

**Namespace**

```yaml
name_prefix: cldr
```

Where names are procedurally generated by the Framework, rather than explicitly provided by the user, this will be prefixed. Sometimes it will be joined with other sources of uniqueness, such as the first two characters of the cloud provider, to generate a unique name for a given subsystem.

All deployment names should be prefixed to allow differentiation and accurate discovery or filtering. You are recommended to combine this with unique default Tags when there are a lot of Deployments sharing some Tenant to avoid clashes.

NOTE: Many subsystems require that deployments within a given Tenant have a unique name, as such you should avoid using the default name prefix `cldr` if multiple people may be using this automation Framework in your Deployments.

Ideally it should not exceed 6 chars, and must start with a letter. your personal Initials are an excellent option in most cases.

WARNING: Do not use punctuation in your `name_prefix`, particularly '-' and '_' can cause extremely weird behavior in some subsystems, particularly security subsystems.

WARNING: If you are deploying to Azure, we strongly recommend it be <= 4 chars, as Azure has quite short field character count limits in some cases, particularly storage.

It will default to the name_prefix `cldr` as set in [roles/cloudera_deploy/defaults](https://github.com/cloudera-labs/cloudera-deploy/blob/main/roles/cloudera_deploy/defaults/main.yml) if not provided by the User.

**Default Tags**

These tags are applied by default to all deployed infrastructure, particularly chargeable infrastructure on cloud providers.

You can also set specific Tags on specific deployments, such as a Datahub, in their sections of the Definition.

Some subsystems require certain tags to self-identify different functions, this is particularly common with certain cloud Kubernetes implementations. These will be automatically applied for you where appropriate.

You should use tags to identify your services to make it easy to track and remove them when no longer needed.

```yaml
tags:
  owner: dchaffelson@cloudera.com
  enddate: "01312022"
```

IMPORTANT: The two-space indent of the tag `key: value` pairs is critical for correct YAML structure

NOTE: For backwards compatibility reasons, we give the example date above in American or 'Freedom' format of MMDDYYYY. We recommend most people use ISO8601 or YYYYMMDD format.

**Cloud Infrastructure**

_**Cloud Provider**_

Specifies the Cloud Infrastructure provider, CDP presently supports GCP, AWS and Azure

```yaml
infra_type: aws  # gcp, azure
```

NOTE: Only necessary when using Public Cloud Infrastructure and when you do not specify an Ansible Inventory

_**Cloud Region**_

Specify the default region you prefer for your infrastructure provider

It must be valid for the selected infra_type

```yaml
infra_region: us-east-1
```
    
IMPORTANT: Generally it is not a problem if the region you set in Definition (or these defaults) differs from the region you have set in the CLI Config. The exception is where you wish to combine other tools with this Framework and they start looking in a different region. Be mindful of mixed configurations.

The automation service attempts to validate that the requested build will work in the given region, so some may be rejected as not all regions provide all cloud services.

NOTE: You are advised to build a reference deployment, including all services you intend to use, before investing into data and application development in a particular Cloud provider or region. You can immediately remove it once this validation is complete.

[Listing](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.RegionsAndAvailabilityZones.html) of AWS regions.

[Listing](https://azuretracks.com/2021/04/current-azure-region-names-reference) of Azure regions (Note: not microsoft.com).

Or you can run this in the runner:

```yaml
az account list-locations --query '[*].name'
```
    
[Listing](https://cloud.google.com/compute/docs/regions-zones) of GCP regions.

**Cloud Credentials**

Path to Google Cloud Credentials file, if using Google Cloud

Should be in your local profile

WARNING: We recommend they should not be located anywhere near a version controlled directory like git to avoid accidental inclusion!

If using Azure or AWS the credentials will be automatically collected from your local user profile; these credentials are required because Deployments require a GCP Service Account which is handled differently.

```yaml
gcloud_credential_file: '~/.config/gcloud/mycreds.json'
```

**SSH**

NOTE: Your SSH keys should be in your local profile (typically `~/.ssh`), or the Definition Path, or some other persistent directory available to the Runner. You are advised to be careful of not inadvertently committing your private SSH keys to version control.

NOTE: If you have set the necessary SSH Information into your Ansible Inventory for deploying CDP Base, you can leave these fields commented out of your profile. They are only really necessary for Public Cloud Deployments.

**Public Key File**

A Public Key file is required if using Azure or GCP as Cloud Infrastructure, or deploying Private Cloud

If not supplied, one will be generated using the supplied `name_prefix`, along with a matching Private Key file, and ignoring the `private_key_file` setting below. The default location is the usual ssh path in the User's Home Directory `~/.ssh`

```yaml
public_key_file: '~/.ssh/cldr.pub'
```

**Private key file**

Required if deploying Dynamic Inventory to set the Ansible Connection Parameters.

NOTE: Must be set if public_key_file is set as Ansible validates that your keys match as a convenience

```yaml
private_key_file: '~/.ssh/cldr.pem'
```

**Key Name**

Required for AWS Cloud Infrastructure

Defaults to the Namespace if not set

Must be set if public_key_file is set, even if not using AWS.

```yaml
public_key_id: cldr
```

**Cloudera License**

Path to your Cloudera License file, if you want to supply one.

Required if deploying a CDP Cluster for Private Cloud in a mode other than Trial, or where files are required to be downloaded from behind authentication on archive.cloudera.com

Should be in your local profile using bash expansion, or the definition directory.

```yaml
license_file: "~/.cdp/my_cloudera_license.txt
```

NOTE: Putting the license file in `~/.cdp/` is a convenient convention, but not required.

**Full Profile YAML**

```yaml
admin_password: Mysecretpword1!
name_prefix: cldr
tags:
  owner: dchaffelson@cloudera.com
  enddate: "01312022"
infra_type: aws
infra_region: us-east-1
gcloud_credential_file: '~/.config/gcloud/mycreds.json'
public_key_file: '~/.ssh/cldr.pub'
private_key_file: '~/.ssh/cldr.pem'
public_key_id: cldr
license_file: "~/.cdp/my_cloudera_license.txt"
```

### Globals

Globals is a structure to allow particular variables to be propagated across the entire Framework in a simple dictionary, and they are usually set by the top level playbook and then inherited by the Collections.

Globals are usually taken from some default or User set value, as such you are advised to not set the `globals` dictionary directly, as you may squash other variables the Framework expects to be there. If you want to change something you have found in `globals` during debugging, look to where it is set upstream in the Playbook or other locations.

Generally only add new values to globals when developing new features if they are needed in multiple Collections, such as `cloudera.exe` and `cloudera.cluster`.

**General**

The Globals in this section are explained elsewhere, as most come from the User Profile in Cloudera-Deploy, or are derived from its initialization process.

```yaml
globals:
  admin_password: MySuperSecret1!
  artifacts:
    create_deployment_details: yes
    directory: ~/.config/cloudera-deploy/artifacts
  cloudera_license_file: ~/.cdp/my_cloudera_license.txt
  create_utility_service: yes
  dynamic_inventory:
    vm:
      count: 6
      os: centos8
  gcloud_credential_file: ~/.config/gcp/credentials.json
  infra_type: aws
  name_prefix: cldr
  namespace_cdp: cldr-aw
  region: eu-west-1
  tags:
    mykey: myvalue
  utility_bucket_name: cldr-0123456789-uk-west-1
```

**Object Storage Name**

```yaml
globals:
  storage:
    name: us-west-1-default
```

This particular global is to allow the User to set a unique value to be used when constructing the name for a cloud object store, such as an S3 bucket or Azure storage account.

It defaults to the region + aws profile name for AWS, and just the name of the region for the other clouds. When combined with the namespace, this is usually sufficiently unique for most users.

This odd construction is necessary because, unlike most other cloud resources, these names much be globally unique to all accounts. This is not always the case, but as it usually is we opt for a solution which is less likely to surprise the user with an unexpected error.

**SSH**

Unfortunately the different cloud providers, and OS types that you may use, tend to have slightly different requirements when it comes to remote access via SSH. As such, you may observe there are several options for providing your SSH credentials to the Framework.

In most cases you are advised to supply the path to your `public_key_file` and `private_key_file` in your profile.

If you are using AWS, you should set the `public_key_id` to match.

You may supply the `public_key_text` directly for some Azure use cases, but we will look it up from the file if you supply `public_key_file` instead, and this latter option is preferred security practice.

```yaml
globals:
  ssh:
    key_path: ~/.ssh
    private_key_file: ~/.ssh/mykey.pem
    public_key_file: ~/.ssh/mykey.pub
    public_key_id: mykey
    public_key_text:
```

**Labels**

The global labels are the short strings used as identifiers when constructing larger labels for uniqueness. It allows us to procedurally generate the different names from standard pieces.

They are deliberately short in some cases due to restricted character counts on some fields.

```yaml
globals:
  labels:
    admin: admin
    app: app
    cml: cml
    cde: cde
    credential: cred
    cross_account: xaccount
    data: data
    datalake: dl
    datalake_admin: dladmin
    default: default
    env: env
    group: group
    idbroker: idbroker
    identity: identity
    internet_gateway: igw
    knox: know
    logs: logs
    policy: policy
    private: pvt
    public: pub
    ranger_audit: audit
    role: role
    service_network: svcnet
    storage: storage
    subnet: sbnt
    table: table
    user: user
    vpc: vpc
    vpce: vpce
```

### Infrastructure

**AWS**

```yaml
infra:
  aws:
    profile:
    region:
    arn_partition:
    vpc:
      az_count:
      internet_gateway:
        name:
        suffix:
      labels:
        public_route_table:
        private_route_table:
        public_route_table_suffix:
        private_route_table_suffix:
      existing:
        vpc_id:
        public_subnet_ids:
        private_subnet_ids:
    role:
      tags:
    policy:
      tags:
    storage:
      tags:
    private_endpoints:
```

**Azure**

```yaml
infra:
  azure:
    metagroup:
      name:
      suffix:
    netapp:
      account:
        name:
        suffix:
      pool:
        name:
        size:
        suffix:
        type:
      suffix:
      volume:
        name:
        size:
        suffix:
        type:
    region:
    sp_login_from_env:
    storage:
      class:
      name:
      type:
```

**Dynamic Inventory**

```yaml
infra:
  dynamic_inventory:
    storage:
      delete:
      size:
      type:
    tag:
    tag_key:
    tag_value:
    vm:
      suffix:
      type:
```

**GCP**

```yaml
infra:
  gcp:
    project:
    region:
    storage:
      path:
        data:
        logs:
```

**Security Groups**

```yaml
infra:
  security_group:
    default:
      name:
      suffix:
    knox:
      name:
      suffix:
    vpce:
      name:
      suffix:
```

**Storage**

```yaml
infra:
  storage:
    name:
    path:
      data:
      de:
      logs:
      ml:
      ranger_audit:
```

**Teardown**

```yaml
infra:
  teardown:
    delete_data:
    delete_mirror:
    delete_network:
    delete_ssh_key:
```

**VPC Networking**

```yaml
infra:
  vpc:
    cidr:
    extra_cidr:
    extra_ports:
    name:
    private_subnets:
    private_subnets_suffix:
    public_subnets:
    public_subnets_suffix:
    service_network:
      name:
      subnet:
    user_cidr:
    user_ports:
    tunneled_cidr:
```

### Environment

The Environment definition is one of the largest, because the Framework takes the position that it will only work with one Environment per run for CDP Public. This gives us the convenience of using it as encapsulating object for all configuration at the Platform level.

We'll break the `env` key down into sections around the direct keys and then the complex sub-keys, and then provide the full schema at the end of this section as usual.

**Top Level Environment Keys**

These keys are also at the top level under `env`, but do not have complex substructures and therefore do not need breaking out into a separate explanation

```yaml
env:
  name: cldr-aw-env
  suffix: env
  workload_analytics: yes  # no
  tunnel: yes  # no
  public_endpoint_access: yes  # no
```

_**Name**_

The name may be supplied here, or it will be procedurally generated by concatenating the namespace, cloud provider, and suffix. We include the cloud provider in the generated name as the user may wish to make a group of environments within a namespace but across multiple clouds, and the names must be distinct.

IMPORTANT: The first eight characters of the name of the Environment must be unique within the CDP Tenant, as they are used in generating various DNS entries for some cloud subsystems which you probably do not want to clash.

_**Suffix**_

The suffix used when generating the name procedurally, if the name is not directly provided.

Note that the suffix can be set here within the Environment definition, or at the globals level if you are composing your configs.

_**Workload Analytics**_

This simple true or false value enables workload analytics for this particular environment.

It defaults to false.

**L0, L1, and L2 network architectures**

One of the main consequential decisions when deploying a CDP Public Environment is the Cloud Networking Architecture you wish to deploy. While this has more complex implications in terms of the Network topology created, we abstract the typical cases into top level flags here.

We generally refer to network setups for CDP Public as falling into three categories.

If all nodes have public IPs and internet access, we call this L0 or Public - set tunnel & endpoints to false
If gateway nodes have public IPs, but other nodes do not, we call this L1 or semi-public - set tunnel & endpoints to true
If no nodes have public IPs, we call this L2 or private - set tunnel to true and endpoints to false

Note that L0 and L1 will work out of the box as the Framework will put the IP of the Ansible Controller on the network security allow-list by default. This is a practical consideration because the Controller _usually_ needs to connect to hosts to do more configuration work, and the User also _usually_ wants to access those machines from their workstation which is _usually_ running the Controller.

For an L2 configuration, the User will need to have some other arrangements to access to the private IPs within the deployed network. Perhaps a jumpbox, VPN, VPC Peering, or one of many such possibilities. These deployments are typical of Production cloud networking in enterprise customers and setup of them is outside the scope of this Framework.

_**Tunnel**_

Setting tunnel option to true enables CCM gateway which removes the need for the environment hosts to have a public IP address.

The default is false.

_**Public Endpoint Access**_

Setting public_endpoint_access to true enables public workload endpoint access gateway which lets users access workload from the internet.

The default is false.

Needed when tunneling is enabled, but you don't have the direct connectivity with the VPC via a VPN or similar.

**Environment AWS sub-structure**

As the actual definition of Infrastructure to be deployed on AWS lives under the `infra` tag, this section under `env` is primarily concerned with handling the naming and deployment of the necessary Policies and Roles.

While the Roles and Policies are created on AWS, and therefore you would reasonably think they should be part of the Infrastructure section, we moved them into the Platform section because the selection of Roles and Policies is closely tied to the shape of the CDP Public Environment and Datalake to be created, especially the cross-account access. It's not a perfect demarcation, but we have found this setup to be the least-worst of the available options.

_**Policy Naming**_

Every value in this section has a practical default.

You can override the suffix used in policy name generation with the `suffix` key, or directly set the literal name used for the policy object with the appropriate `name` subkey.

You may set tags to be applied to Policies here.

We plan to support directly supplying the policy documents from local files in the future, presently the Framework uses the official Cloudera policies directly from Cloudera's codebase.

```yaml
env:
  aws:
    policy:
      name:
        bucket_access: cldr-bucket-access-pol
        cross_account: cldr-xaccount-pol
        datalake_admin_s3: cldr-dl-admin-pol
        idbroker: cldr-idbroke-pol
        log: cldr-log-pol
        ranger_audit_s3: cldr-ranger-s3-audit-pol
      suffix: pol
      tags:
        pol_key: pol_val
```

_**Roles**_

Similar to Policies above, here you can set the `label` used when generating names for Roles. The label simply specifies the short descriptive string for that individual component type, and the 'suffix' is the string appended for this particular class of object.

If you do not set them here in the Environment configuration, they are usually set to one of the global suffix or label defaults. As such, you do not need to set any of these values in most cases.

You can also set the names for the Roles directly.

```yaml
env:
  aws:
    role:
      label:
        cross_account: xaccount
        datalake_admin: dladmin
        idbroker: idbroker
        log: log
        ranger_audit: audit
      name:
        cross_account: cldr-xaccount-rl
        datalake_admin: cldr-dladmin-rl
        idbroker: cldr-idbroker-rl
        log: cldr-log-rl
        ranger_audit:
```

_**Storage**_

Here you can simply override the default suffix used for naming policies and Roles for storage.

Not to be confused with the naming of buckets or storage accounts in the Infrastructure definition.

```yaml
env:
  aws:
    storage:
      suffix:
```

**Environment Azure sub-structure**

_**Azure Application Name**_

Explicitly set the name, or just the suffix to use when procedurally generating the name.

```yaml
env:
  azure:
    app:
      name: cldr-xaccount-app
      suffix: app
```

_**Azure Custom Policy for Cross Account Role**_

The Policy is stored in version control in Cloudera Labs and set to the minimum necessary policies for all CDP Public deployments to function.

You use your own policy document if you wish, but we recommend consultation with Cloudera Support first.

You may also override the suffix used when naming the Policy during creation

```yaml
env:
  azure:
    policy:
      suffix: policy
      url: https://raw.githubusercontent.com/cloudera-labs/snippets/main/policies/azure/cloudbreak_minimal_multiple_rgs_v1.json
```

_**Azure Roles**_

```yaml
env:
  azure:
    role:
      assignment:
        cross_account:
          contributor:
          role:
        datalake_admin:
          data:
            storageowner:
          logs:
            storageowner:
        idbroker:
          mgdidentop:
          vmcontributor:
        log:
          storagecontr:
        ranger_audit:
          storagecontr:
      label:
        data:
        datalake_admin:
        idbroker:
        identity:
        log:
        ranger_audit:
        xaccount:
      name:
        cross_account:
        datalake_admin:
        idbroker:
        log:
        ranger_audit:
      name_suffix:
        admin:
        assignment:
        contributor:
        operator:
        owner:
        user:
      suffix:
```

_**Azure Storage**_

```yaml
env:
  azure:
    storage:
      path:
        data:
        logs:
      suffix:
```

**Environment GCP sub-structure**

```yaml
env:
  gcp:
    bindings:
      cross_account:
      logs:
    role:
      label:
        cross_account:
        datalake_admin:
        idbroker:
        identity:
        log:
        ranger_audit:
      name:
        cross_account:
        datalake_admin:
        idbroker:
        identity:
        log:
        ranger_audit:
      suffix:
    storage:
      path:
        data:
        logs:
      suffix:
```

**Environment CDP sub-structure**

```yaml
env:
  cdp:
    admin_group:
      name:
      resource_roles:
      roles:
      suffix:
    control_plane:
      cidr:
      ports:
    credential:
      name:
      name_suffix:
      suffix:
    cross_account:
      account_id:
      external_id:
    group_suffix:
    user_group:
      name:
      resource_roles:
      roles:
      suffix:
```

**Environment Datalake sub-structure**

```yaml
env:
  datalake:
    name:
    suffix:
    user_sync:
    version:
    scale:
```

**Environment Teardown sub-structure**

```yaml
env:
  teardown:
    delete_admin_group:
    delete_credential:
    delete_cross_account:
    delete_policies:
    delete_roles:
    delete_user_group:
```

### Datahubs

When the `datahub` key is included at the top level, you are required to provide an array of `definitions` providing at enough information for the Framework to know which one you want deployed.

Datahub Deployment configurations are prepared in the `Prepare for CDP Datahub clusters` task within the [cloudera.exe.runtime.initialize_base](https://github.com/cloudera-labs/cloudera.exe/blob/main/roles/runtime/tasks/initialize_base.yml) Role within the Cloudera.exe Collection.

They are deployed in the `Request CDP Datahub deployments` Task within the `cloudera.exe.runtime.setup_base` Role, and leverage the [cloudera.cloud.datahub_clusters](https://cloudera-labs.github.io/cloudera.cloud/datahub_cluster.html) module.

**Minimum Datahub Definition**

The minimal definition to create a Datahub is to provide the name of a predefined Datahub Definition in the `definition` key within the array of definitions under the datahub key, e.g.

```yaml
datahub:
  definitions:
      definition: Streams Messaging Light Duty for AWS
```

NOTE: A listing of available Datahub Definitions can be found in the CDP UI by navigating to:
`Management Console > Environments > Your Environment > Cluster Definitions`

You may also use a Jinja Template via the `include` key, there is an example [here](https://github.com/cloudera-labs/cloudera.exe/blob/main/roles/runtime/templates/datahub_streams_messaging_light.j2)

You may also specify a CDP Datahub Template (sometimes called a Cluster Blueprint) using the `template` key, which will be paired with the `instance_groups` (either from defaults, or supplied by you) in order to produce a Definition to be deployed.

NOTE: Available Cluster Templates, including Custom Templates, can be found in the CDP UI by navigating to:
`Management Console > Shared Resources > Cluster Templates`

So, in summary, a Datahub Definition is a combination of a Template and Instance Groups. You may either use predefined Datahub Definitions from the CDP Control Plane, or pass in various methods of constructing one yourself.

**Naming your Datahubs**

If you do not supply a `name` key in your Datahub Definition, the Framework will attempt to create a name for you.

Datahub names must be unique within a Tenant, so you have several options:

* You are advised to supply your own unique name as the best option
* Or you can set the `suffix` key which will be concatenated with the namespace and cloud provider in order to generate a fairly unique but deterministic name, e.g. the suffix `dhub01` with the namespace `cldr` on AWS would produce `cldr-aw-dhub01`

**Instance Groups**

You may provide a detailed specification of your own instance groups, either on a per-Datahub basis in the `definitions` array, or by supplying a replacement `instance_group_base` key.

In most cases this is not recommended, as they predefined Datahub Definitions have best-practice configurations in place.

Of particular note for `instance_groups` are the look-up tables in [cloudera.exe.runtime.vars](https://github.com/cloudera-labs/cloudera.exe/blob/main/roles/runtime/vars/main.yml) which specify defaults for compute and storage used in various cases. You may wish to change the types and sizes of these values to suit your own scale.

NOTE: If you use a pre-defined Datahub Definition using the `definition` key, the instance_groups here are ignored, as the Definition has them baked in. You need to use a `template` or one of the other methods to override the instance_groups.

**Image Catalog**

When constructing a Datahub Template, the appropriate image for deployment is selected from the CDP Control Plane Image Catalog.

The default [Image Catalog](https://docs.cloudera.com/data-hub/cloud/create-cluster-aws/topics/mc-choose-image-catalog.html) for CDP is used, but you may supply the name and URL for a custom image catalog if you wish.

Preparing a custom image catalog is outside the scope of the Automation Framework.

**Full Datahub Definition**

This is the full specification with the most common default values included for your convenience.

In most cases you would not supply most of these values in your own Definition, and actually doing so is likely to cause a maintenance burden.

```yaml
datahub:
  definitions:
    - name: streams-messaging-dhub-01
      include: datahub_streams_messaging_light.j2
      template: Streams Messaging Light Duty: Apache Kafka
      definition: Streams Messaging Light Duty for AWS
      suffix: streams-dhub-01
      instance_groups:
        - nodeCount: 1
          instanceGroupName: master
          instanceGroupType: GATEWAY
          instanceType: "{{ run__datahub_compute[run__infra_type].std_gp }}"
          rootVolumeSize: 100
          recoveryMode: MANUAL
          recipeNames:
            - some_recipe_name
          attachedVolumeConfiguration:
          - volumeSize: 100
            volumeCount: 1
            volumeType: "{{ run__datahub_storage[run__infra_type].std }}"
      tags:
        key: value
  compute:
    aws:
      std_gp: 'm5.2xlarge'
      lrg_gp: 'm5.4xlarge'
      std_mem: 'r5.4xlarge'
      dsk_mem: 'r5d.4xlarge'
      std_gpu: "p2.8xlarge"
    azure:
      std_gp: 'Standard_D8_v3'
      lrg_gp: 'Standard_D16_v3'
      std_mem: 'Standard_D16_v3'
      dsk_mem: 'Standard_D8_v3'
      std_gpu: 'Standard_D8_v3'
    gcp:
      std_gp: 'e2-standard-8'
      lrg_gp: 'e2-standard-8'
      std_mem: 'e2-standard-8'
      dsk_mem: 'e2-standard-8'
      std_gpu: 'e2-standard-8'
  image_catalog:
    name: cdp-default
    url: https://cloudbreak-imagecatalog.s3.amazonaws.com/v3-prod-cb-image-catalog.json
  instance_group_base:
    nodeCount: 1
    instanceGroupName: master
    instanceGroupType: GATEWAY
    instanceType: "{{ run__datahub_compute[run__infra_type].std_gp }}"
    rootVolumeSize: 100
    recoveryMode: MANUAL
    recipeNames:
      - some_recipe_name
    attachedVolumeConfiguration:
      - volumeSize: 100
        volumeCount: 1
        volumeType: "{{ run__datahub_storage[run__infra_type].std }}"
  storage:
    aws:
      std: 'standard'
      fast: 'st1'
      eph: 'ephemeral'
    azure:
      std: 'StandardSSD_LRS'
      fast: 'StandardSSD_LRS'
      eph: 'StandardSSD_LRS'
    gcp:
      std: 'pd-standard'
      fast: 'pd-standard'
      eph: 'pd-standard'
```

### Data Engineering

```yaml
de:
  definitions:
    - name: cde-01
      instance_type: 'm5.2xlarge'
      minimum_instances: 1
      maximum_instances: 4
      minimum_spot_instances: 0
      maximum_spot_instances: 0
      enable_public_endpoint: yes
      enable_workload_analytics: yes
      initial_instances: 1
      initial_spot_instances: 0
      root_volume_size: 100
      chart_value_overrides:
        - chartName: dex-app
        - overrides: dexapp.api.gangScheduling.enabled:true
      skip_validation: yes
      tags:
        definition-tag: value
      use_ssd: yes
      virtual_clusters:
        - name: cloudera-deployed-vc-1
          cpu_requests: 32
          memory_requests: '128Gi'
          spark_version: 'SPARK2'
          acl_users: '*'
          runtime_spot_component: 'NONE'
          chart_value_overrides:
           - chartName: dex-app
           - overrides: pipelines.enabled:true
  suffix: de-svc
  tags:
    default_tag: value
  force_delete: no
  vc_suffix: de-vc
```

### Data Flow

```yaml
df:
  suffix:
  min_k8s_nodes:
  max_k8s_nodes:
  public_loadbalancer:
  loadbalancer_ip_ranges:
  kube_ip_ranges:
  cluster_subnets:
  loadbalancer_subnets:
  teardown:
    persist:
  force_delete:
  terminate_deployments:
```

### Data Warehouse

```yaml
dw:
  definitions:
  suffix:
```

### Machine Learning

```yaml
ml:
  definitions:
  k8s_request_base:
  suffix:
  tags:
  public_loadbalancer:
```

### Operational Database

```yaml
opdb:
  definitions:
  suffix:
```

### Data Management

```yaml
data:
  storage:
    # A list of lists of locations (read/[only|write]) defined in a policy and assigned to a Role
    - read_only: bool
      locations: []
      policy:
        name:
        suffix:
        delete: bool
      role:
        datalake_admin: bool
        name:
        suffix:
        delete: bool
  policy:
    suffix:
    aws:
      suffix:
      read_only:
        suffix:
        url:
      read_write:
        suffix:
        url:
  role:
    suffix:
    aws:
      suffix:
  teardown:
      delete_policies:
      delete_roles:
```
