---
title: Ansible role rhel_iso_kickstart
author: Steffen Scheib
---
## Preface

I have been playing around with the idea of writing a blog post series about my Ansible roles for quite a while now. Not only to 'promote' my Ansible roles, but also to show case
how extremely flexible Ansible really is. Often times people think of Ansible as "just another configuration management tool". While this is absolutely true, Ansible - in my
opinion - is so much more than that. Ansible is great in automating a series of tasks across multiple systems - that is really the **strength of Ansible**.

To disclose it right away: I am working at Red Hat as Senior Technical Account Manager Ansible - so please take the above with a grain of salt, as I am most probably biased.

With that being said, let's dive into my Ansible role [`rhel_iso_kickstart`](https://github.com/sscheib/ansible-role-rhel_iso_kickstart) :sunglasses:

## Introduction

I regularly deploy new [Red Hat Satellite](https://www.redhat.com/de/technologies/management/satellite) instances - for testing purposes. Or when I broke one of the Satellite
instances (again) while playing around a bit *too much*. Or when I (once again) move to a new server or basically revamp my complete infrastructure. So you see, I have quite a
need for that :grin:.

I also noticed that a few of my colleagues had the same need, as well as some of our customers. I couldn't find an existing *automated way* of downloading a given RHEL ISO and
implanting a Kickstart into it, while also allowing to customize certain things, such as enabling [FIPS](https://en.wikipedia.org/wiki/Federal_Information_Processing_Standards).

So I decided, I'll just write an Ansible role for everybody to use to ease this regular task.

## Prerequisite: Obtaining an API token to authenticate to the Red Hat Customer Portal

First off, to download an ISO from the [Red Hat Customer Portal](https://access.redhat.com) you need to be a Red Hat subscriber. If you don't own any subscriptions, you can
make use of [Red Hat's Developer Subscription](https://developers.redhat.com/articles/faqs-no-cost-red-hat-enterprise-linux) which is provided at no cost by Red Hat.

Once you created your account and are able to download from the Red Hat Customer Portal, you need to create an API Token, which we'll use to authenticate to the Red Hat
Customer Portal. For that, simply login to Red Hat's Customer Portal and create an [API Token](https://access.redhat.com/management/api).

Note down that token, as we are going to need it for the role to function.

## Prerequisite: Obtaining the checksum of an ISO to download

The Red Hat Customer Portal API enables downloading of ISO images only **by checksum**. To download an ISO, you first need to identify the checksum to pass to the role. This
checksum can be retrieved for any ISO on the Red Hat Customer Portal and can be found on the respective download page of the ISO itself.

For RHEL you can visit [https://access.redhat.com/downloads/content/rhel](https://access.redhat.com/downloads/content/rhel) and simply click on **Show details**.
As an example, the **Red Hat Enterprise Linux 8.8 Binary DVD** will show the following additional details:

```shell
File name: rhel-8.8-x86_64-dvd.iso
File Size: 11.7 GB
SHA-256 Checksum: 517abcc67ee3b7212f57e180f5d30be3e8269e7a99e127a3399b7935c7e00a09 
Last Updated: 2023-04-26
```

We are going to need the **SHA-256 Checksum** :slightly_smiling_face:.

## Installation

Installation is easy and straight forward. I tag my Role releases using [semantic versioning](https://semver.org/) on GitHub. Installing a specific tag works through providing
a `requirements.yml` to the `ansible-galaxy` command.

If you'd want to install v2.0.0 of my Role, you could use the following `requirements.yml`:

```yaml
---
roles:
  - name: 'rhel_iso_kickstart'
    src: 'https://github.com/sscheib/ansible-role-rhel_iso_kickstart.git'
    scm: 'git'
    version: 'v2.0.0'
```

Installing it is as easy as that:

```shell
ansible-galaxy role install -r requirements.yml
```

## A deeper look into the role

So what actually does this role?

The role does the following:

1. Download a RHEL ISO to your local filesystem - that can be your local machine or a server in a data center
1. Optional: Unpack the ISO
1. Optional: Add a custom Kickstart or the Kickstart shipped with the role to the unpacked ISO
1. Optional: Adjust the Kickstart to set a root password, create users and add custom `%post` sections
1. Optional: Validate the Kickstart using `ksvalidator`
1. Optional: Adjust the kernel parameters to load the Kickstart automatically
1. Optional: Enable FIPS in the kernel parameters
1. Optional: Adjust the timeout of GRUB, so you don't have to wait 60 seconds for the installation to start
1. Optional: Adjust the GRUB menu entry to use; Either validate the ISO before installing or directly install it
1. Optional: Create the actual ISO
1. Optional: Make the ISO bootable on BIOS and UEFI systems
1. Optional: Implant an MD5 sum so it can be checked during booting

TL;DR: Quite a lot.

Chances are that you do not need or want to use certain features of the role, such as enabling FIPS. Then I have good news for you: All steps but the very first are optional and
can be configured through variables. :sunglasses:

## Use cases

I have identified three major use cases for the role:

1. Only download an ISO, don't touch it
1. Download the ISO and enable FIPS, so you don't have to do it manually every time before the ISO boots, but do not perform any other changes to the ISO
1. Embed a customized Kickstart into the ISO to use it for unattended installing

I am sure, there plenty of other use cases I haven't thought of yet. If you happen to have a different use case, consider contributing with a feature request or
simply comment on this blog post :slightly_smiling_face:

Now, let's take look at the above use cases and how to configure the role to achieve each of them.

### Downloading an ISO

Downloading the ISO is the easiest of the three use cases. You'll only need to configure:

- Which ISO to download
- Where to download it to
- The permissions of both the download directory and the ISO
- Provide the API token to authenticate against the Red Hat Customer Portal

An example of a playbook could look something like this:

```yaml
- hosts: 'localhost'
  gather_facts: false
  roles:
    - name: 'rhel_iso_kickstart'
  vars:
    # the checksum of a RHEL 8.8 ISO to download from the Red Hat Customer Portal
    checksum: '517abcc67ee3b7212f57e180f5d30be3e8269e7a99e127a3399b7935c7e00a09'
    download_directory: '/home/steffen/workdir'
    download_timeout: '3600'
    download_directory_owner: 'steffen'
    download_directory_group: 'steffen'
    download_directory_mode: '0755'
    iso_owner: 'steffen'
    iso_group: 'steffen'
    iso_mode: '0600'
    api_token: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          [..]
```

### Downloading an ISO and enabling FIPS for the ISO

Enabling FIPS on top of downloading the image is not much more complicated. You need to configure the following:

- Which ISO to download
- Where to download it to
- The permissions of both the download directory and the ISO
- Provide the API token to authenticate against the Red Hat Customer Portal
- Disable Kickstart validation
- Specify a temporary work directory and the permissions to it
- Specify a temporary mount path to mount the original downloaded ISO (to extract it's content)
- Destination path to the custom ISO along with the permissions of it
- Enabling the option to enable FIPS
- Enabling the option to recreate the custom ISO every time
- Enabling the option to implanting an MD5 checksum which can be checked during booting

An example playbook could look something like this:

```yaml
- hosts: 'localhost'
  gather_facts: false
  roles:
    - name: 'rhel_iso_kickstart'
  vars:
    # the checksum of a RHEL 8.8 ISO to download from the Red Hat Customer Portal
    checksum: '517abcc67ee3b7212f57e180f5d30be3e8269e7a99e127a3399b7935c7e00a09'
    download_directory: '/home/steffen/workdir'
    download_directory_owner: 'root'
    download_directory_group: 'root'
    download_directory_mode: '0755'
    iso_owner: 'root'
    iso_group: 'root'
    iso_mode: '0600'
    validate_kickstart: false
    temporary_mount_path: '/mnt'
    temporary_work_dir_path: '/home/steffen/workdir'
    temporary_work_dir_path_owner: 'root'
    temporary_work_dir_path_group: 'root'
    temporary_work_dir_path_mode: '0755'
    temporary_work_dir_source_files_path: 'src'
    temporary_work_dir_source_files_path_owner: 'root'
    temporary_work_dir_source_files_path_group: 'root'
    temporary_work_dir_source_files_path_mode: '0755'
    dest_dir_path: '/home/steffen/workdir'
    custom_iso_owner: 'root'
    custom_iso_group: 'root'
    custom_iso_mode: '0755'
    force_recreate_custom_iso: true
    implant_md5: true
    enable_fips: true
    api_token: !vault |
          $ANSIBLE_VAULT;1.1;AES256
```

### Download a RHEL ISO, implant a Kickstart, create users, add custom `%post` sections, implant an MD5 checksum and enable FIPS

This use case leverages basically all the functionality the role offers. It will:

- Download a RHEL ISO
- Implant a Kickstart into the ISO
- Add create user statements to the Kickstart
- Implant custom `%post` sections, but adding to the default ones
- Configure SSH keys for the users that are created
- Implant a MD5 checksum
- Enable FIPS

A thorough example of the possible usage can be found down below. Some of the variables are redundant, as they merely reflect the defaults already set, but I wanted to show
*what* can be changed.

Example playbook:

```yaml
{% raw %}
---
- hosts: 'localhost'
  gather_facts: false
  roles:
    - name: 'rhel_iso_kickstart'
  vars:
    # the checksum of a RHEL 8.8 ISO to download from the Red Hat Customer Portal
    checksum: '517abcc67ee3b7212f57e180f5d30be3e8269e7a99e127a3399b7935c7e00a09'
    download_directory: '/home/steffen/workdir'
    download_timeout: 3600
    kickstart_path: 'example.ks'
    validate_kickstart: true
    ksvalidator_package_name: 'pykickstart'
    temporary_mount_path: '/mnt'
    temporary_work_dir_path: '/home/steffen/workdir'
    temporary_work_dir_path_owner: 'root'
    temporary_work_dir_path_group: 'root'
    temporary_work_dir_path_mode: '0755'
    temporary_work_dir_source_files_path: 'src'
    temporary_work_dir_source_files_path_owner: 'root'
    temporary_work_dir_source_files_path_group: 'root'
    temporary_work_dir_source_files_path_mode: '0755'
    dest_dir_path: '/home/steffen'
    xorriso_package_name: 'xorriso'
    isolinux_bin_path: 'isolinux/isolinux.bin'
    boot_cat_path: 'isolinux/boot.cat'
    pxelinux_cfg_path: 'isolinux/isolinux.cfg'
    grub_cfg_path: 'isolinux/grub.conf'
    download_directory_owner: 'steffen'
    download_directory_group: 'steffen'
    download_directory_mode: '0755'
    cleanup_iso: false
    cleanup_work_dir: false
    iso_owner: 'steffen'
    iso_group: 'steffen'
    iso_mode: '0600'
    custom_iso_owner: 'root'
    custom_iso_group: 'root'
    custom_iso_mode: '0755'
    force_recreate_custom_iso: true
    grub_menu_selection_timeout: 3
    implant_md5: false
    implantisomd5_package_name: 'isomd5sum'
    quiet_assert: false
    enable_fips: true
    post_sections: >
      {{
         _def_post_sections +
         [
           {
             'name': 'Custom %post section',
             'template': 'custom_post.j2'
           },
           {
             'name': 'Another %post section',
             'template': 'another_post.j2'
           }
         ]
      }}
    users:
      - name: 'ansible-user'
        gid: 2000
        uid: 2000
        gecos: 'Ansible User'
        create_user_group: true
        groups:
          - 'wheel'
        shell: '/bin/bash'
        home: '/home/remote-ansible'
        privileged: true
        lock: false
        password: !vault |
              $ANSIBLE_VAULT;1.1;AES256
              [..]
        authorized_keys:
          - !vault |
              $ANSIBLE_VAULT;1.1;AES256
              [..]

          - !vault |
              $ANSIBLE_VAULT;1.1;AES256
              [..]
    kickstart_root_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          [..]
    api_token: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          [..]
{% endraw %}
```

## Conclusion

I hope this role spares you a little time in your day to day work. That's already it for the time being - I hope you enjoyed it :sunglasses:
