---
id: ansible_developers
title: Goal-oriented Guides
sidebar_label: Developers

---

- TOC
{:toc}

=== Detailed Setup for Developers

This document is structured with suggested sequences of steps at the top, and the various processes you need underneath. Use the suggested sequences or create your own as appropriate.

Many of the Processes in here have been come to over years of trying different approaches, in the main our philosophy has been to adopt the least-worst option, minimising user surprise in the least number of the simplest possible steps, even if that results in a more complex implementation for us in code where few users ever tread.

We welcome all suggestions and discussions for improvement, but if you're going to tell us we're 'using Docker Wrong' then we'd greatly appreciate an explanation on how you'd improve it.

==== Developers setup on OSX

The Developer onboarding is aimed at those who want to modify the automation tooling itself to change behaviors.

.The following chapters of this documentation provide the necessary process:

. Prepare your xref:_install_homebrew_and_git_on_osx[OSX Deps]
. Follow the xref:_main_setup_process[Main Setup]
. (Optional) Setup xref:_setup_gpg_commit_signing[Commit Signing] for Contributions
. (Optional) Get setup to Develop the xref:_getting_started_with_developing_collections[Collections] themselves

==== Developer setup on Windows

Because Windows already needs to set up Linux compatibility to make containers work, we’re just going to leverage that across the board for simplicity.

IMPORTANT: You must use WSL2 or you are going to get some really weird errors.

We assume you have a recent version of 64bit Windows 10 with Administrator access and virtualization enabled, if you are restricted by your org from doing this then they should have a replacement process as this is pretty standard stuff for developers.

. Install xref:_install_windows_subsystem_for_linux_wsl2[WSL2]
. Handle xref:_handle_line_endings_on_windows[Line Endings] on Windows
. Follow the xref:_main_setup_process[Main Setup]
. (Optional) Setup xref:_setup_gpg_commit_signing[Commit Signing] for Contributions
. (Optional) Get setup to Develop the xref:_getting_started_with_developing_collections[Collections] themselves

=== Process Guides

==== Main Setup Process

This Setup Process is simply the https://github.com/cloudera-labs/cloudera-deploy/blob/main/readme.adoc#2-quickstart[Quickstart] from the Cloudera-Deploy Readme with more steps and explanations.

===== Prerequisites

NOTE: It is easiest to test your credentials within the Runner, as the dependencies are already installed for you

. Cloud Credentials (CDP Public) and/or Host Infrastructure (CDP Private)
** If you are deploying to a public cloud, you will need an active credential for that provider with rights to deploy resources for the Deployment you specify.
** Unless otherwise instructed your ‘default’ credential will be used for the cloud provider.
** You can check this with `aws sts get-caller-identity` and comparing outputs to the AWS UI, if using AWS
** For Azure, consider `az account list`, or if you need to refresh your credentials, `az login`.
** For GCP, check that your Service Account credential is being picked up in the Cloudera Deploy Profile, and then test with `gcloud auth list`

. CDP Credentials (CDP Public)
** If you are deploying CDP Public Cloud, you will need an active credential within a CDP Tenant.
** Unless otherwise instructed, your ‘default’ credential and tenant will be used.
** You can check this with `cdp iam get-user` and comparing outputs to the CDP UI

. Code Deps:
.. Docker and Git installed locally (quick option)(<<_setup_docker,See Setup Docker>>)
.. or - Ability to install Python and CLI dependencies on the machine of your choice (slow option)(See xref:_manual_ansible_controller_setup_on_centos7[Centos7 Script] for an example)

===== Get the Repo
. Open a terminal and switch to your preferred projects directory
[source,bash]
cd ~/Projects

. Clone or update the repo
[source,bash]
git clone https://github.com/cloudera-labs/cloudera-deploy.git && cd cloudera-deploy && chmod +x ./quickstart.sh

NOTE: If you already have the repo cloned, don’t forget to xref:_refresh_your_cloudera_deploy_local_repo[update it]

===== Get the Runner

. Have xref:_setup_docker[Docker] Installed and running
. (Optional) Set your provider to download a reduced image and save disk space
.. AWS
[source,bash]
export provider=aws

.. Azure
[source,bash]
export provider=azure

.. GCP
[source,bash]
export provider=gcp

.. Default is all of them, but it’s a ~2.3GB image download
. Run the quickstart script in the Repo you just cloned above
** Option A: Run the quickstart bare, it will mount the parent directory of the cloudera-deploy repo, which will be your code directory if you followed the directions above, e.g. `~/Projects`
[source,bash]
./quickstart.sh

** Option B: Include an explicit path on your local machine, it will be mounted in the /runner/project folder in the container. If your code or Definitions are not in this path you may not be able to access them in the container.
** Don’t worry if you get it wrong, just *stop the container* and rerun quickstart and it’ll make a new one with the new mount.
[source,bash]
./quickstart.sh ~/code

. This will drop you into a new shell in the container with all dependencies resolved for working with CDP and Cloud Infrastructure, you'll know it has worked because the shell prompt is Orange with `cldr <version> #>`
** It will automatically load your local machine user profile so you have access to your credentials, ssh keys, etc.
** If you run the quickstart script again, it’ll simply create another bash session on the container, providing useful parallelism
** If you stop the container, the next time you run quickstart it will be updated and recreated, so any changes within the container filesystem and not persisted back to your Project directory or mounted user profile will be lost
** As long as you run commands from within the /runner path, it will log your Ansible back to ~/.config/cloudera-deploy/log
. If you already have a CDP Credential in your local user profile, you can test it with
[source,bash]
cdp iam get-user

.. It will use the default CDP credential, or you can use a different profile by setting the CDP_PROFILE environment variable, or setting cdp_profile in your cloudera-deploy Definition files.
.. You should compare the UUID of your user returned by this command in the terminal with the UUID of your user reported in the User Profile in the CDP UI so you are certain that you are deploying to the expected Tenant
. Check you have a credential for your chosen Cloud Infrastructure Provider. The default is AWS, and again you can provide a specific profile or use your default. You can check it by running
[source,bash]
aws sts get-caller-identity

.. You should likewise compare the Account ID reported here with the Account ID in the AWS IAM UI to ensure you are targeting the expected Account. This is similar for other providers.

===== Prepare your Profiles to run a Deployment
NOTE: that you should execute any Ansible commands from /runner in the Runner, as it has all the defaults set for you and it may fail to find dependencies otherwise.

NOTE: If you have different settings for different deployments you can create additional profile files under the directory above to store the different configurations. To use an alternative cloudera-deploy profile, specify the `-e profile=<profile_name>` option when running the ansible-playbook command.

. Edit the default user Profile to personalise the Password, Namespace, SSH Keys, etc. for your Deployments. Note that this file is created the first time you run the quickstart.
[source,bash]
vi ~/.config/cloudera-deploy/profiles/default

. You will need CDP Credentials, and Credentials for your target Infrastructure of choice. Fortunately the Runner has most of these dependencies available to you
.. CDP with pre-issued Keys and optional profile
[source,bash]
cdp configure --profile default

.. AWS with pre-issued Keys
[source,bash]
aws configure --profile default

** AWS SSO requires awscliv2 which is not installed in the Runner by default
.. Azure via interactive login
[source,bash]
az login

.. Google Cloud via init
[source,bash]
gcloud init

===== Deployment Run Commands

Provided you have completed the prerequisites to set up a cloud provider credential, and CDP Public Cloud credential, and your Cloudera-Deploy Profile, the following command creates a default Sandbox with CDP Public & Private Cloud in your default CDP Tenant and Cloud Infra Provider with no further interaction from the user:
[source,bash]
ansible-playbook project/cloudera-deploy/main.yml -e "definition_path=examples/sandbox" -t run,default_cluster

NOTE: So that is three dependencies, and two commands, and you have a complete Hybrid Cloud Data Platform

The command is structured typically for an ansible-playbook.

* If you have used quickstart.sh to mount a local project directory with your definitions.yml and application.yml into the runner container (as explained in the steps in Get the Runner above), then you won’t have your own main.yml close to hand. You can instead use the default main.yml in /opt/cloudera-deploy/, and reference your project dir at /runner/project:
[source,bash]
ansible-playbook /opt/cloudera-deploy/main.yml -e "definition_path=/runner/project/<your definition path>" -t <your tags>

* Note that we pass in some ansible 'extra variables' using the -e flag. The only required variable points to the definition_path which contains the files that describe the deployment you want. You can provide many more extra vars, or even files directly on the command-line per usual Ansible options.

* Note that we pass in Ansible Tags with `-t`, in this case the tags instruct Cloudera-Deploy to build CDP Public Cloud to the 'Runtimes' level, and also deploy a 'default' or basic CDP Base Cluster on EC2 machines in the same VPC. There are many other tags that may be used to control behavior, and are explained elsewhere.

.Here are additional commands which will come in handy:

* Teardown and delete everything related to this definition:
[source,bash]
ansible-playbook project/cloudera-deploy/main.yml -e "definition_path=examples/sandbox" -t teardown

WARNING: This will `teardown` everything related to this definition and name_prefix, make sure that is actually what you want to be doing before running it.

* Just deploy a CDP Public Datalake:
[source,bash]
ansible-playbook project/cloudera-deploy/main.yml -e "definition_path=examples/sandbox" -t plat

NOTE: This uses the same definition, but then uses a different Ansible Tag to only deploy part of it, more explanation of the Definitions and Tags will follow elsewhere in the Architecture Docs.

* Just deploy CDP Private Cluster Trial on Public Cloud Infra:
[source,bash]
ansible-playbook project/cloudera-deploy/main.yml -e "definition_path=examples/sandbox" -t infra,default_cluster

NOTE: This leverages the dynamic inventory feature to make a simple cluster on EC2 instances on AWS without the user needing to learn how first, and is very handy for trials and platform testing

==== Refresh your Cloudera-Deploy local repo
If you have previously used Cloudera-Deploy but haven’t touched it in a while, here is a guide to refreshing your setup and getting results quickly

[source,bash]
cd cloudera-deploy
git fetch --all
git pull
docker stop $(docker ps -qa)
./quickstart.sh
cdp iam get-user
aws iam get-user

==== Manual Ansible Controller setup on Centos7

We provide an example script for initialising an Ansible Controller on a Centos7 box https://github.com/cloudera-labs/cloudera-deploy/blob/main/centos7-init.sh[here]

==== Install Homebrew and Git on OSX
.Install XCode command line tools
[source,bash]
xcode-select --install

.Install Homebrew
[source,bash]
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

.Install git (through Homebrew)
[source,bash]
brew install git

If you are going to use AWS SSO, you may also with to install awscliv2 on your local machine
[source, bash]
brew install awscli@2

==== Setup Docker

.Guide for Windows
Follow the instructions provided by https://docs.docker.com/docker-for-windows/install/[Docker]

NOTE: You want to be using WSL2, make sure Docker can see the underlying Ubuntu (or similar) for execution. You are advised to stick to the guide here, as we have found Windows throws some intractable filesystem and networking errors with creative linux-on-Windows setups for Docker and SSH.

.Guide for OSX
Follow the instructions provided by https://docs.docker.com/docker-for-mac/install/[Docker]

==== Install Windows Subsystem for Linux (WSL2)

.There’s a lot of guides on how to do this, here’s a summary:

. Enable Developer mode (on older versions of Win10)
.. Windows Settings > Update & Security
.. Select the Developer Mode radio button
.. You may not need to do this, don’t worry if it’s not there and the rest of the process works, as the latest releases don’t require you to do this to install Linux
.. If you did have to enable it, you may have to reboot (yay Windows)
. Enable Windows Subsystem for Linux v2
.. Control Panel > Programs & Features > Turn Windows Features on and off
.. Tick the box for ‘Windows Subsystem for Linux’
.. Make sure you either setup WSL2 from the start, or do the upgrade process. WSL1 has some strange behaviors
.. You’ll probably have to reboot. Yay Windows!
. Install Ubuntu 18 (other distros untested, including Ubuntu 20)
.. Try to do it from the Microsoft Store
... Open the store (search store in launch bar)
... Search for ‘Linux’ in the store
... Select and install Ubuntu
.. If you can’t do it from the store, try this in the cmd.exe prompt `lxrun /install`
.. It’ll ask you to set a username and password, keep it short and simple, doesn’t have to match your actual Windows user
. You may have to set this as the default bash environment as Docker for Windows likes to steal it away after updates
.. List the currently installed distros by opening cmd.exe
.. `wsl --list --all`
.. Set ubuntu as default if it is not
.. `wsl -s Ubuntu`

==== Handle line-endings on Windows
Windows defaults to a different line ending standard than Linux/OSX.

NOTE: You only need to follow this step if you plan on editing the code on Windows.

The Cloudera Labs repos standardise on linux line endings, the easiest way to handle this on Windows involves basically two steps.

. Set git to always not use Windows line endings, so it doesn’t rewrite your files when you checkout
.. `git config --global core.autocrlf false`
.. Set your IDE to use Linux line endings, if not for everything then at least the Cloudera Labs projects
.. In Pycharm, this is in File > Settings > Editor > Code Style > General > Line Separator: set to ‘Unix and macOS (\n)

==== Setup GPG commit signing

NOTE: You can skip this step if you are not planning on contributing code

.DCO:
Cloudera-Labs uses the Developer Certificate of Origin (DCO) approach to open source contribution, by signing your commits you are attesting that you are allowed to submit them by whoever owns your work (you, or your employer). We also require commit signing to validate the supply chain on our code.

.Background:
There is a good explanation https://nifi.apache.org/gpg.html[here], it also covers setup for machines with other OS.

This subguide assumes that you want to set up your development machine so that it automatically signs all your commits without bothering you too much. If you are just checking out the code to inspect it and not for contributing back to the community, then you can skip this step.
You may want to modify this process to ask you for your passphrase or manually sign commits each time at your preference.

.References:
https://nifi.apache.org/gpg.html  +
https://stackoverflow.com/a/46884134  +
https://superuser.com/a/954536  +
https://withblue.ink/2020/05/17/how-and-why-to-sign-git-commits.html  +

.Testing:
OSX Catalina 10.15.7 on MacBook Pro 2019

.Process:
. Update or install xref:_install_homebrew_and_git_on_osx[Homebrew]
. Install dependencies
[source,bash]
brew install gpg2 pinentry-mac && brew install --cask gpg-suite

** gpg-suite is not strictly necessary, but it makes it easier to integrate signing with IDEs like IntelliJ as it helps you manage the passphrase in your OSX keychain
** Pinentry-mac makes it easy to sign commits within your IDE, like Pycharm, without having to always commit via terminal
. Create a directory for gnupg to store details
[source,bash]
mkdir ~/.gnupg

. Put the following in `~/.gnupg/gpg-agent.conf`
[source,bash]
default-cache-ttl 600
max-cache-ttl 7200
default-cache-ttl-ssh 600
max-cache-ttl-ssh 7200
pinentry-program /usr/local/bin/pinentry-mac

. Enable it in your user profile such as ~/.bash_profile or ~/.zprofile
[source,bash]
export GPG_TTY=$(tty)
gpgconf --launch gpg-agent

. Set correct permissions on your gnupg user directory
[source,bash]
chown -R $(whoami) ~/.gnupg/
find ~/.gnupg -type f -exec chmod 600 {} \;
find ~/.gnupg -type d -exec chmod 700 {} \;

. Generate yourself a key
[source,bash]
gpg --full-gen-key

.. Key type 4
.. Keysize 4096
.. Expiration 1y or 2y or whatever
.. Your real name, or github username
.. Your real email address, or the one github recognises as yours in your Settings
.. A Passphrase - don’t forget it, and make sure it is strong
. Verify your key is created and stored
[source,bash]
gpg2 --list-secret-keys --keyid-format SHORT

** Copy your key ID, it’ll look something like `rsa4096/674CB45A`
** You want the second bit, `674CB45A`
. Test your key can be used
[source,bash]
echo "hello world" | gpg2 --clearsign

** You’ll have to enter your passphrase to sign it, then it’ll print the encrypted message
. You may want to add multiple email addresses to the key signing, such as your open source email and/or your employer email and/or your personal email
.. Open your key for editing
[source,bash]
gpg2 --edit-key <your ID here>

.. Then use the adduid command
[source, bash]
adduid

.. Enter the identity Name and Email as before
.. Then update the trust for the new identity
[source, bash]
uid 2
trust

** You probably want trust 5
.. Save to exit
[source, bash]
save

. Configure Github to recognise your signed commits
.. Set your git email for making commits
[source, bash]
git config --global user.email <your@email.com>

*** This email must be one of those in your GPG key set earlier
.. Export your public key for uploading to Github
[source, bash]
gpg2 --armor --export <your ID here>

.. Copy everything including the following lines into your paste buffer
[source, bash]
-----BEGIN PGP PUBLIC KEY BLOCK-----
…
-----END PGP PUBLIC KEY BLOCK-----

. Open github and go to your key settings
https://github.com/settings/keys
. Add new GPG key, paste in your PGP block from your buffer
. Configure git to autosign your commits
[source, bash]
git config --global gpg.program gpg
git config --global user.signingkey <your ID here>
git config --global commit.gpgSign true
git config --global tag.gpgSign true

. Put the following in ~/.gnupg/gpg.conf
[source, bash]
# Enable notty for IDE signing
no-tty
# Enable gpg to use the gpg-agent
use-agent

. Configure IntelliJ to sign your commits
.. This should be as simple as restarting your IDE once this process is complete
.. Then when you make a new commit you can tick the box to ‘Sign-off commit’ in the dialog box
. Configure vsCode to sign commits
.. Find the following flag in the config and enable it `git.enableCommitSigning`


==== Using the Ansible Runner Independent of Cloudera-Deploy

In order to minimise time spent on dependency management and troubleshooting issues arising from users on different systems, we provide a standardised container image.
The image is prepared in such a way that you can use it as a shell, a python environment, a container on Kubernetes, within other CICD Frameworks, Ansible Tower, or simply as an ansible-runner.

.Testing:

* OSX Catalina 10.15.7 on MacBook Pro 2019
* Windows 10.0.19042 on Intel gaming rig (Tasteful RGB Edition)

.Manual Process:
. To run this process on Windows you are expected to be within your xref:_install_windows_subsystem_for_linux_wsl2[WSL2] subsystem
. Clone the Cloudera Labs Ansible Runner implementation repo into your preferred local Projects directory
[source, bash]
git clone https://github.com/cloudera-labs/cldr-runner.git && cd cldr-runner

. Linux only: Mark the run_project.sh script and build.sh script as executable
[source, bash]
chmod +x ./run_project.sh
chmod +x ./build.sh

. Ensure xref:_setup_docker[Docker] is running on your host machine
. Copy the absolute path to the root of your code projects directory that contains the projects you want to execute within the container environment, e.g. `/Users/dchaffelson/Projects`
. Launch the runner targeting the project you want to execute by passing the absolute path as the argument to the run_project.sh script, e.g. `./run_project.sh /Users/dchaffelson/Projects`
** The script will build the container image from the latest release bits, this will take a few minutes the first time, the resulting image will be ~2GB
** You will then be dropped into a shell session in directory /runner in the container environment. Your Project will be mounted at /runner/project. You will have all the currently known dependencies for working with CDP pre-installed with conflicts resolved
** Note that the container must be stopped for a new project directory to be mounted to a new build, if there is already a container of the same name running you will just get a new shell session in it
. At this point you may wish to install additional dependencies to the container, particularly those which may be behind VPN or on your corporate VCS.
[source, bash]
ansible-galaxy install -r project/deps/ansible-deps.yml
pip install -r project/deps/python-deps.txt

NOTE: By default, the container is recreated if stopped, but it will not stop if you close your shell session as it is held open by a background tty. Try not to kill that.

==== Getting Started with Developing Collections

NOTE: You can skip this step if you only want to use the Collections to create your own playbooks.  +
This step is setting up the to Develop the Collections themselves.

This will guide you through setting up a directory structure convenient for developing and executing the Collections within the Runner, or other execution environments.

You only need to do this if you want to contribute directly to the Collections or Python clients underlying the interactions with Cloudera products - you do not need to go through this setup process if you simply wish to use cloudera-deploy with your own YAML Definitions, as the Collections and Clients should not need to be modified in those cases and are already pre-installed in the Runner.

.Why do it this way:
Ansible expects to find collections within a path ‘collections/ansible_collections’ on a series of predefined or default paths within your environment. By default, the Runner has this Path variable prepopulated in a helpful fashion to the pre-installed Collections, this process guides you through modifying that to point at your own versions which you have to maintain yourself.

For development purposes, creating this path in your favourite coding Projects directory, and then checking out the collections under it and renaming them to match the expected namespace may seem slightly arcane but it is the lowest-friction method for ongoing development we have found over many years of doing this.

.Process:
. Make the directory tree Ansible expects in the same parent code directory that cloudera-deploy is in, e.g.
[source, bash]
cd cloudera-deploy && mkdir -p ../ansible_dev/collections/ansible_collections/cloudera

** cloudera is the base namespace of our collection
** Your Projects directory should also have your Ansible Playbooks and other codebase in it, so that you can mount the root of it to the Runner and have access to all your codebase, e.g. `~/Projects/cloudera-deploy` should be where Cloudera-Deploy is located
. Fork each of the sub-collections and cdpy into your personal github, and replace <myAccount> with your actual github account below
. Checkout each of the sub-collections into this folder, e.g.:
[source, bash]
cd ~/Projects
git clone -b devel https://github.com/<myAccount>/cdpy.git cdpy
cd ansible_dev/collections/ansible_collections/cloudera
git clone -b devel https://github.com/<myAccount>/cloudera.exe.git exe
git clone -b devel https://github.com/<myAccount>/cloudera.cloud.git cloud
git clone -b devel https://github.com/<myAccount>/cloudera.cluster.git cluster
+
NOTE: The cloned directories above must be named "exe", "cloud" and "cluster", respectively. Ensure you specify the directory name as the last parameter in the command line, as shown above.
Each of the subcollections should be on the ‘devel’ branch so you can PR them back them with your changes
+
. Your Code Project directory should now look something like this:
[source,bash]
/Projects/ansible_dev/collections/ansible_collections/cloudera
/Projects/ansible_dev/collections/ansible_collections/cloudera/exe
/Projects/ansible_dev/collections/ansible_collections/cloudera/cloud
/Projects/ansible_dev/collections/ansible_collections/cloudera/cluster
/Projects/cdpy
/Projects/cloudera-deploy
. Before you invoke quickstart.sh, set the environment variable below to tell Ansible where to find your code inside your execution environment once it is mounted in the Container at /runner/project:
[source, bash]
export CLDR_COLLECTION_PATH="ansible_dev/collections"
export CLDR_PYTHON_PATH=/runner/project/cdpy/src
+
NOTE: You might want to set this in your bash or zsh profile on your local machine so it is persistent

. Then, when you run quickstart.sh in cloudera-deploy, it will pick up this extra Collection location and add cdpy to the PYTHONPATH, and use these instead of the release versions basked into the Container

. You can confirm this is working by running this inside the Runner
[source, bash]
ansible-galaxy collection list

It should look something like:
[source, bash]
----
# /runner/project/ansible_dev/collections/ansible_collections
Collection       Version
---------------- -------
cloudera.cloud   0.1.0
cloudera.cluster 2.0.0
cloudera.exe     0.0.1
Cloudera.runtime 0.0.1

# /home/runner/.ansible/collections/ansible_collections
Collection           Version
-------------------- -------
amazon.aws           1.4.0
ansible.netcommon    1.5.0
ansible.posix        1.1.1
azure.azcollection   1.4.0
community.aws        1.4.0
community.crypto     1.4.0
community.general    2.1.1
community.mysql      1.2.0
community.postgresql 1.1.1
google.cloud         1.0.2
netapp.azure         21.3.0
----

If you see duplication of collections because you are using the runner AND mounting your own versions, you probably have not activated the CLDR_COLLECTION_PATH variable correctly, and thus quickstart.sh is not picking it up.

As another test, you should also be able to invoke python inside the container and use cdpy
[source,python]
from cdpy.cdpy import Cdpy
c = Cdpy()
c.iam.get_user()

To test that cdpy is present and you can access your account as on cmdline

You may now edit the collections or cdpy and have the changes immediately available for use within the Runner, which is an awful lot easier than having to compile and crossload them after every change.

==== Install Dependencies without using Runner
We prefer that you use the Runner, because it sets many defaults to avoid common issues and thus save you and us a lot of issue reproduction time. However, we understand that there are many situations where it may not be appropriate, such as air-gapped environments, or when you want to run the install locally on the hardware and not have a separate ansible controller.

. Create a new virtualenv, or activate an existing one, we do not recommend installing dependencies in system python on most OS.
. Install dependencies for your hosting infrastructure version following the pathway laid out in the https://github.com/cloudera-labs/cldr-runner/blob/main/Dockerfile[Dockerfile] in ansible-runner
Install any additional dependencies you may have

NOTE: THe Dockerfile resolves combined dependencies for all our Hybrid Cloud Deployments, you probably only need a subset for your environment.

==== Developing within the Runner
While we recommend using the Runner as your execution environment when doing development, actually developing directly against the Runner using something like Visual Studio may not be a great idea due to the file system latency commonly encountered.

Generally when the maintainers work with this system we are editing the files directly on our host system using our IDE, and those files are RW mounted into the container for execution via the `/runner/project` mechanism, which then does not noticeably incur any performance degradation.

==== Working with AWS SSO

.Why:
Traditionally AWS users would use static keys, but more recently using temporary credentials via SSO is more commonplace.


The upside of AWS SSO is better credential management, the downside being additional complexity, reliance on the fairly awful AWSCLIv2, and a lack of OOTB automation integration.


.Setup AWS SSO:
. Install awscli v2 on your local machine
[source,bash]
brew install awscli

. Check you have version 2 (currently 2.5.4)
[source,bash]
aws --version

. Login to AWS SSO via Okta (or however), examine the list of accounts and select the one you want to set up, copy the name to use as your AWS Profile name
. Setup and login to AWS SSO for your selected account
[source,bash]
aws configure sso --profile <selected name here>

.. Enter your Start URL, e.g. `https://<app-name>.awsapps.com/start`
.. SSO Region, e.g. `us-east-1`
.. It’ll launch your browser (which is why we do it on your local machine)
.. Complete the login process to authenticate the session
.. Select which AWS account you wish to set up
.. Set your default region for this profile, e.g. `us-east-1`
.. Set your default output for this profile, e.g. `json`
. Now whenever you run a deployment you must set the profile name to match the one you have setup, e.g. in definition.yml
[source,yaml]
aws_profile: my-aws-profile

NOTE: You will likely have to re-login to SSO before each run

==== Working with named credential profiles

For CDP, AWS, and other credential-based access controls it is common to have multiple accounts that may be used from time to time. As such, it is useful to be able to easily, safely, and precisely switch between these accounts.

This is typically achieved through use of multiple named profiles, a good introductory guide to this is https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html[here] for AWS.

To set up a named profile for CDP, follow https://docs.cloudera.com/cdp/latest/cli/topics/mc-cli-generating-an-api-access-key.html[this] guide to create an API access key, and then https://docs.cloudera.com/cdp/latest/cli/topics/mc-configuring-cdp-client-with-the-api-access-key.html[this] guide to configure your local credentials by simply adding the `--profile <profile name>` flag to the commands

Credentials are typically stored in a dot-prefix directory in the user profile on Linux, and in the user home directory on Windows, e.g. `~/.aws/credentials`, `~/.cdp/credentials` etc. They may be directly edited in these files if you prefer instead of using the `configure` CLI commands.

By default, Cloudera-Deploy will use the `default` profiles for each account touched, you can set the Definition yaml keys `cdp_profile: <profile name>` and/or `aws_profile: <profile name>` to use a specific named profile instead.

.Things to know:

* There should always be a `default` credential which points to a 'safe' account for when you deliberately or accidentally run commands without specifying a profile. This is generally a dev account, or even an expired credential, to ensure you don't accidentally delete prod.
* It is typical with CDP Public Cloud to pair a CDP Tenant with specific Cloud Infrastructure accounts to avoid confusion, therefore if you have a `dev` and `prod` CDP tenant with profiles of the same name, we recommend the paired cloud infrastructure accounts are also named `dev` and `prod`.

==== Using a Jumpbox

A common deployment scenario is to funnel all cluster access through a jump/bastion host.

In this case, there are three possibilities:

. To run the Ansible Runner from the jump host itself
. To deploy the dependencies within the boundary of the Jump Host
. To run the Ansible Runner locally and tunnel connections through the jump host.

In scenario 3, the following will be necessary to tunnel both SSH connection and HTTP calls through the jump host.

.HTTP
In the runner, edit `/runner/env/envvars` and add `http_proxy=<proxy>` where<proxy> is the name:port of your http proxy (e.g. a SOCKs proxy running on localhost).

Alternatively, edit quickstart.sh to pass this value through from your local machine if it is available.

.SSH
In your inventory file, under the `[deployment:vars]` group, add the following variable to set additional arguments on the SSH command used by Ansible.
[source,bash]
ansible_ssh_common_args='-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ProxyCommand="ssh -W %h:%p -q <jump host>"'

Optionally, in your SSH config file (e.g. ~/.ssh/config) you can configure an alias with predefined parameters for the jump host. This makes it easier to manage between different deployments and makes the argument string easier to read.

[source,bash]
Host myJump
    IdentityFile ~/.ssh/myKey
    StrictHostKeyChecking = no
    User myUser
    HostName jump.host.name

With this SSH config the proxy string would look like this:
[source,bash]
ansible_ssh_common_args='-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ProxyCommand="ssh -W %h:%p -q myJump"'
