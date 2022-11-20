---
id: ansible_cdp
title: CDP Private Cloud
sidebar_label: CDP Private Cloud
---

- TOC
{:toc}

---

## CDP Private Cloud Deployments

CDP Base represents deployment of 'Traditional' clusters via Cloudera Manager, which you might think of historically as hadoop or big data clusters. In practice, there is an Ansible Collection called `cloudera.cluster` which is excellent at configuring Cloudera Manager across various versions of Linux regardless of whether that OS is running in Containers, Virtual Machines, Cloud Infra like EC2, or old school baremetal nodes.

Cloudera.cluster is wrapped within Cloudera-Deploy, so again in practice you are most likely to simply use Cloudera-Deploy as the entrypoint to drive `cloudera.cluster` to create the cluster you want via the Definition mechanism explained in these docs.

The differences are that Cloudera.cluster has been built over many generations to support Cloudera Manager versions 5 to 7, and supports many legacy options and behaviors for user convenience. The result of this is that, while simple deployments are as simple as the more modern CDP Public Cloud, complex deployments will require a greater familiarity with Ansible and Cloudera Manager templates.

## Ansible Controller options

We still strongly recommend that you use the cloudera-runner Docker Container shipped with Cloudera-Deploy as your Ansible controller for CDP Base deployments, because the consistency and logging is going to save you time and effort.

However, it is not uncommon to use the target Cloudera Manager node as the Controller. It is also not uncommon to need to deploy to air-gapped environments or environments where Docker is not permitted.

As such, there is a https://github.com/cloudera-labs/cloudera-deploy/blob/main/centos7-init.sh[script] for deploying the dependencies for Cloudera-Deploy on centos7 as an example in the repo for Power Users in these scenarios. Note that the script covers dependencies for all Cloudera Products on all current Clouds, so you may wish to exclude parts irrelevant to your scenario.

NOTE: It is worth noting that the `admin_password` field from your Profile, as outlined in the Quickstart, will be used as the Cloudera Manager admin password during `cloudera.cluster` deployments.

## Inventories
An inventory file can be supplied using the traditional -i flag at runtime or by including either
inventory_static.ini or inventory_template.ini in the definitions path.

.inventory_static.ini
An Ansible inventory covering the hosts in-scope for the deployment. An inventory with this name will be picked up automatically if present without using the `-i` to specify the inventory file at runtime.

Common inventory group names are:

* cloudera_manager
* cluster_master_nodes
* cluster_worker_nodes
* cluster_edge_nodes

Each host in these groups should be assigned one of the host templates that is defined in cluster.yml. These can be applied per host by setting `host_template=TemplateName` after the hostname, or per group by setting `host_template=TemplateName` in the group vars. See https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#adding-variables-to-inventory[here] for more information on Ansible inventory vars.

Nodes that should be added to the CDP Cluster as defined in the cluster.yml must be added as children to the `cluster` group. For example, if all data nodes are present under the `cluster_worker_nodes` group, then this group should be listed under `cluster:children`. See https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#inheriting-variable-values-group-variables-for-groups-of-groups[here] for an example of setting group children.

All nodes/groups should be added as children of the `deployment` group `[deployment:children]`. This includes the `cluster` group, as well as all non-CDP Cluster groups, for example, Cloudera Manager and supporting infrastructure nodes, perhaps running HAProxy or an RDBMS, that are deployed with the cluster but are not running CDP Roles. This allows for deployment-wide variables to be set, such as the common ssh key that can be used to access all nodes.

WARNING: Do not use the `[all:vars]` group if you are using an independent Ansible Controller as Plays will also be applied to it via the `localhost` mechanic. This includes from your Laptop or the Ansible-Runner supplied in Cloudera-Deploy.

## Process Overview

In this section we will walk through a typical deployment, commenting on options at various points

### Prepare a Definition Path

You are advised to create a working directory for your definition in your preferred Projects directory.

If you have checked out Cloudera-Deploy from Github following the Quickstart, then you could make a copy of the folder `cloudera-deploy/examples/sandbox` into the same base code Projects directory, and use it as a starting point.

You might then have:
```yaml
~/Projects/cloudera-deploy  # contains quickstart.sh, main.yml
~/Projects/my-definition  # copied from ~/Projects/cloudera-deploy/examples/sandbox
```

### Infrastructure and Inventory
You are advised to prepare your nodes in advance by whatever mechanism you prefer, and then include them in an Ansible Inventory matching the shape of the Cluster you want to deploy.

You may use Cloudera-Deploy to automatically provision EC2 instances based on the inventory template, but this is intended for test and training purposes. An example inventory template is included in the https://github.com/cloudera-labs/cloudera-deploy/blob/main/examples/sandbox/inventory_template.ini[Cloudera-Deploy Sandbox Example]. There is also a copy of the `static_inventory.ini` file in this directory.

If you want to use a static inventory, you may use any of the typical Ansible inventory approaches such as `-i` on the cmdline, the Inventory path, or including a file `inventory_static.ini` in your Definition Path which Cloudera-Deploy will automatically pick up.

You may wish to include additional variables in the Inventory, most typically the ssh user and private key. These should probably be in the `[deployment:vars]` section.

```yaml
[deployment:vars]
ansible_ssh_private_key_file=~/.ssh/root_key
ansible_user=root
```

NOTE: When you run quickstart.sh to launch the Runner Container, it will mount your local user `~/.ssh` directory into the container, so there is no need to copy keys around provided you use bash expansion for the user home directory.

### Cloudera-Deploy and the Runner

If you have not already done so, you should follow the instructions to setup your OS dependencies and get a copy of Cloudera-Deploy. New users are encouraged to use the https://github.com/cloudera-labs/cloudera-deploy/blob/main/readme.adoc[Quickstart], while Power Users who want to immediately understand the inner workings may wish to follow selected parts of the <<developers.adoc#_detailed_setup_for_developers,Developers>> guide.

Regardless of which method you select, complete the setup process until you are in the orange coloured interactive shell. Ensure that /runner/project within the shell has mounted your Projects directory that contains the Definition working directory you created in step one.

IMPORTANT: At this point you should ensure that the directory listing in `/runner/project` within the Runner matches your projects directory on your local machine, we earlier gave the example of `~/Projects` for this.

When using the Ansible Runner, you should be aware that most changes to the filesystem within the *container* are not persistent - that is, when you kill the container, most changes will be lost. When we are working on files that we want to persist, such as cluster templates, we can keep these on the local filesystem outside of the container, and mount the local filesystem directory inside the container, which is the purpose of the /runner/project mountpoint. This means that we can read and write to those files both inside and outside of the container and you can safely terminate your container without losing them.

Apart from the working dir, the container will mount specific directories out of your local user profile such as ~/.ssh, ~/.config, ~/.aws etc. into the container to pass-through credentials and other requirements. You should assume that any changes outside of these profile directories and /runner/project are not persisted.

NOTE: If we do not specify which directory to mount, the quickstart.sh script will mount the parent directory that the current session is in. i.e. if you run quickstart.sh from the path `~/Projects/cloudera-deploy`, then `~/Projects` in the host filesystem will be mounted to `/runner/projects` in the Container.

NOTE: If you did not stop the container previously, quickstart.sh will ignore any arguments passed and simply give you a new terminal session in the existing container. Therefore, if you want to change the mounted directory you must stop and restart the container with the new path.

## Executing Playbooks

To run the playbook, we use the `ansible-playbook` command. The entry point to the playbook is in main.yml in cloudera-deploy. With our mount point in /runner/project, the full path is `/runner/project/cloudera-deploy/main.yml`. This is passed as the first argument to ansible-playbook.

We also need to provide some additional arguments. We do this with the -e flag. The first argument to pass is the definition_path. This is the path to the directory that contains our definition, inventory, etc.

NOTE: It is usually a good idea to use Cloudera-Deploy to verify your infrastructure before committing to the full deployment, by using the same main playbook and adding the ‘verify’ tag, as below. This is particularly handy for real-world deployments for a quick sanity check.

An example command is shown below. Notice that we do not need to specifically provide an inventory list. The playbook will look in the definition_path for an inventory file, which is included in the cloudera-deploy-definitions examples. Of course, you can provide an inventory file using the -i if you want.

NOTE: Commands like `ansible-playbook` should be run from the `/runner` directory in the Ansible-Runner to pick up the defaults and other useful configurations

```yaml
ansible-playbook /runner/project/cloudera-deploy/main.yml \
 -e "definition_path=/runner/project/my-definition/" \
 -t verify
 -v
```

Following this will be a lot of output into the terminal which tracks the stages of our deployment.
```yaml
-----
PLAY [Init Cloudera-Deploy Run] ******************************************************************************************************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************************************************************************************************
Thursday 13 May 2021  18:54:13 +0000 (0:00:00.014)   	0:00:00.014 **********
ok: [localhost]
-----
```

To run the actual full deployment against your inventory, the most common tag to use is `full_cluster`. The complete listing of all tags can be found by reviewing the Plays in https://github.com/cloudera-labs/cloudera-deploy/blob/main/cluster.yml[cluster.yml] in cloudera-deploy.

.The command would look something like this, with verbosity at level 2

```yaml
ansible-playbook /runner/project/cloudera-deploy/main.yml \
 -e "definition_path=/runner/project/my-definition/" \
 -t full_cluster
 -vv
```

Expect the deployment to take 30 to 90 minutes or longer, assuming you encounter no errors.

If you do run into issues, most runs are idempotent so you can re-run with increased verbosity of the terminal output by adding the ‘-v’ flag to the ansible-playbook command. You can scale the verbosity by adding more v’s up to ‘-vvvvv’ for maximum verbosity.

There is nothing you need to do until the playbook completes, however it can be useful to have a scroll through the output and get a feel for what it is doing.

Eventually, you will get to some output that looks like the following. This indicates that Cloudera Manager is being installed, and then a check runs to wait for the server to start responding. When you get past this step, you’ll be able to access the CM UI in your browser.

It will be installed on the host that was under the cloudera_manager title in your inventory, and on port 7180 for HTTP and 7183 for HTTPS. The username is typically 'admin', and the password will be the admin_password you set in your Profile or Definition.

```yaml
-----
TASK [cloudera.cluster.server : Install Cloudera Manager Server] *********************************************************************************************************************************************************************************************
Thursday 13 May 2021  19:41:50 +0000 (0:00:06.555)   	0:47:36.812 **********
….
RUNNING HANDLER [cloudera.cluster.common : wait cloudera-scm-server] *****************************************************************************************************************************************************************************************
Thursday 13 May 2021  19:42:45 +0000 (0:00:09.338)   	0:48:31.682 **********
-----
```

The next important step to watch out for comes right at the end of the playbook. This is the Import Cluster Template step. In this step, the playbook is using the CM API to insert our cluster template, which allows CM to handle the complex logic of deploying the software, pushing out configurations and completing initializations and first runs. During this step, you will not see much useful output in the terminal.

Instead, you should go inside the CM web UI and go to the ‘Running Commands’ page, where you will be able to drill down into the ‘Import Cluster Template’ command and watch the individual steps that CM performs. This is the best place to debug any errors that you might encounter during the Import Cluster Template step.

```yaml
-----
TASK [cloudera.cluster.cluster : Import cluster template]
******************************************************
-----
```

NOTE: Deploying parcels can take some time if downloading directly from Cloudera Repos over slow or long-distance connections. Consider using the local repo options if doing multiple builds.

After the Template is imported, the First Run is completed, and then a cluster Restart command will run.

In the terminal session, our playbook has now completed and we will see the results at the end of the output. We should see a success message and a quick recap of the steps it took.


```yaml
-----
TASK [Deployment results] ***************************************************************************************************************
Thursday 13 May 2021  20:59:45 +0000 (0:00:00.287)   	2:05:31.793 **********
ok: [localhost] => {
	"msg": "Success!"
}

PLAY RECAP ***************************************************************************************************************
ccycloud-1.cddemo.root.hwx.site : ok=162  changed=49   unreachable=0	failed=0	skipped=151  rescued=0	ignored=0
ccycloud-2.cddemo.root.hwx.site : ok=71   changed=23   unreachable=0	failed=0	skipped=65   rescued=0	ignored=0
ccycloud-3.cddemo.root.hwx.site : ok=71   changed=23   unreachable=0	failed=0	skipped=65   rescued=0	ignored=0
ccycloud-4.cddemo.root.hwx.site : ok=71   changed=23   unreachable=0	failed=0	skipped=65   rescued=0	ignored=0
localhost              	: ok=173  changed=11   unreachable=0	failed=0	skipped=149  rescued=0	ignored=1

Thursday 13 May 2021  20:59:45 +0000 (0:00:00.064)   	2:05:31.857 **********
===============================================================================
cloudera.cluster.cluster : Import cluster template --------------------------------- 4132.25s
cloudera.cluster.daemons : Install Cloudera Manager daemons package ---------------- 1415.92s
cloudera.cluster.user_accounts : Create local user accounts ------------------------ 294.11s
cloudera.cluster.user_accounts : Set home directory permissions -------------------- 254.82s
cloudera.cluster.common : wait cloudera-scm-server --------------------------------- 99.33s
cloudera.cluster.agent : Install Cloudera Manager agent packages ------------------- 64.32s
cloudera.cluster.os : Populate service facts --------------------------------------- 60.19s
cloudera.cluster.jdk : Install JDK ------------------------------------------------- 50.16s
cloudera.cluster.krb5_server : Install KRB5 server --------------------------------- 39.23s
geerlingguy.postgresql : Ensure PostgreSQL packages are installed. ----------------- 38.81s
cloudera.cluster.cluster : Restart Cloudera Management Service --------------------- 35.66s
cloudera.cluster.mgmt : Start Cloudera Management Service -------------------------- 34.83s
cloudera.cluster.krb5_client : Install KRB5 client libraries ----------------------- 34.25s
cloudera.cluster.kerberos : Import KDC admin credentials --------------------------- 25.34s
Gather facts from connected inventory ---------------------------------------------- 20.92s
cloudera.cluster.krb5_server : Start Kerberos KDC ---------------------------------- 19.53s
cloudera.cluster.deployment/repometa : Download parcel manifest information -------- 19.00s
cloudera.cluster.os : Install rngd ------------------------------------------------- 18.34s
cloudera.cluster.rdbms : Copy SQL to change template to UTF-8 ---------------------- 16.64s
Gathering Facts -------------------------------------------------------------------- 16.31s
-----
```

From this, we can see that the build took 2:05:31.793 (2 hours 5 minutes) in total, around 1 hour of this was the Import Cluster Template which includes the parcel downloads. Pre-downloading and hosting a cluster-local parcel repository can speed this up dramatically.

