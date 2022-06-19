---
title: Introducing Pipelines Into Your Network Automation
date: 2022-06-19
tags:
  - ansible
  - gitlab
  - pipelines
  - juniper
description: 
permalink: posts/{{ title | slug }}/index.html

---
Lets bring your Ansible playbook changes and testing to the next level with GIT Pipelines. Pipelines allow for more advanced testing and changes and adds a gui interface which some may find much easier than manually running Ansible playbooks. Having your playbooks and code hosted on a GIT platform also enables code revision and team reviews/approvals.

### The Ansible setup
For this tutorial we will start with a simple Ansible structure of generating config from templates and applying that to network devices. 
Below shows how you would normally run Ansible, the user would update source files and host vars then manually run the Ansible playbooks:

![Ansible Structure](/images/blog-post-stuff.png)

While Ansible is a great tool this still is quite a manual process. After updating the relevant files you still need to run each playbook and view the output of each task to confirm all ok. You also have no easy way of having a colleague review your changes before they are implemented which in most businesses is an absolute requirement. 

Below is the basic workings of my Ansible playbook which we will use and build upon. You can take a look at the source code on my [Gitlab Account](https://gitlab.com/jamsgrove/introducing-pipelines-into-your-network-automation/-/tree/Basic-Ansible). Please note the repo is split into different branches showing the different stages we will touch on.

```yaml
jamisons@unraid-ubuntu:~/network_iac$ ansible-playbook main.yml --tags precheck

TASK [Generate tacacs configuration] **************************************************************
ok: [vSRX-RTR01]

TASK [Commit check config and generate diff] ******************************************************
changed: [vSRX-RTR01]

TASK [Display "show | compare" output] ************************************************************
ok: [vSRX-RTR01] => {
    "diff_output.diff_lines": [
        "",
        "[edit system tacplus-server]",
        "+   172.16.16.10 {",
        "+       secret \"$9$0CIGq.5/9p\"; ## SECRET-DATA",
        "+       single-connection;",
        "+       source-address 192.168.178.179;",
        "+   }",
        "+   172.16.16.11 {",
        "+       secret \"$9$0CIGq.5/9p\"; ## SECRET-DATA",
        "+       single-connection;",
        "+       source-address 192.168.178.179;",
        "+   }",
        "-   172.16.100.10 secret \"$9$0CIGq.5/9p\"; ## SECRET-DATA",
    ]
}

PLAY RECAP ****************************************************************************************
vSRX-RTR01  : ok=5    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

Playbook run took 0 days, 0 hours, 0 minutes, 4 seconds
```

1. ```TASK [Generate tacacs configuration]``` Using Jinja templates, host and group vars we will generate configuration for the router
2. ```TASK [Commit check config and generate diff]``` Apply the configuration change to the device
3. ```TASK [Display "show | compare" output]``` Display the output of the previous task whichs shows us what changed

The above works great and using the "precheck" and "deploy" tags I can test and deploy changes easy as. In the real world though we need proper testing, peer review of my changes and an easy way to look at all past changes (and who did them!). Enter Gitlab pipelines.

### The pipeline



To follow on with the below, please have the below set up and ready:

1. [Your own instance of Gitlab](https://about.gitlab.com/install/) - <em>docker is the quickest and simplest method</em>
2. [Gitlab runner installed](https://docs.gitlab.com/runner/install/docker.html)
3. [Gitlab runner registered to your Gitlab instance](https://docs.gitlab.com/runner/register/)
4. Add your Ansible repository into your Gitlab instance

Now we are ready lets start with the most important part of the pipeline, the [.gitlab-ci.yml](https://docs.gitlab.com/ee/ci/quick_start/) file. With this file we can define jobs to run, images to build and heaps more. As an intial test lets put the following into our .gitlab-ci.yml file:
```yaml
---
image: jamisons/ubuntu-20-04:v1

variables:
  ANSIBLE_CONFIG: $CI_PROJECT_DIR/ansible.cfg

before_script:
  - ansible-galaxy install juniper.junos

stages:
  - precheck

test_configuration:
  stage: precheck
  script:
    - ansible-playbook main.yml --tags precheck
```
The first line defines an image that we will download from docker-hub to run our playbooks, here i have created my own image called [jamisons/ubuntu-20-04:v1](https://hub.docker.com/repository/docker/jamisons/ubuntu-20-04). It is based on Ubuntu 20.04 and has Ansible installed as well as a number of python modules handy for Juniper automation. See [here](https://docs.docker.com/docker-hub/) on how to make up your own image or feel free to use my one.

The following code defines variables we want to import into the active pipeline, in this case we want to tell docker the "ANSIBLE_CONFIG" variables can be found in our project directory:
```yaml
variables:
  ANSIBLE_CONFIG: $CI_PROJECT_DIR/ansible.cfg
```
