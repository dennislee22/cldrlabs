---
id: ansible_troubleshoot
title: Operations & Troubleshooting
sidebar_label: Operations

---

- TOC
{:toc}

## Ansible vs cloudera.cluster

_**Can't detect the required Python library**_

This is most commonly caused by having more than the usual number of versions of Python installed.

Very possibly you've ended up with 2.7, 3.6 and 3.8, or something like that, and the library you are looking for is in the wrong version.
Remove extraneous versions, figure out what is installing them and prevent it.

_**cannot dnf install with python2**_

This usually happens when Ansible decides it simply must use /usr/bin/python even though a perfectly good python3 or other is preferred.

Most commonly this is caused by legacy Ansible python version detection where it will *always* use /usr/bin/python if present. This is fixed by adding `ansible_python_interpreter=auto` to all your target hosts in your inventory, such as in the `[deployment:vars]` group.

## Cloud Credentials

_**403 error from AWS**_

Could be a credential refreshing, retry to see if it’s isolated

_**Error like: Profile given for AWS was not found.  Please fix and retry.**_

This looks to be caused when an underlying call uses authentication from boto, rather than boto3 - and boto doesn’t support the new access scheme. Find the failing Task and report it to us.

_**The SSO session associated with this profile has expired or is otherwise invalid**_

You need to relogin

_**no valid credential sources for Terraform AWS Provider found**_

Using Terraform for Cloudera Deploy, this typically means your SSO token has expired, relogin to continue

_**WSL2 - Browser does not launch**_

The browser may not open during the aws configure sso command if you have the DISPLAY environment variable set in WSL2. This is likely to happen if you run Linux GUI applications in WSL and don't have a browser installed in the WSL distribution.

The quick fix in this case is to unset DISPLAY and re-run the SSO configure command.

## Docker Usage

_**docker: Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?**_

You need to enable WSL integration in the Docker Desktop settings for Resources > WSL Integration

_**SSH_AUTH_SOCK is empty or not set, unable to proceed. Exiting**_

Run the following command to activate the ssh-agent:

```yaml
----
eval `ssh-agent -s`
----
```

If that command doesn't work, try

```yaml
----
eval "$(ssh-agent)"
----
```

_**docker: Error response from daemon: invalid mount config for type "bind": field Source must not be empty.**_

Probably your SSH_AUTH_SOCK is not populated from the ssh-agent. Same fix as above.

_**The path /private/tmp/com.apple.launchd.[some random sting]/Listeners is not shared from the host and is not known to Docker.**_

You need to add you /private directory to the file sharing in your Docker settings for `Resources > File Sharing`

## Code signing for Cloudera Labs

_**GPG Signing is failing for my git commits**_

Could be lots of reasons, start by doing a commit on cmdline with `GIT_TRACE=1` set to see the specific error.

A common cause is your GPG Key expiring, to fix this on OSX:
* comment out no-tty in ~/.gnupg/gpg.conf
* Follow https://stackoverflow.com/a/43728576/4717963[this process] on SO to reset your key expiry
* Re-enable no-tty