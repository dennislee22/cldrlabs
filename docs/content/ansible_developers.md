---
id: ansible_developers
title: Goal-oriented Guides
sidebar_label: Goal-oriented Guides

---

- TOC
{:toc}

## Detailed Setup for Developers

This document is structured with suggested sequences of steps at the top, and the various processes you need underneath. Use the suggested sequences or create your own as appropriate.

Many of the Processes in here have been come to over years of trying different approaches, in the main our philosophy has been to adopt the least-worst option, minimising user surprise in the least number of the simplest possible steps, even if that results in a more complex implementation for us in code where few users ever tread.

We welcome all suggestions and discussions for improvement, but if you're going to tell us we're 'using Docker Wrong' then we'd greatly appreciate an explanation on how you'd improve it.

### Developers setup on OSX

The Developer onboarding is aimed at those who want to modify the automation tooling itself to change behaviors.

_**The following chapters of this documentation provide the necessary process:**_

* Prepare your [OSX Deps](#install-homebrew-and-git-on-osx)
* Follow the [Main Setup](#main-setup-process)
* (Optional) Setup [Commit Signing](#setup-gpg-commit-signing) for Contributions
* (Optional) Get setup to Develop the [Collections](#getting-started-with-developing-collections) themselves

### Developer setup on Windows

Because Windows already needs to set up Linux compatibility to make containers work, we just going to leverage that across the board for simplicity.

IMPORTANT: You must use WSL2 or you are going to get some really weird errors.

We assume you have a recent version of 64bit Windows 10 with Administrator access and virtualization enabled, if you are restricted by your org from doing this then they should have a replacement process as this is pretty standard stuff for developers.

* Install [WSL2](#install-windows-subsystem-for-linux_wsl2)
* Handle [Line Endings](#handle-line_endings-on-windows) on Windows
* Follow the [Main Setup](#main-setup-process)
* (Optional) Setup [Commit Signing](#setup-gpg-commit-signing) for Contributions
* (Optional) Get setup to Develop the [Collections](#getting-started-with-developing-collections) themselves

## Process Guides

### Main Setup Process

This Setup Process is simply the [Quickstart](https://github.com/cloudera-labs/cloudera-deploy/blob/main/readme.adoc#2-quickstart) from the Cloudera-Deploy Readme with more steps and explanations.

**Prerequisites**

NOTE: It is easiest to test your credentials within the Runner, as the dependencies are already installed for you

1. Cloud Credentials (CDP Public) and/or Host Infrastructure (CDP Private)

	* If you are deploying to a public cloud, you will need an active credential for that provider with rights to deploy resources for the Deployment you specify.
	* Unless otherwise instructed your default credential will be used for the cloud provider.
	* You can check this with `aws sts get-caller-identity` and comparing outputs to the AWS UI, if using AWS
	* For Azure, consider `az account list`, or if you need to refresh your credentials, `az login`.
	* For GCP, check that your Service Account credential is being picked up in the Cloudera Deploy Profile, and then test with `gcloud auth list`

2. CDP Credentials (CDP Public)
	* If you are deploying CDP Public Cloud, you will need an active credential within a CDP Tenant.
	* Unless otherwise instructed, your default credential and tenant will be used.
	* You can check this with "cdp iam get-user" and comparing outputs to the CDP UI

3. Code Deps:
	* Docker and Git installed locally.
	* or - Ability to install Python and CLI dependencies on the machine of your choice (slow option)(See [Centos7 Script](#manual-ansible-controller-setup-on-centos7) for an example)

**Get the Repo**

1. Open a terminal and switch to your preferred projects directory

    ```yaml
    cd ~/Projects
    ```

2. Clone or update the repo

    ```yaml
    git clone https://github.com/cloudera-labs/cloudera-deploy.git && cd cloudera-deploy && chmod +x ./quickstart.sh
    ```

NOTE: If you already have the repo cloned, don't forget to update it.

**Get the Runner**

1. Have [Docker](#setup-docker) installed and running

2. (Optional) Set your provider to download a reduced image and save disk space

    * AWS

        ```yaml
        export provider=aws
        ```

    * Azure
        
        ```yaml
        export provider=azure
        ```

    * GCP
        
        ```yaml
        export provider=gcp
        ```

    * Default is all of them, but it's a ~2.3GB image download

3. Run the quickstart script in the Repo you just cloned above

    * Option A: Run the quickstart bare, it will mount the parent directory of the cloudera-deploy repo, which will be your code directory if you followed the directions above, e.g. `~/Projects`

    ```yaml
    ./quickstart.sh
    ```

    * Option B: Include an explicit path on your local machine, it will be mounted in the /runner/project folder in the container. If your code or Definitions are not in this path you may not be able to access them in the container.

    * Dont worry if you get it wrong, just stop the container and rerun quickstart and it will make a new one with the new mount.

    ```yaml
    ./quickstart.sh ~/code
    ```

4. This will drop you into a new shell in the container with all dependencies resolved for working with CDP and Cloud Infrastructure, you'll know it has worked because the shell prompt is Orange with `cldr <version> #>`

    * It will automatically load your local machine user profile so you have access to your credentials, ssh keys, etc.
    * If you run the quickstart script again, it will simply create another bash session on the container, providing useful parallelism
    * If you stop the container, the next time you run quickstart it will be updated and recreated, so any changes within the container filesystem and not persisted back to your Project directory or mounted user profile will be lost
    * As long as you run commands from within the /runner path, it will log your Ansible back to ~/.config/cloudera-deploy/log

5. If you already have a CDP Credential in your local user profile, you can test it with
    
    ```yaml
    cdp iam get-user
    ```

    * It will use the default CDP credential, or you can use a different profile by setting the CDP_PROFILE environment variable, or setting cdp_profile in your cloudera-deploy Definition files.
    * You should compare the UUID of your user returned by this command in the terminal with the UUID of your user reported in the User Profile in the CDP UI so you are certain that you are deploying to the expected Tenant


6. Check you have a credential for your chosen Cloud Infrastructure Provider. The default is AWS, and again you can provide a specific profile or use your default. You can check it by running

    ```yaml
    aws sts get-caller-identity
    ```

    * You should likewise compare the Account ID reported here with the Account ID in the AWS IAM UI to ensure you are targeting the expected Account. This is similar for other providers.

**Prepare your Profiles to run a Deployment**

NOTE: that you should execute any Ansible commands from /runner in the Runner, as it has all the defaults set for you and it may fail to find dependencies otherwise.

NOTE: If you have different settings for different deployments you can create additional profile files under the directory above to store the different configurations. To use an alternative cloudera-deploy profile, specify the `-e profile=<profile_name>` option when running the ansible-playbook command.

1. Edit the default user Profile to personalise the Password, Namespace, SSH Keys, etc. for your Deployments. Note that this file is created the first time you run the quickstart.

    ```yaml
    vi ~/.config/cloudera-deploy/profiles/default
    ```

2. You will need CDP Credentials, and Credentials for your target Infrastructure of choice. Fortunately the Runner has most of these dependencies available to you

    * CDP with pre-issued Keys and optional profile

    ```yaml
    cdp configure --profile default
    ```

    * AWS with pre-issued Keys

    ```yaml
    aws configure --profile default
    ```

    AWS SSO requires awscliv2 which is not installed in the Runner by default

    * Azure via interactive login

    ```yaml
    az login
    ```

    * Google Cloud via init
    ```yaml
    gcloud init
    ```

**Deployment Run Commands**

Provided you have completed the prerequisites to set up a cloud provider credential, and CDP Public Cloud credential, and your Cloudera-Deploy Profile, the following command creates a default Sandbox with CDP Public & Private Cloud in your default CDP Tenant and Cloud Infra Provider with no further interaction from the user:

```yaml
ansible-playbook project/cloudera-deploy/main.yml -e "definition_path=examples/sandbox" -t run,default_cluster
```

NOTE: So that is three dependencies, and two commands, and you have a complete Hybrid Cloud Data Platform

The command is structured typically for an ansible-playbook.

* If you have used quickstart.sh to mount a local project directory with your definitions.yml and application.yml into the runner container (as explained in the steps in Get the Runner above), then you won't have your own main.yml close to hand. You can instead use the default main.yml in /opt/cloudera-deploy/, and reference your project dir at /runner/project:

    ```yaml
    ansible-playbook /opt/cloudera-deploy/main.yml -e "definition_path=/runner/project/<your definition path>" -t <your tags>
    ```

* Note that we pass in some ansible 'extra variables' using the -e flag. The only required variable points to the definition_path which contains the files that describe the deployment you want. You can provide many more extra vars, or even files directly on the command-line per usual Ansible options.

* Note that we pass in Ansible Tags with `-t`, in this case the tags instruct Cloudera-Deploy to build CDP Public Cloud to the 'Runtimes' level, and also deploy a 'default' or basic CDP Base Cluster on EC2 machines in the same VPC. There are many other tags that may be used to control behavior, and are explained elsewhere.

_**Here are additional commands which will come in handy:**_

* Teardown and delete everything related to this definition:
    
    ```yaml
    ansible-playbook project/cloudera-deploy/main.yml -e "definition_path=examples/sandbox" -t teardown
    ```

WARNING: This will `teardown` everything related to this definition and name_prefix, make sure that is actually what you want to be doing before running it.

* Just deploy a CDP Public Datalake:
    
    ```yaml
    ansible-playbook project/cloudera-deploy/main.yml -e "definition_path=examples/sandbox" -t plat
    ```

NOTE: This uses the same definition, but then uses a different Ansible Tag to only deploy part of it, more explanation of the Definitions and Tags will follow elsewhere in the Architecture Docs.

* Just deploy CDP Private Cluster Trial on Public Cloud Infra:

    ```yaml
    ansible-playbook project/cloudera-deploy/main.yml -e "definition_path=examples/sandbox" -t infra,default_cluster
    ```

NOTE: This leverages the dynamic inventory feature to make a simple cluster on EC2 instances on AWS without the user needing to learn how first, and is very handy for trials and platform testing

### Refresh your Cloudera-Deploy local repo
If you have previously used Cloudera-Deploy but haven't touched it in a while, here is a guide to refreshing your setup and getting results quickly

```yaml
    cd cloudera-deploy
    git fetch --all
    git pull
    docker stop $(docker ps -qa)
    ./quickstart.sh
    cdp iam get-user
    aws iam get-user
```

### Manual Ansible Controller setup on Centos7

We provide an example script for initialising an Ansible Controller on a Centos7 box [here](https://github.com/cloudera-labs/cloudera-deploy/blob/main/centos7-init.sh)

### Install Homebrew and Git on OSX
Install XCode command line tools

```yaml
xcode-select --install
```

Install Homebrew

```yaml
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Install git (through Homebrew)

```yaml
brew install git
```

If you are going to use AWS SSO, you may also with to install awscliv2 on your local machine

```yaml
brew install awscli@2
```

### Setup Docker

**Guide for Windows**
Follow the instructions provided by https://docs.docker.com/docker-for-windows/install/[Docker]

NOTE: You want to be using WSL2, make sure Docker can see the underlying Ubuntu (or similar) for execution. You are advised to stick to the guide here, as we have found Windows throws some intractable filesystem and networking errors with creative linux-on-Windows setups for Docker and SSH.

**Guide for OSX**
Follow the instructions provided by https://docs.docker.com/docker-for-mac/install/[Docker]

### Install Windows Subsystem for Linux (WSL2)

_**There's a lot of guides on how to do this, here's the summary:**_

1. Enable Developer mode (on older versions of Win10)
    * Windows Settings > Update & Security
    * Select the Developer Mode radio button
    * You may not need to do this, don't worry if it's not there and the rest of the process works, as the latest releases don't require you to do this to install Linux
    * If you did have to enable it, you may have to reboot (yay Windows)

2. Enable Windows Subsystem for Linux v2
    * Control Panel > Programs & Features > Turn Windows Features on and off
    * Tick the box for 'Windows Subsystem for Linux'
    * Make sure you either setup WSL2 from the start, or do the upgrade process. WSL1 has some strange behaviors
    * You'll probably have to reboot. Yay Windows!

3. Install Ubuntu 18 (other distros untested, including Ubuntu 20)
    * Try to do it from the Microsoft Store
    ** Open the store (search store in launch bar)
    ** Search for Linux in the store
    ** Select and install Ubuntu
    * If you can't do it from the store, try this in the cmd.exe prompt `lxrun /install`
    * It'll ask you to set a username and password, keep it short and simple, doesn't have to match your actual Windows user

4. You may have to set this as the default bash environment as Docker for Windows likes to steal it away after updates

    * List the currently installed distros by opening cmd.exe
    * `wsl --list --all`    
    * Set ubuntu as default if it is not
    * `wsl -s Ubuntu`

### Handle line-endings on Windows
Windows defaults to a different line ending standard than Linux/OSX.

NOTE: You only need to follow this step if you plan on editing the code on Windows.

The Cloudera Labs repos standardise on linux line endings, the easiest way to handle this on Windows involves basically two steps.

1. Set git to always not use Windows line endings, so it doesn't rewrite your files when you checkout
    * `git config --global core.autocrlf false`
    * Set your IDE to use Linux line endings, if not for everything then at least the Cloudera Labs projects
    * In Pycharm, this is in File > Settings > Editor > Code Style > General > Line Separator: set to Unix and macOS (\n)

### Setup GPG commit signing

NOTE: You can skip this step if you are not planning on contributing code

_**DCO:**_

Cloudera-Labs uses the Developer Certificate of Origin (DCO) approach to open source contribution, by signing your commits you are attesting that you are allowed to submit them by whoever owns your work (you, or your employer). We also require commit signing to validate the supply chain on our code.

_**Background:**_

There is a good explanation [here](https://nifi.apache.org/gpg.html), it also covers setup for machines with other OS.

This subguide assumes that you want to set up your development machine so that it automatically signs all your commits without bothering you too much. If you are just checking out the code to inspect it and not for contributing back to the community, then you can skip this step.
You may want to modify this process to ask you for your passphrase or manually sign commits each time at your preference.

_**References:**_

* https://nifi.apache.org/gpg.html
* https://stackoverflow.com/a/46884134
* https://superuser.com/a/954536
* https://withblue.ink/2020/05/17/how-and-why-to-sign-git-commits.html

_**Testing:**_

OSX Catalina 10.15.7 on MacBook Pro 2019

_**Process:**_

1. Update or install [Homebrew](#install-homebrew-and-git-on-osx)

2. Install dependencies. gpg-suite is not strictly necessary, but it makes it easier to integrate signing with IDEs like IntelliJ as it helps you manage the passphrase in your OSX keychain. Pinentry-mac makes it easy to sign commits within your IDE, like Pycharm, without having to always commit via terminal

    ```yaml
    brew install gpg2 pinentry-mac && brew install --cask gpg-suite
    ```
    
3. Create a directory for gnupg to store details

    ```yaml
    mkdir ~/.gnupg
    ```
    
4. Put the following in `~/.gnupg/gpg-agent.conf`

    ```yaml
    default-cache-ttl 600
    max-cache-ttl 7200
    default-cache-ttl-ssh 600
    max-cache-ttl-ssh 7200
    pinentry-program /usr/local/bin/pinentry-mac
    ```
5. Enable it in your user profile such as ~/.bash_profile or ~/.zprofile

    ```yaml
    export GPG_TTY=$(tty)
    gpgconf --launch gpg-agent
    ```

6. Set correct permissions on your gnupg user directory

    ```yaml
    chown -R $(whoami) ~/.gnupg/
    find ~/.gnupg -type f -exec chmod 600 {} \;
    find ~/.gnupg -type d -exec chmod 700 {} \;
    ```

7. Generate yourself a key

    ```yaml
    gpg --full-gen-key
    ```

    * Key type 4
    * Keysize 4096
    * Expiration 1y or 2y or whatever
    * Your real name, or github username
    * Your real email address, or the one github recognises as yours in your Settings
    * A Passphrase - don't forget it, and make sure it is strong

8. Verify your key is created and stored

    ```yaml
    gpg2 --list-secret-keys --keyid-format SHORT
    ```

    Copy your key ID, it'll look something like `rsa4096/674CB45A`
    You want the second bit, `674CB45A`

9.  Test your key can be used

    ```yaml
    echo "hello world" | gpg2 --clearsign
    ```
    
    You'll have to enter your passphrase to sign it, then it'll print the encrypted message

10. You may want to add multiple email addresses to the key signing, such as your open source email and/or your employer email and/or your personal email
    
    * Open your key for editing

    ```yaml
    gpg2 --edit-key <your ID here>
    ```
    * Then use the adduid command
    ```yaml
    adduid
    ```

    * Enter the identity Name and Email as before
    
    * Then update the trust for the new identity
    ```yaml
    uid 2
    trust
    ```

    You probably want trust 5

    * Save to exit
    ```yaml
    save
    ```    

11. Configure Github to recognise your signed commits
    
    * Set your git email for making commits
    
    ```yaml
    git config --global user.email <your@email.com>
    ```

    This email must be one of those in your GPG key set earlier
 
    * Export your public key for uploading to Github
    
    ```yaml
    gpg2 --armor --export <your ID here>
    ```

    * Copy everything including the following lines into your paste buffer
    
    ```yaml
    -----BEGIN PGP PUBLIC KEY BLOCK-----
    ...
    -----END PGP PUBLIC KEY BLOCK-----
    ```

12. Open github and go to your key settings `https://github.com/settings/keys`

13. Add new GPG key, paste in your PGP block from your buffer

14. Configure git to autosign your commits

    ```yaml
    git config --global gpg.program gpg
    git config --global user.signingkey <your ID here>
    git config --global commit.gpgSign true
    git config --global tag.gpgSign true
    ```

15. Put the following in `~/.gnupg/gpg.conf`
    
    ```yaml
    # Enable notty for IDE signing
    no-tty
    # Enable gpg to use the gpg-agent
    use-agent
    ```

16. Configure IntelliJ to sign your commits
    * This should be as simple as restarting your IDE once this process is complete
    * Then when you make a new commit you can tick the box to `Sign-off commit` in the dialog box

17. Configure vsCode to sign commits
    * Find the following flag in the config and enable it `git.enableCommitSigning`

### Using the Ansible Runner Independent of Cloudera-Deploy

In order to minimise time spent on dependency management and troubleshooting issues arising from users on different systems, we provide a standardised container image.
The image is prepared in such a way that you can use it as a shell, a python environment, a container on Kubernetes, within other CICD Frameworks, Ansible Tower, or simply as an ansible-runner.

**Testing:**

    * OSX Catalina 10.15.7 on MacBook Pro 2019
    * Windows 10.0.19042 on Intel gaming rig (Tasteful RGB Edition)

**Manual Process:**

1. To run this process on Windows you are expected to be within your [WSL2](#install-windows-subsystem-for-linux-wsl2) subsystem

2. Clone the Cloudera Labs Ansible Runner implementation repo into your preferred local Projects directory
   
   ```yaml
   git clone https://github.com/cloudera-labs/cldr-runner.git && cd cldr-runner
   ```

3. Linux only: Mark the run_project.sh script and build.sh script as executable
    
    ```yaml
    chmod +x ./run_project.sh
    chmod +x ./build.sh
    ```

4. Ensure [Docker](#setup-docker) is running on your host machine

5. Copy the absolute path to the root of your code projects directory that contains the projects you want to execute within the container environment, e.g. `/Users/dchaffelson/Projects`

6. Launch the runner targeting the project you want to execute by passing the absolute path as the argument to the run_project.sh script, e.g. `./run_project.sh /Users/dchaffelson/Projects`

    * The script will build the container image from the latest release bits, this will take a few minutes the first time, the resulting image will be ~2GB
    
    * You will then be dropped into a shell session in directory /runner in the container environment. Your Project will be mounted at /runner/project. You will have all the currently known dependencies for working with CDP pre-installed with conflicts resolved
    
    * Note that the container must be stopped for a new project directory to be mounted to a new build, if there is already a container of the same name running you will just get a new shell session in it

7. At this point you may wish to install additional dependencies to the container, particularly those which may be behind VPN or on your corporate VCS.

    ```yaml
    ansible-galaxy install -r project/deps/ansible-deps.yml
    pip install -r project/deps/python-deps.txt
    ```

NOTE: By default, the container is recreated if stopped, but it will not stop if you close your shell session as it is held open by a background tty. Try not to kill that.

### Getting Started with Developing Collections

NOTE: You can skip this step if you only want to use the Collections to create your own playbooks.  +
This step is setting up the to Develop the Collections themselves.

This will guide you through setting up a directory structure convenient for developing and executing the Collections within the Runner, or other execution environments.

You only need to do this if you want to contribute directly to the Collections or Python clients underlying the interactions with Cloudera products - you do not need to go through this setup process if you simply wish to use cloudera-deploy with your own YAML Definitions, as the Collections and Clients should not need to be modified in those cases and are already pre-installed in the Runner.

_**Why do it this way:**_

Ansible expects to find collections within a path "collections/ansible_collections" on a series of predefined or default paths within your environment. By default, the Runner has this Path variable prepopulated in a helpful fashion to the pre-installed Collections, this process guides you through modifying that to point at your own versions which you have to maintain yourself.

For development purposes, creating this path in your favourite coding Projects directory, and then checking out the collections under it and renaming them to match the expected namespace may seem slightly arcane but it is the lowest-friction method for ongoing development we have found over many years of doing this.

_**Process:**_

1. Make the directory tree Ansible expects in the same parent code directory that cloudera-deploy is in, e.g.
 
    ```yaml
    cd cloudera-deploy && mkdir -p ../ansible_dev/collections/ansible_collections/cloudera
    ```

    * cloudera is the base namespace of our collection
    * Your Projects directory should also have your Ansible Playbooks and other codebase in it, so that you can mount the root of it to the Runner and have access to all your codebase, e.g. `~/Projects/cloudera-deploy` should be where Cloudera-Deploy is located

2. Fork each of the sub-collections and cdpy into your personal github, and replace <myAccount> with your actual github account below

3. Checkout each of the sub-collections into this folder, e.g.:
 
    ```yaml
    cd ~/Projects
    git clone -b devel https://github.com/<myAccount>/cdpy.git cdpy
    cd ansible_dev/collections/ansible_collections/cloudera
    git clone -b devel https://github.com/<myAccount>/cloudera.exe.git exe
    git clone -b devel https://github.com/<myAccount>/cloudera.cloud.git cloud
    git clone -b devel https://github.com/<myAccount>/cloudera.cluster.git cluster
    ```

    NOTE: The cloned directories above must be named "exe", "cloud" and "cluster", respectively. Ensure you specify the directory name as the last parameter in the command line, as shown above.
    Each of the subcollections should be on the ‘devel’ branch so you can PR them back them with your changes

4. Your Code Project directory should now look something like this:
   
   ```yaml
   /Projects/ansible_dev/collections/ansible_collections/cloudera
   /Projects/ansible_dev/collections/ansible_collections/cloudera/exe
   /Projects/ansible_dev/collections/ansible_collections/cloudera/cloud
   /Projects/ansible_dev/collections/ansible_collections/cloudera/cluster
   /Projects/cdpy
   /Projects/cloudera-deploy
   ```

5. Before you invoke quickstart.sh, set the environment variable below to tell Ansible where to find your code inside your execution environment once it is mounted in the Container at /runner/project:
    ```yaml
    export CLDR_COLLECTION_PATH="ansible_dev/collections"
    export CLDR_PYTHON_PATH=/runner/project/cdpy/src
    ```
    NOTE: You might want to set this in your bash or zsh profile on your local machine so it is persistent

6. Then, when you run quickstart.sh in cloudera-deploy, it will pick up this extra Collection location and add cdpy to the PYTHONPATH, and use these instead of the release versions basked into the Container

7. You can confirm this is working by running this inside the Runner
    ```yaml
    ansible-galaxy collection list
    ```

    It should look something like:
    ```yaml
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
    ```

    If you see duplication of collections because you are using the runner AND mounting your own versions, you probably have not activated the CLDR_COLLECTION_PATH variable correctly, and thus quickstart.sh is not picking it up.

    As another test, you should also be able to invoke python inside the container and use cdpy
    ```yaml
    from cdpy.cdpy import Cdpy
    c = Cdpy()
    c.iam.get_user()
    ```

    To test that cdpy is present and you can access your account as on cmdline

    You may now edit the collections or cdpy and have the changes immediately available for use within the Runner, which is an awful lot easier than having to compile and crossload them after every change.
    
### Install Dependencies without using Runner

We prefer that you use the Runner, because it sets many defaults to avoid common issues and thus save you and us a lot of issue reproduction time. However, we understand that there are many situations where it may not be appropriate, such as air-gapped environments, or when you want to run the install locally on the hardware and not have a separate ansible controller.

1. Create a new virtualenv, or activate an existing one, we do not recommend installing dependencies in system python on most OS.

2. Install dependencies for your hosting infrastructure version following the pathway laid out in the [Dockerfile](https://github.com/cloudera-labs/cldr-runner/blob/main/Dockerfile) in ansible-runner
Install any additional dependencies you may have

NOTE: THe Dockerfile resolves combined dependencies for all our Hybrid Cloud Deployments, you probably only need a subset for your environment.

### Developing within the Runner

While we recommend using the Runner as your execution environment when doing development, actually developing directly against the Runner using something like Visual Studio may not be a great idea due to the file system latency commonly encountered.

Generally when the maintainers work with this system we are editing the files directly on our host system using our IDE, and those files are RW mounted into the container for execution via the `/runner/project` mechanism, which then does not noticeably incur any performance degradation.

### Working with AWS SSO

_**Why:**_

Traditionally AWS users would use static keys, but more recently using temporary credentials via SSO is more commonplace.

The upside of AWS SSO is better credential management, the downside being additional complexity, reliance on the fairly awful AWSCLIv2, and a lack of OOTB automation integration.

_**Setup AWS SSO:**_

1. Install awscli v2 on your local machine

    ```yaml
    brew install awscli
    ```

2. Check you have version 2 (currently 2.5.4)

    ```yaml
    aws --version
    ```

3. Login to AWS SSO via Okta (or however), examine the list of accounts and select the one you want to set up, copy the name to use as your AWS Profile name

4. Setup and login to AWS SSO for your selected account

    ```yaml
    aws configure sso --profile <selected name here>
    ```
    
    * Enter your Start URL, e.g. `https://<app-name>.awsapps.com/start`
    * SSO Region, e.g. `us-east-1`
    * It'll launch your browser (which is why we do it on your local machine)
    * Complete the login process to authenticate the session
    * Select which AWS account you wish to set up
    * Set your default region for this profile, e.g. `us-east-1`
    * Set your default output for this profile, e.g. `json`

5. Now whenever you run a deployment you must set the profile name to match the one you have setup, e.g. in definition.yml

    ```yaml
    aws_profile: my-aws-profile
    ```

NOTE: You will likely have to re-login to SSO before each run

### Working with named credential profiles

For CDP, AWS, and other credential-based access controls it is common to have multiple accounts that may be used from time to time. As such, it is useful to be able to easily, safely, and precisely switch between these accounts.

This is typically achieved through use of multiple named profiles, a good introductory guide to this is [here](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html) for AWS.

To set up a named profile for CDP, follow [this](https://docs.cloudera.com/cdp/latest/cli/topics/mc-cli-generating-an-api-access-key.html) guide to create an API access key, and then [this](https://docs.cloudera.com/cdp/latest/cli/topics/mc-configuring-cdp-client-with-the-api-access-key.html) guide to configure your local credentials by simply adding the `--profile <profile name>` flag to the commands

Credentials are typically stored in a dot-prefix directory in the user profile on Linux, and in the user home directory on Windows, e.g. `~/.aws/credentials`, `~/.cdp/credentials` etc. They may be directly edited in these files if you prefer instead of using the `configure` CLI commands.

By default, Cloudera-Deploy will use the `default` profiles for each account touched, you can set the Definition yaml keys `cdp_profile: <profile name>` and/or `aws_profile: <profile name>` to use a specific named profile instead.

_**Things to know:**_

* There should always be a `default` credential which points to a 'safe' account for when you deliberately or accidentally run commands without specifying a profile. This is generally a dev account, or even an expired credential, to ensure you don't accidentally delete prod.
    
* It is typical with CDP Public Cloud to pair a CDP Tenant with specific Cloud Infrastructure accounts to avoid confusion, therefore if you have a `dev` and `prod` CDP tenant with profiles of the same name, we recommend the paired cloud infrastructure accounts are also named `dev` and `prod`.

### Using a Jumpbox

A common deployment scenario is to funnel all cluster access through a jump/bastion host.

In this case, there are three possibilities:

1. To run the Ansible Runner from the jump host itself
2. To deploy the dependencies within the boundary of the Jump Host
3. To run the Ansible Runner locally and tunnel connections through the jump host.

In scenario 3, the following will be necessary to tunnel both SSH connection and HTTP calls through the jump host.

* HTTP
In the runner, edit `/runner/env/envvars` and add `http_proxy=<proxy>` where<proxy> is the name:port of your http proxy (e.g. a SOCKs proxy running on localhost).

Alternatively, edit quickstart.sh to pass this value through from your local machine if it is available.

* SSH
In your inventory file, under the `[deployment:vars]` group, add the following variable to set additional arguments on the SSH command used by Ansible.

```yaml
    ansible_ssh_common_args='-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ProxyCommand="ssh -W %h:%p -q <jump host>"'
```

Optionally, in your SSH config file (e.g. ~/.ssh/config) you can configure an alias with predefined parameters for the jump host. This makes it easier to manage between different deployments and makes the argument string easier to read.

```yaml
    Host myJump
        IdentityFile ~/.ssh/myKey
        StrictHostKeyChecking = no
        User myUser
        HostName jump.host.name
```

With this SSH config the proxy string would look like this:

```yaml
    ansible_ssh_common_args='-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ProxyCommand="ssh -W %h:%p -q myJump"'
```

## CDP Private Cloud Deployments

CDP Base represents deployment of 'Traditional' clusters via Cloudera Manager, which you might think of historically as hadoop or big data clusters. In practice, there is an Ansible Collection called `cloudera.cluster` which is excellent at configuring Cloudera Manager across various versions of Linux regardless of whether that OS is running in Containers, Virtual Machines, Cloud Infra like EC2, or old school baremetal nodes.

Cloudera.cluster is wrapped within Cloudera-Deploy, so again in practice you are most likely to simply use Cloudera-Deploy as the entrypoint to drive `cloudera.cluster` to create the cluster you want via the Definition mechanism explained in these docs.

The differences are that Cloudera.cluster has been built over many generations to support Cloudera Manager versions 5 to 7, and supports many legacy options and behaviors for user convenience. The result of this is that, while simple deployments are as simple as the more modern CDP Public Cloud, complex deployments will require a greater familiarity with Ansible and Cloudera Manager templates.

### Ansible Controller options

We still strongly recommend that you use the cloudera-runner Docker Container shipped with Cloudera-Deploy as your Ansible controller for CDP Base deployments, because the consistency and logging is going to save you time and effort.

However, it is not uncommon to use the target Cloudera Manager node as the Controller. It is also not uncommon to need to deploy to air-gapped environments or environments where Docker is not permitted.

As such, there is a [script](https://github.com/cloudera-labs/cloudera-deploy/blob/main/centos7-init.sh) for deploying the dependencies for Cloudera-Deploy on centos7 as an example in the repo for Power Users in these scenarios. Note that the script covers dependencies for all Cloudera Products on all current Clouds, so you may wish to exclude parts irrelevant to your scenario.

NOTE: It is worth noting that the `admin_password` field from your Profile, as outlined in the Quickstart, will be used as the Cloudera Manager admin password during `cloudera.cluster` deployments.

### Inventories
An inventory file can be supplied using the traditional -i flag at runtime or by including either
inventory_static.ini or inventory_template.ini in the definitions path.

_**inventory_static.ini**_

An Ansible inventory covering the hosts in-scope for the deployment. An inventory with this name will be picked up automatically if present without using the 
`-i` to specify the inventory file at runtime.

Common inventory group names are:

* cloudera_manager
* cluster_master_nodes
* cluster_worker_nodes
* cluster_edge_nodes

Each host in these groups should be assigned one of the host templates that is defined in cluster.yml. These can be applied per host by setting `host_template=TemplateName` after the hostname, or per group by setting `host_template=TemplateName` in the group vars. See [here](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#adding-variables-to-inventory) for more information on Ansible inventory vars.

Nodes that should be added to the CDP Cluster as defined in the cluster.yml must be added as children to the `cluster` group. For example, if all data nodes are present under the `cluster_worker_nodes` group, then this group should be listed under `cluster:children`. See [here](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#inheriting-variable-values-group-variables-for-groups-of-groups) for an example of setting group children.

All nodes/groups should be added as children of the `deployment` group `[deployment:children]`. This includes the `cluster` group, as well as all non-CDP Cluster groups, for example, Cloudera Manager and supporting infrastructure nodes, perhaps running HAProxy or an RDBMS, that are deployed with the cluster but are not running CDP Roles. This allows for deployment-wide variables to be set, such as the common ssh key that can be used to access all nodes.

WARNING: Do not use the `[all:vars]` group if you are using an independent Ansible Controller as Plays will also be applied to it via the `localhost` mechanic. This includes from your Laptop or the Ansible-Runner supplied in Cloudera-Deploy.

### Process Overview

In this section we will walk through a typical deployment, commenting on options at various points.

* Prepare a Definition Path

You are advised to create a working directory for your definition in your preferred Projects directory.

If you have checked out Cloudera-Deploy from Github following the Quickstart, then you could make a copy of the folder `cloudera-deploy/examples/sandbox` into the same base code Projects directory, and use it as a starting point.

You might then have:
  
  ```yaml
    ~/Projects/cloudera-deploy  # contains quickstart.sh, main.yml
    ~/Projects/my-definition  # copied from ~/Projects/cloudera-deploy/examples/sandbox
  ```

* Infrastructure and Inventory
You are advised to prepare your nodes in advance by whatever mechanism you prefer, and then include them in an Ansible Inventory matching the shape of the Clu
ster you want to deploy.

You may use Cloudera-Deploy to automatically provision EC2 instances based on the inventory template, but this is intended for test and training purposes. An example inventory template is included in the [Cloudera-Deploy Sandbox Example](https://github.com/cloudera-labs/cloudera-deploy/blob/main/examples/sandbox/inventory_template.ini). There is also a copy of the `static_inventory.ini` file in this directory.

If you want to use a static inventory, you may use any of the typical Ansible inventory approaches such as `-i` on the cmdline, the Inventory path, or including a file `inventory_static.ini` in your Definition Path which Cloudera-Deploy will automatically pick up.

You may wish to include additional variables in the Inventory, most typically the ssh user and private key. These should probably be in the `[deployment:vars]` section.

  ```yaml
    [deployment:vars]
    ansible_ssh_private_key_file=~/.ssh/root_key
    ansible_user=root
  ```

NOTE: When you run quickstart.sh to launch the Runner Container, it will mount your local user `~/.ssh` directory into the container, so there is no need to copy keys around provided you use bash expansion for the user home directory.

* Cloudera-Deploy and the Runner

If you have not already done so, you should follow the instructions to setup your OS dependencies and get a copy of Cloudera-Deploy. New users are encouraged to use the [Quickstart](https://github.com/cloudera-labs/cloudera-deploy/blob/main/readme.adoc), while Power Users who want to immediately understand the inner 
workings may wish to follow selected parts of the <<developers.adoc#_detailed_setup_for_developers,Developers>> guide.

Regardless of which method you select, complete the setup process until you are in the orange coloured interactive shell. Ensure that /runner/project within the shell has mounted your Projects directory that contains the Definition working directory you created in step one.

IMPORTANT: At this point you should ensure that the directory listing in `/runner/project` within the Runner matches your projects directory on your local machine, we earlier gave the example of `~/Projects` for this.

When using the Ansible Runner, you should be aware that most changes to the filesystem within the *container* are not persistent - that is, when you kill the container, most changes will be lost. When we are working on files that we want to persist, such as cluster templates, we can keep these on the local filesystem outside of the container, and mount the local filesystem directory inside the container, which is the purpose of the /runner/project mountpoint. This means that we can read and write to those files both inside and outside of the container and you can safely terminate your container without losing them.

Apart from the working dir, the container will mount specific directories out of your local user profile such as ~/.ssh, ~/.config, ~/.aws etc. into the container to pass-through credentials and other requirements. You should assume that any changes outside of these profile directories and /runner/project are not persisted.

NOTE: If we do not specify which directory to mount, the quickstart.sh script will mount the parent directory that the current session is in. i.e. if you run quickstart.sh from the path `~/Projects/cloudera-deploy`, then `~/Projects` in the host filesystem will be mounted to `/runner/projects` in the Container.

NOTE: If you did not stop the container previously, quickstart.sh will ignore any arguments passed and simply give you a new terminal session in the existing container. Therefore, if you want to change the mounted directory you must stop and restart the container with the new path.

### Executing Playbooks

To run the playbook, we use the `ansible-playbook` command. The entry point to the playbook is in main.yml in cloudera-deploy. With our mount point in /runner/project, the full path is `/runner/project/cloudera-deploy/main.yml`. This is passed as the first argument to ansible-playbook.

We also need to provide some additional arguments. We do this with the -e flag. The first argument to pass is the definition_path. This is the path to the directory that contains our definition, inventory, etc.

NOTE: It is usually a good idea to use Cloudera-Deploy to verify your infrastructure before committing to the full deployment, by using the same main playbook and adding the verify tag, as below. This is particularly handy for real-world deployments for a quick sanity check.

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
PLAY [Init Cloudera-Deploy Run] ******************************************************************************************************************************
************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************
************************************************************************************************
Thursday 13 May 2021  18:54:13 +0000 (0:00:00.014)   	0:00:00.014 **********
ok: [localhost]
-----
```

To run the actual full deployment against your inventory, the most common tag to use is `full_cluster`. The complete listing of all tags can be found by reviewing the Plays in https://github.com/cloudera-labs/cloudera-deploy/blob/main/cluster.yml[cluster.yml] in cloudera-deploy.

The command would look something like this, with verbosity at level 2

```yaml
ansible-playbook /runner/project/cloudera-deploy/main.yml \
 -e "definition_path=/runner/project/my-definition/" \
 -t full_cluster
 -vv
```

Expect the deployment to take 30 to 90 minutes or longer, assuming you encounter no errors.

If you do run into issues, most runs are idempotent so you can re-run with increased verbosity of the terminal output by adding the -v flag to the ansible-playbook command. You can scale the verbosity by adding more vs up to -vvvvv for maximum verbosity.

There is nothing you need to do until the playbook completes, however it can be useful to have a scroll through the output and get a feel for what it is doing.

Eventually, you will get to some output that looks like the following. This indicates that Cloudera Manager is being installed, and then a check runs to wait for the server to start responding. When you get past this step, youll be able to access the CM UI in your browser.

It will be installed on the host that was under the cloudera_manager title in your inventory, and on port 7180 for HTTP and 7183 for HTTPS. The username is typically 'admin', and the password will be the admin_password you set in your Profile or Definition.

```yaml
-----
TASK [cloudera.cluster.server : Install Cloudera Manager Server] *********************************************************************************************
************************************************************************************************
Thursday 13 May 2021  19:41:50 +0000 (0:00:06.555)   	0:47:36.812 **********
.
RUNNING HANDLER [cloudera.cluster.common : wait cloudera-scm-server] *****************************************************************************************
************************************************************************************************
Thursday 13 May 2021  19:42:45 +0000 (0:00:09.338)   	0:48:31.682 **********
-----
```

The next important step to watch out for comes right at the end of the playbook. This is the Import Cluster Template step. In this step, the playbook is using the CM API to insert our cluster template, which allows CM to handle the complex logic of deploying the software, pushing out configurations and completing initializations and first runs. During this step, you will not see much useful output in the terminal.

Instead, you should go inside the CM web UI and go to the Running Commands page, where you will be able to drill down into the Import Cluster Template command and watch the individual steps that CM performs. This is the best place to debug any errors that you might encounter during the Import Cluster Template step.

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