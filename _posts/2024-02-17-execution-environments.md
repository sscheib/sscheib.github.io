---
title: Execution Environments - From Zero to Hero. An in-depth explanation.
author: Steffen Scheib
---

## Preface

To understand why execution environments (EEs) are such a crucial part of modern Ansible we need to briefly look into the past.
To a time where there were no EEs, but *something* else to accommodate for certain situations.

In the following brief look into the past, I'll cover only the basics. For experienced Ansible users which are aware of Ansible's history this chapter is most likely useless -
feel free to skip to the next chapter :slightly_smiling_face:.

Okay, let's get into it :sunglasses:.

## A quick look into the past

Probably most users of Ansible encountered at some point a situation, where they needed to install a specific version of a Python module as a specific module of a collection
required it to be installed.

Let's take as an example [`community.general.proxmox`](https://docs.ansible.com/ansible/latest/collections/community/general/proxmox_module.html). The
module `proxmox` [specifies](https://docs.ansible.com/ansible/latest/collections/community/general/proxmox_module.html#requirements) the following requirements:

> The below requirements are needed on the host that executes this module.
> - proxmoxer <!-- markdownlint-disable-line MD032 -->
> - requests

Both [`proxmoxer`](https://github.com/proxmoxer/proxmoxer) and [`requests`](https://github.com/psf/requests) are
[Python packages](https://packaging.python.org/en/latest/tutorials/installing-packages/).

At this point we know, we need to install both Python packages *somehow* on our system where we execute Ansible. This leaves us with basically three choices:

1. Use the system's package manager to install the packages - if packaged versions are available
1. Install the Python packages using [`pip installs packages`, better known as `pip`](https://ianbicking.org/blog/2008/10/pyinstall-is-dead-long-live-pip.html) to install the
   Python packages either in the user context ([`--user`](https://pip.pypa.io/en/stable/user_guide/#user-installs)) or in the system context (**don't ever install
   Python packages in the system context!**)
1. Use a [`Python Virtual Environment`, better known as `venv`](https://docs.python.org/3/library/venv.html)

We can choose either of the three options above and we should be good to go, right?

*Theoretically*, yes. *Practically*, not really - well, at least it comes with its challenges.

Why? Allow me to tell you little a story that might sound very familiar to you :innocent:.

Let's assume we'd pick option three, the Python Virtual Environments. `venvs` are usually what most Ansible users would go with, as it 'separates' the Python packages of the
actual Python system packages and therefore keeps the actual system 'clean'.

Okay great, now you installed both `requests` and `proxmoxer`. It's in your `venv`. Now you can get started with developing the Ansible role you wanted.

You start developing your role and you install more and more Ansible collections. With it, you install more and more Python packages as well as system packages, because, of
course, most of the collections depend on specific Python and/or system packages.

Finally you are done: You couldn't be happier right now. Your role works *exactly* as you've imagined.

Now a colleague asks you whether you can share the Ansible code you have developed as he is in need of *exactly* such a role you have been working very hard on. Since you
published your role on your company's git, you point your colleague to the git repository of your Ansible role.

The colleague comes back to you asking:
> What Python packages do I need to install to make your role work?

Of course you have the answer: You'll simply look what's installed inside your `venv` which you have been using the whole time while developing the role. You extract the list
of Python packages and send it to your colleague. Your colleagues comes back once again because the role still doesn't work.

Then you remember: Right! You also installed system packages and made *some* modifications to your system's configuration in order to make *something* work. If you just could
remember *what* you installed and modified.

Does that sound familiar to you? :joy:

It doesn't even need to be a colleague that asks you to use your role. The same issue happens when you migrate to a new development machine or upgrade to a newer operating
system version, etc. Certainly it happened to me. And not only once :rofl:.

In the above story, I haven't even started pulling in different versions of Ansible and/or Python that make things even more complex. Let alone if you'd like to deploy
that role now in production where there are stricter security requirements, of course.

Of course, you could argue that a proper documentation and a well-defined [Python requirements file](https://pip.pypa.io/en/stable/reference/requirements-file-format/)
basically eliminates all of the issues above. You are right - for the most part at least. There are situations where a perfect documentation is not sufficient. Just imagine
you develop on Debian and you want to migrate to a different operating system. System package names and system configurations are most definitively different. Sure, you'll be
able to make it work, but it requires effort. Effort that is unnecessary and a cumbersome burden most people would like to avoid because they can spent their time better with
something more productive.

So, what's then a better solution to achieve the same - or even better - outcome?
Meet [Ansible's execution environments (EEs)](https://docs.ansible.com/ansible/latest/getting_started_ee/index.html#getting-started-with-execution-environments).

## Ansible's execution environments (EEs): An introduction

So what exactly are EEs?

First and foremost, EEs are container images - they are basically packaged Ansible control nodes. Essentially, you can picture it as your development machine
or your centralized Ansible control node that runs your Ansible content in production, packaged in a container image with everything it needs to run specific Ansible content.

EEs come by default with a few things:

1. Ansible, usually `ansible-core`
1. [`ansible-runner`](https://ansible.readthedocs.io/projects/runner/en/latest/)
1. A specific Python version that fits the Ansible version used

That's really it - for the bare minimum.

But what makes EEs so powerful and versatile is their great flexibility bundled with all the advantages of containers. If you haven't worked with containers before, you might
not see the massive benefits of EEs just yet. I understand that. Some people might even be afraid of containers as they haven't been using them until this point.

Rest assured: Containers are not bad. They are actually awesome if you understand the benefits of them and understand that you need to adapt in some ways to them.
This might sound at first like a big task, but it really is not.

Especially in the context of EEs, containers are not just not bad, but are *the* solution to all the problems I outlined above. I know, this might not be clear yet at this
point. You - hopefully - understand or at the very least begin to understand, what I mean once you finished reading this blog post.

For those who are afraid of containers: You don't *really* need to work with containers in that sense when dealing with EEs. When building and executing EEs, you essentially
work with two 'wrappers' that take care of building and running your containers in such a way that you usually don't need to interact with container images and containers
directly. I explain all of that in greater detail a bit later.

With the knowledge of what EEs are, I guess everyone can easily imagine why they solve the issues `venvs` have.

Essentially, you **define and extend your EE** as necessary **while you actively develop** Ansible content. This is *exactly* the same workflow you are used to when developing
Ansible Content with `venvs`. This time however, since everything is now neatly packaged in a container, you can now essentially share your development environment. With
*everything* you have modified. Be it settings within the EE, installed system packages or Python packages, as well as every Ansible content you depend on when running your
Ansible code.

EEs effectively eliminate the ".. but it works on my machine!" :sunglasses:.

### Execution environments: Getting started

Working with EEs is pretty straight-forward once you know what you need. In the previous chapter I referred to two 'wrappers' that enable you to use and create EEs - these are:

1. [`ansible-navigator`](https://ansible.readthedocs.io/projects/navigator/)
1. [`ansible-builder`](https://ansible.readthedocs.io/projects/builder/en/latest/)

Let me give you a *very brief* overview over these tools.

#### `ansible-navigator`

`ansible-navigator` is essentially an extension for all (well, most of) the `ansible-` commands (e.g. `ansible-playbook`, `ansible-inventory`, `ansible-doc`, etc.) you are
used to when working with Ansible on the command line (CLI). You *need* `ansible-navigator` to run your Ansible content via CLI inside an EE, as `ansible-playbook` is not
able to run something inside an EE.

#### `ansible-builder`

`ansible-builder` is the tool to use when building execution environments. It makes creating and extending existing EEs easy and straight-forward. Under the hood, it
essentially interacts with [`podman`](https://podman.io/) to create the images.

Now that we know the *very basics* about EEs, let's look a little deeper.

#### Existing EEs: An overview

The understand the concept behind EEs, please allow me to contextualize the Ansible ecosystem with regards to EEs a little.

There are existing EEs which you can utilize to get started. Both the Ansible community and Red Hat maintain a few EEs. The ones maintained by the
Ansible community are either hosted on [quay.io](https://quay.io/) or on GitHub's Container Registry [ghcr.io](https://ghcr.io). The ones Red Hat maintains are officially called
*Ansible Automation Platform execution environments*, which are [*Certified Container Images*](https://catalog.redhat.com/software/containers/explore) and are provided via
Red Hat's Container Registry [registry.redhat.io](https://catalog.redhat.com/). I'll refer to Red Hat's EEs typically as "certified EEs", as it makes communication
easier :slightly_smiling_face:.

The difference between EEs of the Ansible community (we at Red Hat, usually refer to them as *upstream*) and Red Hat's EEs (we at Red Hat, usually refer to them as
*downstream*) is that upstream EEs are *usually* based on CentOS stream, while downstream EEs are based on
[Red Hat's Universal Base Image (UBI)](https://catalog.redhat.com/software/base-images) and are *essentially* Red Hat Enterprise Linux (RHEL) in containers, if you will.

:information_source: To use Red Hat certified EEs you need to be a Red Hat subscriber If you don't own any subscriptions, you can make use of
[Red Hat's Developer Subscription](https://developers.redhat.com/articles/faqs-no-cost-red-hat-enterprise-linux) which is provided at no cost by Red Hat.

The major difference, however, is the level of support you'll get. Upstream in general moves really fast and does not do any
[backports](https://en.wikipedia.org/wiki/Backporting) for older Ansible versions.
In contrast, Red Hat supports their EEs for a [defined life cycle](https://access.redhat.com/support/policy/updates/ansible-automation-platform) and backports bug fixes and/or
security fixes to older EEs which are using older `ansible-core` versions, which are still under support by Red Hat.

Please don't get confused that the [linked life cycle](https://access.redhat.com/support/policy/updates/ansible-automation-platform) points to the Ansible Automation Platform
life cycle. That's not a mistake. The reason being that Red Hat treats several components of the Ansible ecosystem (such as Execution and Decision Environments, `ansible-core`,
`ansible-navigator`, `ansible-builder` among other things) as one *platform*. As Ansible Automation Platform subscriber, you'll get a defined set of components in a
certain version, which ensures that all components are compatible with each other.

Upstream essentially treats various components, such as EEs, `ansible-builder`, `ansible-navigator` etc. as separate projects and therefore you *might* encounter difficulties
using the latest upstream versions together.

**Please don't get me wrong**: I am not saying you should avoid upstream releases. That's not what I am saying at all. Upstream projects are *the core of everything* we do at
Red Hat as we follow the ['upstream first'](https://www.redhat.com/en/about/open-source/participation-guidelines) guideline and you should definitively make use of upstream
projects. However, you *might* want to reconsider using upstream projects if you are going to use them in production or you require a certain level of support to sustain your
business.

:warning: Please keep in mind that this is a **very simplified** overview of upstream and downstream projects in the Open Source world. If you run a business and are about
to make a decision to either use upstream or downstream projects, I highly encourage you to do your own research to build up a solid understanding of the topic before
you decide.

Now that the *basic* principles of upstream vs. downstream with regards to EEs are out of the way, let's actually get our hands dirty :slightly_smiling_face:.

#### Deciding which existing EE to obtain

Alright, let's obtain one EE and start using it - but where do you get them and which one to chose?

For upstream EEs, you are probably going to choose either the [Ansible Creator EE](https://github.com/ansible/creator-ee), which is aimed towards Ansible content developers
or the [AWX EE](https://quay.io/repository/ansible/awx-ee) which is meant to be used when running your code in [AWX](https://github.com/ansible/awx) (one of Red Hat's
upstream projects that are part of the Ansible Automation Platform).

In this blog post I am going to focus on using Red Hat's certified EEs.

Certified EEs come, at the time of this writing, in different variants:

- Either based on UBI 8 or UBI 9
- Either contain `ansible-core` *only* or `ansible-core` and a set of [*certified collections*](https://catalog.redhat.com/software/search?target_platforms=Red%20Hat%20Ansible%20Automation%20Platform)
- In a *special* case, one EE contains Ansible Engine 2.9, but this EE is going to go away in the future as Ansible Engine reached its downstream end of life in December last
  year (31.12.2023). This EE is/was provided merely for the reason to support customers in their migration from older Ansible Automation Platform versions, as Ansible itself
  has changed drastically.

*Technically* that's four variants, but UBI 8 or UBI 9 'only' change the underlying operating system :sweat_smile:.

Again, Red Hat's EEs are hosted on [registry.redhat.io](https://catalog.redhat.com) where the 'user-browsable interface' is [catalog.redhat.com](https://catalog.redhat.com).
When browsing to [registry.redhat.io](https://catalog.redhat.com) with your browser, you'll be redirected to [catalog.redhat.com](https://catalog.redhat.com) automatically, but
you'll be pulling the EEs of `registry.redhat.io`.

Now, *finally*, here are some of the available EEs:

- [ansible-automation-platform/ee-minimal-rhel8](https://catalog.redhat.com/software/containers/ansible-automation-platform/ee-minimal-rhel8/62bd87442c0945582b2b4b37)
- [ansible-automation-platform/ee-minimal-rhel9](https://catalog.redhat.com/software/containers/ansible-automation-platform/ee-minimal-rhel9/6447df2aa123f7fc409f847e)
- [ansible-automation-platform-24/ee-minimal-rhel9](https://catalog.redhat.com/software/containers/ansible-automation-platform-24/ee-minimal-rhel9/643d4c13fae71880450b6108)
- [ansible-automation-platform-24/ee-supported-rhel9](https://catalog.redhat.com/software/containers/ansible-automation-platform-24/ee-supported-rhel9/643d4c7255839fe0f27b0f30)
- [ansible-automation-platform-24/ee-supported-rhel8](https://catalog.redhat.com/software/containers/ansible-automation-platform-24/ee-supported-rhel8/63a333ce183540f5962ae01d)
- [ansible-automation-platform-24/ee-minimal-rhel8](https://catalog.redhat.com/software/containers/ansible-automation-platform-24/ee-minimal-rhel8/63a3338544b6f291781716c7)
- [ansible-automation-platform-24/ee-29-rhel8](https://catalog.redhat.com/software/containers/ansible-automation-platform-24/ee-29-rhel8/63a3322ccdc6fa07ca9d7527)

You can already tell a difference from the namespace portion of the EEs. `ansible-automation-platform/ee-minimal-rhel9` vs. `ansible-automation-platform-24/ee-minimal-rhel9`.
Moreover, there is a `supported` variant and a `minimal` variant.

First off: Don't get confused. `supported` in this context means, it will contain `ansible-core` **and** *some* **supported** Ansible collections. The `minimal`
variant is exactly as supported as the `supported` one is, albeit it's totally understandable if you are confused :smile:. The `minimal` EE contains `ansible-core` **only** -
no collections or anything else.

There is also this *special* EE, which contains Ansible Engine 2.9: `ansible-automation-platform-24/ee-29-rhel8`. Again, this EE is/was provided merely for the reason to
support customers in their migration from older Ansible Automation Platform/Ansible versions.

Secondly, the namespace difference (`ansible-automation-platform/ee-minimal-rhel9` vs. `ansible-automation-platform-24/ee-minimal-rhel9`) is due to 'historic' reasons,
basically.

Nowadays, we encourage customers to make use of the *versionless* or *multi stream* EEs as *versioned* or *single stream* EEs are going to go away in the future.
I prefer using the terms *versionless* and *versioned* over *multi stream* and *single stream*, as it - in my opinion - more accurately describes what is meant.

*Versioned* in this specific case refers to EEs that are build for a *specific* Ansible Automation Platform **version** - such as `ansible-automation-platform-24/ee-minimal-rhel9`.
From this example, the Ansible Automation Platform version that contains the EE `ee-minimal-rhel9` is 2.4. This is how EEs where distributed in the past.

Nowadays, EEs are rather independent of the Ansible Automation Platform version and are distributed '*independently*', meaning not tied to a specific Ansible Automation
Platform version in that sense. Hence: *versionless*.

The idea behind the *versionless* EEs is basically to pick and chose which `ansible-core` version you'd like to have included in your EE. The
[version tags](https://catalog.redhat.com/software/containers/ansible-automation-platform/ee-minimal-rhel8/62bd87442c0945582b2b4b37/history) of, for instance,
`ansible-automation-platform/ee-minimal-rhel8` correspond to `ansible-core` versions.

One thing important to know: Often, container images make use of the `latest` tag, which commonly - who would have guessed it - refers to the latest available version of the
container image. But when you are trying to pull `ansible-automation-platform/ee-minimal-rhel8:latest` using `podman`, you'll be greeted with an error:

```shell
# podman pull ansible-automation-platform/ee-minimal-rhel8:latest
Resolved "ansible-automation-platform/ee-minimal-rhel8" as an alias (/etc/containers/registries.conf.d/001-rhel-shortnames.conf)
Trying to pull registry.redhat.io/ansible-automation-platform/ee-minimal-rhel8:latest...
WARN[0001] Failed, retrying in 1s ... (1/3). Error: initializing source docker://registry.redhat.io/ansible-automation-platform/ee-minimal-rhel8:latest: reading manifest latest in registry.redhat.io/ansible-automation-platform/ee-minimal-rhel8: unsupported: This repository does not use the "latest" tag to track the most recent image and must be pulled with an explicit version or image reference. For more information, see: https://access.redhat.com/articles/4301321
```

:information_source: Don't get confused that I am using `podman` to pull the EE. You don't have to do the same. We'll be using `ansible-builder` for that. Using `podman` in
this specific case is just easier.

In [Red Hat's linked knownledgebase article](https://access.redhat.com/articles/4301321) it is explained *why* this change was introduced for a number of container images if
you are curious of the *why*. In the context of the *versionless* EEs you only need to know, that it doesn't work :slightly_smiling_face:.

After all, users typically want the latest available version of `ansible-core` within the same *minor version* (the second digit in e.g 2.**15**.9).
It is not a good idea to jump from say `ansible-core` 2.15 to `ansible-core` 2.16 *without* evaluating what *changed* in between those versions. On the other hand,
you'd most likely want to have the *latest 2.15* version, e.g. 2.15.9, as *usually* there are no *breaking* or *major* changes within the same minor version.

If that's a use-case you can relate to, then I have good news for you: You can specify the tag `2.15` on the execution environment, e.g.
`ansible-automation-platform/ee-minimal-rhel8:2.15`, which will pull always the latest available version within this minor version, which is 2.15.x.

If you want to be more specific, you can do that too, simply specify a tag with an additional z-version (the last digit in e.g. 2.15.**9**), e.g.
`ansible-automation-platform/ee-minimal-rhel8:2.15.9`

And if you want to make *extra sure* that the image your a pulling does not *change* even if you specify a z-version, you can include the `SHA checksum` (also called
`digest`) of the image in the name:
`registry.redhat.io/ansible-automation-platform/ee-minimal-rhel8@sha256:2a17184e6ea2200b1335c0a39d04159fd040d5ce7fc1990a9d424cc20cfacd4d`. This name refers to
`ee-minimal-rhel-8: 2.15.9-4`.

This way you ensure that you *never* get any other image than you used. In turn, however, this means you need to update the name regularly.

I, personally, don't include the `digest` in the name, as I trust Red Hat to not replace an existing image version with a new one. In terms of security, however,
**a bad actor** *could* replace the image version with a new one. Specifying the `digest` will prevent you from pulling that image version.

One last thing to discuss is the absence of the `supported` variants (the ones that contain besides `ansible-core` also *some* **supported** Ansible collections) of EEs, as
well as the absence of the *special EE* (`ee-29-rhel8`) in *versionless* EEs.

For the *special EE* (`ee-29-rhel8`) the reason is simple: There is nothing to choose from. There is only Ansible Engine 2.9 to use and it was never intended to run on RHEL 9,
therefore there is only the RHEL 8 variant. Period. It will also go away in the future, as already mentioned.

For the `supported` variants the answer is a different one: Red Hat envisions that Ansible Automation Platform users are building their very own and very specific EEs that
fulfill exactly their needs on top of a supported `minimal` EE containing only `ansible-core`.

If you think a moment about it, it hopefully makes perfect sense to you. You likely have very specific requirements for an EE. Such as the `ansible-core` version used, the
included collections, the included Python packages, maybe you need access to a corporate proxy which requires you to include a set of certificate authority certificates into
your EE and so on and so forth. All these things couldn't possibly come from Red Hat.

At first, this might mean to you that you'll need to dive a little deeper into EEs than you'd like to, but it is worthwhile, and really easy.
I mean, the chances are high, that you'll need to build your own EEs in the future if you are in a corporate environment - which most Red Hat customers are - so why not learn
to build them yourself right from the start.

In later sections, we'll learn how to build complex EEs on top of your very own base EEs that fit *exactly* your specific requirement.
Just keep on reading :sunglasses:.

## Using existing EEs with `ansible-navigator`

After we've learned a lot about EEs in the previous chapters, let's actually use them :slightly_smiling_face:. To use the EEs on the CLI, let me first
introduce you to `ansible-navigator`. In an earlier chapter I have briefly touched on `ansible-navigator`, but `ansible-navigator` is crucial for developing and testing
EEs before actually putting them into production - so we need to have a deeper look into it to actually leverage its functionalities.

### Installing `ansible-navigator`

There are multiple ways you can install `ansible-navigator`, which are detailed in the
[installation documentation](https://ansible.readthedocs.io/projects/navigator/installation/).

To install a stable version of `ansible-navigator` on RHEL (I'll refer to it as *downstream*), you need access to one of the Red Hat Ansible Automation Platform
repositories, for instance:

- Red Hat Ansible Automation Platform 2.4 for RHEL 8 x86_64 (RPMs), repository ID: `ansible-automation-platform-2.4-for-rhel-8-x86_64-rpms`
- Red Hat Ansible Automation Platform 2.4 for RHEL 9 x86_64 (RPMs), repository ID: `ansible-automation-platform-2.4-for-rhel-9-x86_64-rpms`

:information_source: Once again the hint that the above repositories require you to be a [Red Hat subscriber](#existing-ees-an-overview).

Simply enable the repository you've chosen using `subscription-manager`:

<!-- markdownlint-disable MD014 -->
```shell
$ sudo subscription-manager repos --enable ansible-automation-platform-2.4-for-rhel-8-x86_64-rpms
```
<!-- markdownlint-enable MD014 -->

Then install `ansible-navigator` using `dnf` or `yum`:

<!-- markdownlint-disable MD014 -->
```shell
$ sudo dnf install ansible-navigator
```
<!-- markdownlint-enable MD014 -->

If you'd rather like to use the latest available upstream version, you can do so with [`pip`](https://pypi.org/project/pip/):

<!-- markdownlint-disable MD014 -->
```shell
$ sudo dnf install python3-pip
$ python3 -m pip install ansible-navigator --user
```
<!-- markdownlint-enable MD014 -->

For `ansible-navigator` to be able to work with EEs, you either need [`podman`](https://podman.io/) or [`docker`](https://www.docker.com/) on your system, as we are going to use
containers and naturally need *something* to handle them :slightly_smiling_face:. I'll use `podman` for the remainder of this post, but all commands I am going to use with
`podman` are working exactly the same when replacing `podman` with `docker`.

After all, we really only run **one** command with `podman`. The remainder will be handled by `ansible-builder` :sunglasses:.

:information_source: The upstream variant of `ansible-navigator` is not exclusively available on RHEL; You can install it on a few more operating systems. Please refer to the
[installation documentation](https://ansible.readthedocs.io/projects/navigator/installation/) to get an overview of all supported operating systems.

:warning: There is one important difference in the above versions: The downstream variant of `ansible-navigator` will default to an EE that is provided and supported by Red Hat,
while the upstream variant will default to an upstream EE. At the time of this writing, the downstream variant of `ansible-navigator` of the Ansible Automation Platform 2.4
repository will pull `registry.redhat.io/ansible-automation-platform-24/ee-supported-rhel8:latest` while the upstream variant will pull `ghcr.io/ansible/creator-ee`.

### Getting started with `ansible-navigator`

First up: `ansible-navigator` has a ton of features. We'll only cover the very basics of it which gets you up to speed with `ansible-navigator`. If you'd like to dive deeper
into specific features of `ansible-navigator`, the [documentation](https://ansible.readthedocs.io/projects/navigator/) provides a good starting point for that.

As I mentioned already, `ansible-navigator` is meant to extend the current CLI tools, which you are most likely familiar with, such as:

- `ansible-playbook`
- `ansible-inventory`
- `ansible-doc`
- etc.

Extending the functionality of the 'original' CLI tools of Ansible means for `ansible-navigator` - in a nutshell - that it can work with EEs. First, you'd maybe think that
only `ansible-playbook` should be extended or superseded, as this command is where we actually run playbooks.

Let's talk about this hypothesis.

It is easiest to show the difference using `ansible-galaxy collection list` and `ansible-doc`.

Let's first check which collections I have installed locally:

```shell
$ ansible-galaxy collection list

# /home/steffen/.ansible/collections/ansible_collections
Collection                  Version
--------------------------- ----------
ansible.controller          4.3.0
ansible.netcommon           5.3.0
ansible.posix               1.5.4
ansible.tower               3.8.6
ansible.utils               2.10.3
ansible.windows             1.11.0
community.crypto            2.16.2
community.docker            3.3.2
community.general           7.3.0
community.mysql             3.4.0
community.postgresql        2.2.0
community.routeros          2.7.0
community.zabbix            1.9.1
containers.podman           1.10.1
infra.ah_configuration      2.0.2
kubernetes.core             2.3.2
openstack.cloud             2.0.0
redhat.openshift            2.2.0
redhat.rhel_system_roles    1.22.0
redhat.satellite            3.15.0
redhat.satellite_operations 2.1.0
sscheib.insights            0.0.2
theforeman.foreman          3.11.0-dev
zabbix.zabbix               1.2.2
```

Now, let's try the same with `ansible-navigator`.

First, as I use the downstream variant of `ansible-navigator`, I need to login to the container registry of Red Hat (`registry.redhat.io`) - this the single podman
command that is *required* to know about when using the downstream variant of `ansible-navigator`:

```shell
$ podman login registry.redhat.io
Username: myUsername
Password:
Login Succeeded!
```

:information_source: To learn which credentials you can use to login to `registry.redhat.io`, Red Hat has created a great
                     [knowledgebase article](https://access.redhat.com/RegistryAuthentication) which also explains how to use tokens for authentication instead of username
                     and password.

Let's run the equivalent command using `ansible-navigator`:

```shell
$ ansible-navigator collections list
--------------------------------------------------------------------------------------------------------------
Execution environment image and pull policy overview
--------------------------------------------------------------------------------------------------------------
Execution environment image name:     registry.redhat.io/ansible-automation-platform-24/ee-supported-rhel8:latest
Execution environment image tag:      latest
Execution environment pull arguments: None
Execution environment pull policy:    tag
Execution environment pull needed:    True
--------------------------------------------------------------------------------------------------------------
Updating the execution environment
--------------------------------------------------------------------------------------------------------------
Running the command: podman pull registry.redhat.io/ansible-automation-platform-24/ee-supported-rhel8:latest
Trying to pull registry.redhat.io/ansible-automation-platform-24/ee-supported-rhel8:latest...
Getting image source signatures
Checking if image destination supports signatures
Copying blob 4a22b5338626 skipped: already exists  
Copying blob 161ea1f419fb skipped: already exists  
Copying blob 72e13691cee8 skipped: already exists  
Copying blob 74e0c06e5eac skipped: already exists  
Copying config 996b5c1825 done  
Writing manifest to image destination
Storing signatures
996b5c1825b0b3d56296900606fe360a652d4633e0f517a6716bf489579808cc


   Name                                 Version    Shadowed    Type         Path
 0│amazon.aws                           6.4.0      False       contained    /usr/share/ansible/collections/ansible_collections/amazon/aws
 1│ansible.builtin                      2.15.9     False       contained    /usr/lib/python3.9/site-packages/ansible
 2│ansible.controller                   4.5.1      False       contained    /usr/share/ansible/collections/ansible_collections/ansible/controller
 3│ansible.netcommon                    6.0.0      False       contained    /usr/share/ansible/collections/ansible_collections/ansible/netcommon
 4│ansible.network                      3.0.0      False       contained    /usr/share/ansible/collections/ansible_collections/ansible/network
 5│ansible.posix                        1.5.4      False       contained    /usr/share/ansible/collections/ansible_collections/ansible/posix
 6│ansible.scm                          2.0.0      False       contained    /usr/share/ansible/collections/ansible_collections/ansible/scm
 7│ansible.security                     2.0.0      False       contained    /usr/share/ansible/collections/ansible_collections/ansible/security
 8│ansible.snmp                         2.0.0      False       contained    /usr/share/ansible/collections/ansible_collections/ansible/snmp
 9│ansible.utils                        3.0.0      False       contained    /usr/share/ansible/collections/ansible_collections/ansible/utils
10│ansible.windows                      1.14.0     False       contained    /usr/share/ansible/collections/ansible_collections/ansible/windows
11│ansible.yang                         2.0.0      False       contained    /usr/share/ansible/collections/ansible_collections/ansible/yang
12│arista.eos                           7.0.0      False       contained    /usr/share/ansible/collections/ansible_collections/arista/eos
13│cisco.asa                            5.0.0      False       contained    /usr/share/ansible/collections/ansible_collections/cisco/asa
14│cisco.ios                            6.1.0      False       contained    /usr/share/ansible/collections/ansible_collections/cisco/ios
15│cisco.iosxr                          7.0.0      False       contained    /usr/share/ansible/collections/ansible_collections/cisco/iosxr
16│cisco.nxos                           6.0.0      False       contained    /usr/share/ansible/collections/ansible_collections/cisco/nxos
17│cloud.common                         2.1.2      False       contained    /usr/share/ansible/collections/ansible_collections/cloud/common
18│cloud.terraform                      1.1.1      False       contained    /usr/share/ansible/collections/ansible_collections/cloud/terraform
19│frr.frr                              2.0.2      False       contained    /usr/share/ansible/collections/ansible_collections/frr/frr
20│ibm.qradar                           3.0.0      False       contained    /usr/share/ansible/collections/ansible_collections/ibm/qradar
21│junipernetworks.junos                6.0.0      False       contained    /usr/share/ansible/collections/ansible_collections/junipernetworks/junos
22│kubernetes.core                      2.4.0      False       contained    /usr/share/ansible/collections/ansible_collections/kubernetes/core
23│microsoft.ad                         1.1.0      False       contained    /usr/share/ansible/collections/ansible_collections/microsoft/ad
24│openvswitch.openvswitch              2.1.1      False       contained    /usr/share/ansible/collections/ansible_collections/openvswitch/openvswitch
25│redhat.amq_broker                    1.3.0      False       contained    /usr/share/ansible/collections/ansible_collections/redhat/amq_broker
26│redhat.eap                           1.3.1      False       contained    /usr/share/ansible/collections/ansible_collections/redhat/eap
27│redhat.insights                      1.0.7      False       contained    /usr/share/ansible/collections/ansible_collections/redhat/insights
28│redhat.openshift                     2.3.0      False       contained    /usr/share/ansible/collections/ansible_collections/redhat/openshift
29│redhat.redhat_csp_download           1.2.2      False       contained    /usr/share/ansible/collections/ansible_collections/redhat/redhat_csp_download
30│redhat.rhel_idm                      1.10.0     False       contained    /usr/share/ansible/collections/ansible_collections/redhat/rhel_idm
31│redhat.rhel_system_roles             1.21.1     False       contained    /usr/share/ansible/collections/ansible_collections/redhat/rhel_system_roles
32│redhat.rhv                           2.4.2      False       contained    /usr/share/ansible/collections/ansible_collections/redhat/rhv
33│redhat.runtimes_common               1.0.2      False       contained    /usr/share/ansible/collections/ansible_collections/redhat/runtimes_common
34│redhat.sap_install                   1.2.1      False       contained    /usr/share/ansible/collections/ansible_collections/redhat/sap_install
35│redhat.satellite                     3.10.0     False       contained    /usr/share/ansible/collections/ansible_collections/redhat/satellite
36│redhat.satellite_operations          1.3.0      False       contained    /usr/share/ansible/collections/ansible_collections/redhat/satellite_operations
37│redhat.sso                           1.2.1      False       contained    /usr/share/ansible/collections/ansible_collections/redhat/sso
38│sap.sap_operations                   1.0.4      False       contained    /usr/share/ansible/collections/ansible_collections/sap/sap_operations
39│servicenow.itsm                      2.1.0      False       contained    /usr/share/ansible/collections/ansible_collections/servicenow/itsm
40│splunk.es                            3.0.0      False       contained    /usr/share/ansible/collections/ansible_collections/splunk/es
41│trendmicro.deepsec                   3.0.0      False       contained    /usr/share/ansible/collections/ansible_collections/trendmicro/deepsec
42│vmware.vmware_rest                   2.3.1      False       contained    /usr/share/ansible/collections/ansible_collections/vmware/vmware_rest
43│vyos.vyos                            4.0.2      False       contained    /usr/share/ansible/collections/ansible_collections/vyos/vyos
```

:information_source: I copied the collection list from the text-based user interface of `ansible-navigator`. A text-based user interface might be known to you from e.g.
`nmtui`, `tmux` or `vim`. All of them are based on the same library: [`ncurses`](https://en.wikipedia.org/wiki/Ncurses), that's why it might look familiar to you.

Looking at the above output, we can notice that there's quite a difference to the previous `ansible-galaxy collection list`.

Why's that?

First, the EE image gets pulled. **After** the EE image was pulled, it started `ansible-navigator` in the text-based user interface (TUI), and then finally the collections are listed.

Now it might become clear why the majority of the `ansible-navigator` commands are run in an EE: To show correct information, `ansible-navigator` needs to access the Ansible
version inside the EE and with it all the collections, the documentation, the Python packages and so on and so forth.

That is why I can run `ansible-navigator doc vyos.vyos.vyos_static_routes` and get the documentation for it, but when I run the equivalent comment with `ansible-doc`, Ansible
only shows a warning:

```shell
$ ansible-doc vyos.vyos.vyos_static_routes
[WARNING]: vyos.vyos.vyos_static_routes was not found
```

The reason being that the collection `vyos.vyos` is installed within the EE `ansible-automation-platform-24/ee-supported-rhel8`, which is used
by `ansible-navigator` in this example, but not on my local system, where the `ansible-doc vyos.vyos.vyos_static_routes` command will look for.

### An overview of the commands of `ansible-navigator`

With the previous chapter, we found out why it makes sense to have the majority of the conventional Ansible CLI tools extended with EE support. But what features does
`ansible-navigator` actually provide?

Let's have a look at the `--help` output of `ansible-navigator`:

```shell
$ ansible-navigator --help
Usage: ansible-navigator [options]

[..]

Subcommands:
 {subcommand} --help
  builder                                        Build [execution environment](https://docs.ansible.com/ansible/devel/getting_started_ee/index.html) (container image)
  collections                                    Explore available collections
  config                                         Explore the current ansible configuration
  doc                                            Review documentation for a module or plugin
  exec                                           Run a command within an execution environment
  images                                         Explore execution environment images
  inventory                                      Explore an inventory
  lint                                           Lint a file or directory for common errors and issues
  replay                                         Explore a previous run using a playbook artifact
  run                                            Run a playbook
  settings                                       Review the current ansible-navigator settings
  welcome                                        Start at the welcome page
```

That's quite a lot of sub-commands, but how do they map to the original Ansible CLI tools?

The answer is not as straight-forward as you might think. First off: `ansible-navigator` is an **extension** to the original Ansible CLI tools, **not** a **replacement**.
Having said that, *some* original Ansible CLI tools are incorporated into `ansible-navigator`:

Let me start with the obvious ones:

| ansible command                           | ansible-navigator command                   |
| :---------------------------------------- | :------------------------------------------ |
| `ansible-galaxy collection list`          | `ansible-navigator collections list`        |
| `ansible-config`                          | `ansible-navigator config`                  |
| `ansible-doc`                             | `ansible-navigator doc`                     |
| `ansible-inventory`                       | `ansible-navigator inventory`               |
| `ansible-playbook`                        | `ansible-navigator run`                     |

The above table shows the most common Ansible CLI tools, but a few are missing:

- `ansible`
- `ansible-test`
- `ansible-vault`

For both `ansible` and `ansible-test` you can use the `exec` sub-command. For an Ansible ad-hoc command this will look like the following:

```shell
$ ansible-navigator exec -- ansible -m ping localhost
--------------------------------------------------------------------------------------------------------------
Execution environment image and pull policy overview
--------------------------------------------------------------------------------------------------------------
Execution environment image name:     registry.redhat.io/ansible-automation-platform-24/ee-supported-rhel8:latest
Execution environment image tag:      latest
Execution environment pull arguments: None
Execution environment pull policy:    tag
Execution environment pull needed:    True
--------------------------------------------------------------------------------------------------------------
Updating the execution environment
--------------------------------------------------------------------------------------------------------------
Running the command: podman pull registry.redhat.io/ansible-automation-platform-24/ee-supported-rhel8:latest
Trying to pull registry.redhat.io/ansible-automation-platform-24/ee-supported-rhel8:latest...
Getting image source signatures
Checking if image destination supports signatures
Copying blob 4a22b5338626 skipped: already exists  
Copying blob 72e13691cee8 skipped: already exists  
Copying blob 161ea1f419fb skipped: already exists  
Copying blob 74e0c06e5eac skipped: already exists  
Copying config 996b5c1825 done  
Writing manifest to image destination
Storing signatures
996b5c1825b0b3d56296900606fe360a652d4633e0f517a6716bf489579808cc
localhost | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

**Personally**, I do not see much sense in using `ansible-vault` and `ansible-test` with `ansible-navigator`; It is just easier to stick with the original CLI tools for them.

If you'd really like to do it, here are the commands:

<!-- markdownlint-disable MD014 -->
```shell
$ ansible-navigator exec -- ansible-test --help
$ ansible-navigator exec -- ansible-vault --help
```
<!-- markdownlint-enable MD014 -->

What essentially is done with `ansible-navigator exec` is to execute commands inside the EE. No more, no less. For the ad-hoc commands, this can definitively be a benefit to
run it inside an EE, but for `ansible-test` and `ansible-vault` I don't see the value. If you know a use-case, please let me know! :slightly_smiling_face:

Lastly, there are also some sub-commands that are specific to `ansible-navigator`:

- `builder`
- `images`
- `lint`
- `replay`
- `settings`

Let me start with the ones, where I do not see a benefit in using them directly with `ansible-navigator`, but your mileage my vary:

- `builder`: You could theoretically build EEs using `ansible-navigator builder`, but under the hood it basically calls `ansible-builder`, so I decided to stick to
  `ansible-builder` directly
- `lint`: With `ansible-navigator lint` you can lint your Ansible code, *if* `ansible-lint` is installed. Essentially, it calls `ansible-lint` under the hood, so I decided to
   stick to `ansible-lint` directly

So what's left for sub-commands are `images`, `replay` and `settings`.

With `ansible-navigator settings` you can view the current settings of `ansible-navigator` and create sample settings.

### Configuration of `ansible-navigator`

`ansible-navigator` [configuration file locations](https://ansible.readthedocs.io/projects/navigator/settings/) follow the same logic as the well-known
`ansible.cfg`.

Following locations are valid:

- `ansible-navigator.yml` in your current working directory
- `~/.ansible-navigator.yml` in your home directory

:information_source: The `ansible-navigator.yml` configuration file in your current working directory has precedence over the `~/.ansible-navigator.yml` in your home directory.

Alternatively, you can override the configuration file location with an environment variable: `ANSIBLE_NAVIGATOR_CONFIG`. Please note that the environment variable
`ANSIBLE_NAVIGATOR_CONFIG` has the highest precedence.

`ansible-navigator` settings can either be stored in `YAML` or `JSON`. For `YAML` the file extension needs to either be `.yml` or `.yaml`. For `JSON` it is `.json`.

Decide on either format (I suggest `YAML`, of course :slightly_smiling_face:) and make sure that only *one* configuration file is present in either of the locations. If
`ansible-navigator` finds more than one configuration file in one of the possible configuration file locations, it will error out:

```shell
$ ansible-navigator settings --help
  Error: Only one file among '/home/steffen/.ansible-navigator.yml',
         '/home/steffen/.ansible-navigator.yaml' and
         '/home/steffen/.ansible-navigator.json' should be present in
         /home/steffen Found: '/home/steffen/.ansible-navigator.yml' and
         '/home/steffen/.ansible-navigator.json'

   Note: Configuration failed, using default log file location.
         (/home/steffen/ansible-navigator.log) Log level set to debug
   Hint: Review the hints and log file to see what went wrong.
```

You can find a [sample configuration](https://ansible.readthedocs.io/projects/navigator/settings/) in the documentation or you can generate one using `ansible-navigator`.

Okay, let's generate a sample configuration using `ansible-navigator settings --sample` and redirect the output to `~/.ansible-navigator.yml`:

```shell
$ ansible-navigator settings --sample > ~/.ansible-navigator.yml
Trying to pull registry.redhat.io/ansible-automation-platform-24/ee-supported-rhel8:latest...
Getting image source signatures
Checking if image destination supports signatures
Copying blob 4a22b5338626 skipped: already exists  
Copying blob 161ea1f419fb skipped: already exists  
Copying blob 72e13691cee8 skipped: already exists  
Copying blob 74e0c06e5eac skipped: already exists  
Copying config 996b5c1825 done  
Writing manifest to image destination
Storing signatures
```

Unfortunately, this leaves us with a broken configuration, as part of the image pulling is also part of the `~/.ansible-navigator.yml` configuration file:

```shell
$ cat ~/.ansible-navigator.yml 
996b5c1825b0b3d56296900606fe360a652d4633e0f517a6716bf489579808cc
--------------------------------------------------------------------------------------------------------------
Execution environment image and pull policy overview
--------------------------------------------------------------------------------------------------------------
Execution environment image name:     registry.redhat.io/ansible-automation-platform-24/ee-supported-rhel8:latest
Execution environment image tag:      latest
Execution environment pull arguments: None
Execution environment pull policy:    tag
Execution environment pull needed:    True
--------------------------------------------------------------------------------------------------------------
Updating the execution environment
--------------------------------------------------------------------------------------------------------------
Running the command: podman pull registry.redhat.io/ansible-automation-platform-24/ee-supported-rhel8:latest
---
ansible-navigator:
#   ansible:
#     config:
#       # Help options for ansible-config command in stdout mode
#       help: False
#       # Specify the path to the ansible configuration file
#       path: ./ansible.cfg
[..]
```

:information_source: From my point of view, the indentation in the generated `ansible-navigator` sample configuration is wrong. Technically, and according to
the [YAML specification 1.2.2](https://yaml.org/spec/1.2.2/#61-indentation-spaces), *any* number of spaces is fine.
> In YAML block styles, structure is determined by indentation. In general, indentation is defined as a zero or more space characters at the start of a line.
I think, however, that either two (preferred) or four spaces are the common number of spaces used for indentation.

The only way I found to overcome this is to create a small configuration file that disables using EEs and will be overwritten by our redirect:

```shell
$ cat ~/.ansible-navigator.yml 
---
ansible-navigator:
  execution-environment:
    enabled: false
...
```

It is not possible to specify the same settings on the CLI as then `ansible-navigator` will error out:

```shell
$ ansible-navigator settings --ee false --sample
Warning: Issues were found while applying the settings.
   Hint: Command provided: 'settings --ee false --sample'

  Error: Settings file found /home/steffen/ansible-navigator.yaml, but failed to load it.
           error was: 'Settings file cannot be empty.'
   Hint: Try checking the settings file '/home/steffen/ansible-navigator.yaml'and ensure it is properly formatted

   Note: Configuration failed, using default log file location. (/home/steffen/ansible-navigator.log) Log level set to debug
   Hint: Review the hints and log file to see what went wrong.
```

### My configuration for `ansible-navigator`

Below you'll find one of configurations I am using. I don't have a configuration in my home directory: The reason being that I store a copy of my `ansible.cfg`, my
`ansible-navigator.yml` and various other configurations in a 'project' directory. This project directory is structured per use-case and that's why I have special settings in
each `ansible-navigator.yml` and `ansible.cfg` - such as number of forks, the Ansible remote user and the EE to use.

I'll write a separate blog post about how and why I structure my projects this way in the distant future, as I have been asked a couple of times already how I develop
Ansible code and how I organize things - but that's a topic for the other blog post.

Below is the configuration of my `ansible-navigator.yml` for my project `ansible-project-satellite`:

```yaml
---
ansible-navigator:
  mode: 'stdout'
  logging:
    level: 'critical'
    append: true
    file: '/dev/null'

  color:
    enable: true
    osc4: true

  editor:
    command: 'vim_from_setting'
    console: true

  inventory-columns:
    - 'ansible_network_os'
    - 'ansible_network_cli_ssh_type'
    - 'ansible_connection'

  execution-environment:
    container-engine: 'podman'
    enabled: true
    image: 'localhost/ee-ansible_project-satellite-rhel-8:latest'
    pull:
      policy: 'never'

  playbook-artifact:
    enable: false

  ansible:
    config:
      help: false
      path: './ansible.cfg'
    doc:
      help: false
    inventory:
      help: false
      entries:
        - './inventory/'

  ansible-lint:
    config: './.ansible-lint'

  time-zone: 'Europe/Berlin'
...
```

It is a very minimal configuration and should not be taken as "the correct configuration to use". This is my preference, yours probably differ.

Following a quick summary of the settings I applied:

1. I don't want to log to a log-file, that's why basically 'redirect' the log to `/dev/null`
1. I'd like to have colors in my terminal :slightly_smiling_face:. And yes, I develop on the CLI using `vim` only :satisfied:!
1. The editor that should be used when editing anything in `ansible-navigator` should be `vim`
1. I added a few more `inventory-columns` that I find interesting
1. I set `podman` as my `container-engine` and specified a specific image to use. I also never want `ansible-navigator` to pull the image, because it is locally available
1. I don't want to use `playbook-artifacts` - we'll talk about that in a moment
1. I specify the `ansible.cfg` and the `inventories` to use and disable all help in the TUI
1. I played around with `ansible-lint` in `ansible-navigator` and left the `ansible-lint` setting pointing to my configuration file
1. Lastly, I specified the time-zone. This will be used for calculating the time when using `playbook-artifacts`

### `playbook-artifacts` in `ansible-navigator`

`ansible-navigator` does not only extend the original Ansible CLI tools with EEs, but also brings some new features to the table. One of them is `playbook-artifacts`.
`playbook-artifacts` are essentially a structured way of saving the events of a playbook run.

At first, this might sound boring and not useful at all, but here is the thing which makes it great:

You can replay any playbook (using `ansible-navigator replay`) by using the created playbook artifact. This allows for easier troubleshooting as every task is written to the
artifact file with both the arguments it was executed with and what the result of it was.

This can be beneficial as you are able to troubleshoot easier. And if you ever hit a roadblock and simply cannot find out what's wrong, you just share the artifact with one of
your colleagues and can get your colleague's help that way.

I personally don't use them at all times. I enable the artifact creation when I need it.

### Modes of `ansible-navigator`

`ansible-navigator` comes with two modes:

1. `interactive`: That's the default and provides you with the TUI to interact with `ansible-navigator`. This mode is required when replaying artifacts. It is also helpful when
looking at existing EEs to determine what's in them.
1. `stdout`: `stdout` is essentially the 'good old look' of `ansible-playbook`. I prefer using this mode for most of the things I do with `ansible-navigator`.

The mode can either be saved in the configuration file ([as in my above configuration file](#my-configuration-for-ansible-navigator) or can be overwritten by specifying the
`-m` or `--mode` parameter, like in the following example:

<!-- markdownlint-disable MD014 -->
```shell
$ ansible-navigator -m stdout
```
<!-- markdownlint-enable MD014 -->

### Navigating in `ansible-navigator` when using the text-based user interface (TUI)

Navigation in the TUI of `ansible-navigator` is easy and straight-forward when you are used to `vim`.

If you are not used to `vim`, here is a quick run through:
Pressing `:` will enable you to call the different functions. Say you wanted to check the images `ansible-navigator` knows about, then you'll simply type `:images` and press enter.

Getting help is done via `:help`, looking at collections is done via `:collections`, and so on.

There are also shortcuts available (as you might have noticed when looking at `:help`). For instance, `:im` calls `:images`, `:h` calls `:help`, etc. The shortcuts are listed
in `:help`. Going back a screen is done by pressing escape on your keyboard and you can quit `ansible-navigator` with `:quit` or `:q`.

So, once you are in one of these functions, you'll notice on the left-hand side a numbering which starts at 0.

Let's look into an example of `ansible-navigator images`:

```plaintext
  Image                                                    Tag                                          Execution environment                 Created                  Size
0│ansible-test-utility-container                           2.0.0                                        False                                 15 months ago            7.7 MB
1│default-test-container                                   7.14.0                                       False                                 9 months ago             1.6 GB
2│ee-ansible_project-zabbix-rhel-8                         latest                                       True                                  2 months ago             436 MB
3│ee-minimal-rhel8                                         2.15                                         True                                  4 months ago             338 MB
4│ee-supported-rhel8                                       latest                                       True                                  2 weeks ago              1.73 GB
5│rootfs                                                   x86-generic-openwrt-22.03                    False                                 4 months ago             12.1 MB
6│rootfs                                                   x86-generic-openwrt-21.02                    False                                 4 months ago             10.5 MB
7│rootfs                                                   x86-generic-snapshot                         False                                 10 months ago            9.15 MB
```

If I now want to know more about the EE `ee-supported-rhel8`, I simply press `3` and I'll be offered a different set of options:

```plaintext
  Image: ee-supported-rhel:latest                                                            Description
0│Image information                                                                          Information collected from image inspection
1│General information                                                                        OS and python version information
2│Ansible version and collections                                                            Information about ansible and ansible collections
3│Python packages                                                                            Information about python and python packages
4│Operating system packages                                                                  Information about operating system packages
5│Everything                                                                                 All image information
```

To know more about the Ansible version and the EEs included collections, I press `2`:

```plaintext
Image: ee-supported-rhel8:latest (primary) (Information about ansible and ansible collections)                                                                                                                                             
 0│---
 1│ansible:
 2│  collections:
 3│    details:
 4│      amazon.aws: 6.4.0
 5│      ansible.controller: 4.5.1
 6│      ansible.netcommon: 6.0.0
 7│      ansible.network: 3.0.0
 8│      ansible.posix: 1.5.4
 9│      ansible.scm: 2.0.0
10│      ansible.security: 2.0.0
11│      ansible.snmp: 2.0.0
12│      ansible.utils: 3.0.0
13│      ansible.windows: 1.14.0
14│      ansible.yang: 2.0.0
15│      arista.eos: 7.0.0
16│      cisco.asa: 5.0.0
17│      cisco.ios: 6.1.0
18│      cisco.iosxr: 7.0.0
19│      cisco.nxos: 6.0.0
20│      cloud.common: 2.1.2
21│      cloud.terraform: 1.1.1
22│      frr.frr: 2.0.2
23│      ibm.qradar: 3.0.0
24│      junipernetworks.junos: 6.0.0
25│      kubernetes.core: 2.4.0
26│      microsoft.ad: 1.1.0
27│      openvswitch.openvswitch: 2.1.1
28│      redhat.amq_broker: 1.3.0
29│      redhat.eap: 1.3.1
30│      redhat.insights: 1.0.7
31│      redhat.openshift: 2.3.0
32│      redhat.redhat_csp_download: 1.2.2
33│      redhat.rhel_idm: 1.10.0
34│      redhat.rhel_system_roles: 1.21.1
35│      redhat.rhv: 2.4.2
36│      redhat.runtimes_common: 1.0.2
37│      redhat.sap_install: 1.2.1
38│      redhat.satellite: 3.10.0
39│      redhat.satellite_operations: 1.3.0
40│      redhat.sso: 1.2.1
41│      sap.sap_operations: 1.0.4
42│      servicenow.itsm: 2.1.0
43│      splunk.es: 3.0.0
44│      trendmicro.deepsec: 3.0.0
45│      vmware.vmware_rest: 2.3.1
46│      vyos.vyos: 4.0.2
47│  version:
48│    details: ansible [core 2.15.9]
```

Now I can see what Ansible collections are part of the EE, great :slightly_smiling_face:.

If you have more than 9 options available, you need to make use of `:10`, followed by pressing enter to get to the 10th item, or `:35` followed by pressing enter to get to
the 35th item. This is exactly the way you'd navigate to line numbers in `vim` :sunglasses:.

:information_source: Whenever there is nothing more `ansible-navigator` can show to you by selecting an item from the list - like with the collections above - it'll simply do
nothing. So when I would want to get more information about the Satellite collection, which is item 38 and type `:38` followed by enter, `ansible-navigator` will do nothing.

### Running playbooks in `ansible-navigator`

Now we've learned a lot about `ansible-navigator`, but let's actually use it.

I have the following *very* simple playbook:

```yaml
---
- name: 'Configure SSHD'
  hosts: 'g_satellites'
  gather_facts: false
  become: true  # required for the role redhat.rhel_system_roles.sshd'
  roles:
    - role: 'file_deployment'
    - role: 'redhat.rhel_system_roles.sshd'
...
```

In the above playbook I merely include two roles:

1. My own role [`file_deployment`](https://github.com/sscheib/ansible-role-file_deployment) which places two files on the managed nodes that are required to configure `sshd`
in the way I want it
1. The RHEL system role to configure `sshd`:
[`redhat.rhel_system_roles.sshd`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/rhel_system_roles/content/role/sshd/)

For this little demo, I specified
[ansible-automation-platform-24/ee-supported-rhel8](https://catalog.redhat.com/software/containers/ansible-automation-platform-24/ee-supported-rhel8/63a333ce183540f5962ae01d) as
the EE to use and set the pull policy to `missing` in the `ansible-navigator` configuration file in my working directory (`ansible-navigator.yml`). I further specified to
use as inventory the directory `./inventory` and to use the `ansible.cfg` that is in my current directory:

```yaml
---
ansible-navigator:
[..]
  execution-environment:
    container-engine: 'podman'
    enabled: true
    image: 'ee-supported-rhel8'
    pull:
      policy: 'missing'
[..]
  ansible:
    config:
      help: false
      path: './ansible.cfg'
    doc:
      help: false
    inventory:
      help: false
      entries:
        - './inventory/'
[..]
```

Let's quickly examine the folder structure in this directory:

```shell
$ tree .
.
├── ansible.cfg
├── ansible-navigator.yml
├── collections
│   └── requirements.yml
├── inventory
│   ├── g_satellites.yml
│   ├── host_vars
│   │   ├── satellite.office.int.scheib.me
│   │   │   ├── 00a_secrets.yml
│   │   │   ├── 00b_secrets_base.yml
│   │   │   ├── 00c_register_satellite.yml
│   │   │   ├── 00d_user_deployment.yml
│   │   │   ├── 00e_package_installation.yml
│   │   │   ├── 00f_service_management.yml
│   │   │   ├── 00g_file_deployment.yml
│   │   └── satellite.pve.ext.scheib.me
│   │       ├── 00a_secrets.yml
│   │       ├── 00b_secrets_base.yml
│   │       ├── 00c_register_satellite.yml
│   │       ├── 00d_user_deployment.yml
│   │       ├── 00e_package_installation.yml
│   │       ├── 00f_service_management.yml
│   │       ├── 00g_sshd.yml
│   └── ungrouped.yml
├── playbooks
│   ├── 000_setup.yml
│   ├── 00_create_kickstart.yml
│   ├── 01a_register_satellite.yml
│   ├── 01b_user_deployment.yml
│   ├── 01c_package_installation.yml
│   ├── 01d_service_management.yml
│   ├── 01e_sshd.yml
│   ├── 01f_generate_ssl_key_pairs.yml
│   ├── files
│   │   ├── sshd_issue.net
│   │   └── sshd_motd
│   ├── group_vars -> ../inventory/group_vars/
│   └── host_vars -> ../inventory/host_vars/
└── roles
    └── requirements.yml

20 directories, 159 files
```

:information_source: The directory contains *way more* files and directories than shown above (see the last line in the output above), but I removed a lot of the output
to shorten it a little.

My hosts are in this specific case in the group `g_satellites` which is defined in the file `inventory/g_satellites.yml`:

```yaml
---
g_satellites:
  hosts:
    satellite.office.int.scheib.me: {}
    satellite.pve.ext.scheib.me: {}
...
```

The corresponding `host_vars` for both hosts are stored in `inventory/host_vars/<hostname>`. Both `host_vars` directories contain a file `00g_sshd.yml` which is where I get
the settings for the RHEL system role `redhat.rhel_system_roles.sshd`.

As an example what one of them looks like:

<!-- markdownlint-disable MD003 MD022 -->
{% highlight yaml %}
{% raw %}
---
sshd_enable: true
sshd_skip_defaults: true
sshd_manage_firewall: true
sshd_manage_selinux: true
sshd_sysconfig: true
sshd_sysconfig_override_crypto_policy: true
sshd:
  AddressFamily: 'inet'
  AllowAgentForwarding: false
  AllowTcpForwarding: false
  AllowStreamLocalForwarding: false
  AuthenticationMethods: 'publickey'
  AuthorizedKeysFile: '%h/.ssh/authorized_keys'
  Banner: '/etc/issue.net'
  ChallengeResponseAuthentication: false
  ChrootDirectory: 'none'
  Ciphers: >-
    {{
      [
        'chacha20-poly1305@openssh.com',
        'aes256-gcm@openssh.com',
        'aes128-gcm@openssh.com',
        'aes256-ctr',
        'aes192-ctr',
        'aes128-ctr'
      ] | join(',')
    }}
  ClientAliveCountMax: 3
  ClientAliveInterval: 0
  Compression: 'delayed'
  FingerprintHash: 'sha512'
  GatewayPorts: false
  HostbasedAuthentication: false
  HostKey:
    - '/etc/ssh/ssh_host_ecdsa_key'
  HostKeyAlgorithms: >-
    {{
      [
        'ecdsa-sha2-nistp256-cert-v01@openssh.com',
        'ecdsa-sha2-nistp384-cert-v01@openssh.com',
        'ecdsa-sha2-nistp521-cert-v01@openssh.com',
        'ecdsa-sha2-nistp256',
        'ecdsa-sha2-nistp384',
        'ecdsa-sha2-nistp521'
      ] | join(',')
    }}
  IgnoreRhosts: true
  IPQoS: 'lowdelay throughput'
  KexAlgorithms: >-
    {{
      [
        'curve25519-sha256@libssh.org',
        'ecdh-sha2-nistp521',
        'ecdh-sha2-nistp384',
        'ecdh-sha2-nistp256',
        'diffie-hellman-group-exchange-sha256'
      ] | join(',')
    }}
  ListenAddress:
    - '0.0.0.0'
  LoginGraceTime: 60
  LogLevel: 'VERBOSE'
  MACs: >-
    {{
      [
        'hmac-sha2-512-etm@openssh.com',
        'hmac-sha2-256-etm@openssh.com',
        'umac-128-etm@openssh.com',
        'hmac-sha2-512',
        'hmac-sha2-256',
        'umac-128@openssh.com'
      ] | join(',')
    }}
  MaxAuthTries: 3
  MaxSessions: 6
  MaxStartups: '10:30:60'
  PasswordAuthentication: false
  PermitEmptyPasswords: false
  PermitRootLogin: false
  PermitTunnel: false
  PermitTTY: true
  PermitUserEnvironment: false
  PermitUserRC: false
  PidFile: '/var/run/sshd.pid'
  Port: 1905
  PrintMotd: true
  Protocol: 2
  PubkeyAcceptedKeyTypes: 'ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521'
  PubkeyAuthentication: true
  RekeyLimit: '64M 4H'
  Subsystem: 'sftp internal-sftp'
  StrictModes: true
  SyslogFacility: 'AUTH'
  TCPKeepAlive: true
  UseDNS: false
  VersionAddendum: 'none'
  X11Forwarding: false
...
{% endraw %}
{% endhighlight %}
<!-- markdownlint-enable MD003 MD022 -->

:information_source: As a side note: Yes, the configuration above is
[Federal Information Processing Standards (FIPS)](https://en.wikipedia.org/wiki/Federal_Information_Processing_Standards) compliant :rofl:.

Finally, there are also `host_vars` for my role `file_deployment`. As an example you'll find below the configuration for one of the hosts:

```yaml
---
fd_files:
  - src: 'sshd_motd'
    dest: '/etc/motd'
    owner: 'root'
    group: 'root'
    mode: '0644'

  - src: 'sshd_issue.net'
    dest: '/etc/issue.net'
    owner: 'root'
    group: 'root'
    mode: '0644'
...
```

The corresponding files are stored in `playbooks/files`.

*Locally installed* I have my custom role as well as the collection `redhat.rhel_system_roles`:

```shell
$ ansible-galaxy role list
# /home/steffen/.ansible/roles
- sscheib.filebeat, (unknown version)
- ansible-timesync, (unknown version)
- willshersystems.sshd, v0.17.0
- ansible-role-logrotate, (unknown version)
- ansible-role-rsyslog, (unknown version)
- ansible-sshd, (unknown version)
- zabbix_add_host, v1.0.6
- satellite_proxmox_compute_ressource, v1.0
- sscheib.openwrt_dropbear, (unknown version)
- openwrt_dropbear, (unknown version)
- satellite_create_host, v1.0.2
- satellite_publish_promote_content_views, v1.0.2
- satellite_prepare_installation, v1.0.3
- satellite_template_synchronization, v1.0.2
- satellite_global_parameters, v1.0.1
- file_deployment, v1.0.1
- rhel_iso_kickstart, v2.0.6
- user_deployment, v1.0.5
- package_installation, v1.0.1
- service_management, v1.0.2
- generate_ssl_key_pairs, v1.1.5
- register_to_satellite, v1.0.4
# /usr/share/ansible/roles
# /etc/ansible/roles

$ ansible-galaxy collection list

# /home/steffen/.ansible/collections/ansible_collections
Collection                  Version   
--------------------------- ----------
ansible.controller          4.3.0     
ansible.netcommon           5.3.0     
ansible.posix               1.5.4     
ansible.tower               3.8.6     
ansible.utils               2.10.3    
ansible.windows             1.11.0    
community.crypto            2.16.2    
community.docker            3.3.2     
community.general           7.3.0     
community.mysql             3.4.0     
community.postgresql        2.2.0     
community.routeros          2.7.0     
community.zabbix            1.9.1     
containers.podman           1.10.1    
infra.ah_configuration      2.0.2     
kubernetes.core             2.3.2     
openstack.cloud             2.0.0     
redhat.openshift            2.2.0     
redhat.rhel_system_roles    1.22.0    
redhat.satellite            3.15.0    
redhat.satellite_operations 2.1.0     
sscheib.insights            0.0.2     
theforeman.foreman          3.11.0-dev
zabbix.zabbix               1.2.2 
```

That should do it, right? :thinking:

Let's see what happens:

```shell
$ ansible-navigator run -m stdout playbooks/01e_sshd.yml --limit satellite.office.int.scheib.me
ERROR! the role 'file_deployment' was not found in /home/steffen/sources/ansible-project-satellite/playbooks/roles:/home/runner/.ansible/roles:/usr/share/ansible/roles:/etc/ansible/roles:/home/steffen/sources/ansible-project-satellite/playbooks

The error appears to be in '/home/steffen/sources/ansible-project-satellite/playbooks/01e_sshd.yml': line 7, column 7, but may
be elsewhere in the file depending on the exact syntax problem.

The offending line appears to be:

  roles:
    - role: 'file_deployment'
      ^ here
Please review the log for errors.
```

That's certainly correct. The role is not part of my project directory, nor is it part of the EE. Let's see if we can *somehow* make it work *without* building a custom EE
that contains the role.

Let's adjust the `ansible-navigator.yml` configuration a little:

```yaml
---
ansible-navigator:
[..]
  execution-environment:
    container-engine: 'podman'
    enabled: true
    image: 'ee-supported-rhel8'
    pull:
      policy: 'missing'
    volume-mounts:
      - src: '~/.ansible/roles/'
        dest: '/home/runner/.ansible/roles'
        options: 'z'
[..]
...
```

We just specified a [`volume mount`](https://docs.podman.io/en/v4.4/markdown/podman-volume-mount.1.html), which is a container feature to mount a local directory into the
container's filesystem.

:information_source: The above description of a `volume mount` is overly simplified. If you'd like to know more about `volume mounts`, please consult the
[documentation](https://docs.podman.io/en/latest/markdown/podman-volume-mount.1.html) of `podman`.

Rerunning the playbook seems to work now:

```plaintext
$ ansible-navigator run -m stdout playbooks/01e_sshd.yml --limit satellite.office.int.scheib.me

PLAY [Configure SSHD] ****************************************************************************************************************************************************************

TASK [file_deployment : Include tasks to ensure all variables are defined properly] **************************************************************************************************
included: /home/runner/.ansible/roles/file_deployment/tasks/assert.yml for satellite.office.int.scheib.me

TASK [file_deployment : Ensure mandatory variables, as  well as variables, which have a default value, are set (boolean)] ************************************************************
ok: [satellite.office.int.scheib.me] => (item=variable: _fd_quiet_assert) => {
    "__t_var": "_fd_quiet_assert",
    "ansible_loop_var": "__t_var",
    "changed": false,
    "msg": "Variable '_fd_quiet_assert' defined properly - value: 'False'"
}

TASK [file_deployment : Ensure _fd_packages is defined properly] *********************************************************************************************************************
ok: [satellite.office.int.scheib.me] => {
    "changed": false,
    "msg": "Files are defined correctly"
}

TASK [file_deployment : Include tasks to deploy files] *******************************************************************************************************************************
included: /home/runner/.ansible/roles/file_deployment/tasks/file_deployment.yml for satellite.office.int.scheib.me

TASK [file_deployment : file_deployment | Deploy files] ******************************************************************************************************************************
ok: [satellite.office.int.scheib.me] => (item=/etc/motd)
ok: [satellite.office.int.scheib.me] => (item=/etc/issue.net)

TASK [redhat.rhel_system_roles.sshd : Invoke the role, if enabled] *******************************************************************************************************************
included: /usr/share/ansible/collections/ansible_collections/redhat/rhel_system_roles/roles/sshd/tasks/sshd.yml for satellite.office.int.scheib.me
[..]
```

Perfect! Right? :thinking:

Well, yes, but no :rofl:. Here is the thing: If you'd use a customized EE that contains itself also roles, the above `volume mount` would effectively overwrite the directory
in that sense, that the `volume mount` would overlay the existing roles.

To accommodate for that scenario we have two options:

1. Specify a different roles directory using an Ansible configuration option:

    [`roles_path`](https://docs.ansible.com/ansible/latest/reference_appendices/config.html#default-roles-path). We could also specify it via the environment variable
    `ANSIBLE_ROLES_PATH`

1. Mount our local roles directory to a different directory that is checked by Ansible by default

    If we recall the error from above, where it couldn't find the role, there was an important piece of information in the error message - the search paths for Ansible roles:

```plaintext
ERROR! the role 'file_deployment' was not found in /home/steffen/sources/ansible-project-satellite/playbooks/roles:/home/runner/.ansible/roles:/usr/share/ansible/roles:/etc/ansible/roles:/home/steffen/sources/ansible-project-satellite/playbooks
```

So, Ansible will look at:

- `/home/steffen/sources/ansible-project-satellite/playbooks/roles`
- `/home/runner/.ansible/roles`
- `/usr/share/ansible/roles:/etc/ansible/roles`
- `/home/steffen/sources/ansible-project-satellite/playbooks`

We could therefore modify the volume mount to use either `/home/runner/.ansible/roles` or `/usr/share/ansible/roles:/etc/ansible/roles`.

Let's take the safer approach by adjusting the `roles_path`.

Do you recall that I specified the `ansible.cfg` in the `ansible-navigator.yml` file? We can take advantage of that!

We first adjust our `ansible-navigator.yml` to mount the roles somewhere else:

```yaml
---
ansible-navigator:
[..]
  execution-environment:
    container-engine: 'podman'
    enabled: true
    image: 'ee-supported-rhel8'
    pull:
      policy: 'missing'
    volume-mounts:
      - src: '~/.ansible/roles/'
        dest: '/home/runner/custom_roles/'
        options: 'z'
[..]
...
```

Finally, we add the following to our `ansible.cfg`:

```ini
[defaults]
roles_path          = $HOME/.ansible/roles:/usr/share/ansible/roles:/etc/ansible/roles:$HOME/custom_roles
```

Let us check whether everything works as intended:

```plaintext
ansible-navigator run -m stdout playbooks/01e_sshd.yml --limit satellite.office.int.scheib.me

PLAY [Configure SSHD] ****************************************************************************************************************************************************************

TASK [file_deployment : Include tasks to ensure all variables are defined properly] **************************************************************************************************
included: /home/runner/custom_roles/file_deployment/tasks/assert.yml for satellite.office.int.scheib.me

TASK [file_deployment : Ensure mandatory variables, as  well as variables, which have a default value, are set (boolean)] ************************************************************
ok: [satellite.office.int.scheib.me] => (item=variable: _fd_quiet_assert) => {
    "__t_var": "_fd_quiet_assert",
    "ansible_loop_var": "__t_var",
    "changed": false,
    "msg": "Variable '_fd_quiet_assert' defined properly - value: 'False'"
}

TASK [file_deployment : Ensure _fd_packages is defined properly] *********************************************************************************************************************
ok: [satellite.office.int.scheib.me] => {
    "changed": false,
    "msg": "Files are defined correctly"
}

TASK [file_deployment : Include tasks to deploy files] *******************************************************************************************************************************
included: /home/runner/custom_roles/file_deployment/tasks/file_deployment.yml for satellite.office.int.scheib.me

TASK [file_deployment : file_deployment | Deploy files] ******************************************************************************************************************************
ok: [satellite.office.int.scheib.me] => (item=/etc/motd)
[..]
```

Awesome :sunglasses:!

But wait a minute - why does it know about my files in `playbooks/files` and even about the `host_vars` and all that?

`ansible-navigator` mounts the current directory ('project directory') automatically inside the EE. This is why everything is working
[auto-magically](https://ansible.readthedocs.io/projects/navigator/faq/#where-should-the-ansiblecfg-file-go-when-using-an-execution-environment) :rocket:.

:warning: It is great to play around like that on your development machine, but this is not something that should be used in production. **Instead building custom EEs is the
way to go**. This ensures portability and eliminates the typical problems of ".. but it works on my machine!". I just wanted to show case what's possible with
`ansible-navigator` :sweat_smile:

## Building custom EEs with `ansible-builder`

Now that we learned a ton of stuff about `ansible-navigator` and its capabilities, let's take a closer look at `ansible-builder`.

`ansible-builder` is essentially a wrapper that calls `podman` to create container images for running Ansible content - the EEs, we talked about. While using existing EEs
gives you a quick start, there will be a point, where the existing EEs just don't fit your needs (anymore). Here is where `ansible-builder` comes in.

### Installing `ansible-builder`

There are multiple ways you can install `ansible-builder`, which are detailed in the
[installation documentation](https://ansible.readthedocs.io/projects/builder/en/latest/installation/).

To install a stable version of `ansible-builder` on RHEL (I'll refer to it as *downstream*), you need access to one of the Red Hat Ansible Automation Platform
repositories, for instance:

- Red Hat Ansible Automation Platform 2.4 for RHEL 8 x86_64 (RPMs), repository ID: `ansible-automation-platform-2.4-for-rhel-8-x86_64-rpms`
- Red Hat Ansible Automation Platform 2.4 for RHEL 9 x86_64 (RPMs), repository ID: `ansible-automation-platform-2.4-for-rhel-9-x86_64-rpms`

:information_source: Once again the hint that the above repositories require you to be a [Red Hat subscriber](#existing-ees-an-overview).

Simply enable the repository you've chosen using `subscription-manager`:

<!-- markdownlint-disable MD014 -->
```shell
$ sudo subscription-manager repos --enable ansible-automation-platform-2.4-for-rhel-8-x86_64-rpms
```
<!-- markdownlint-enable MD014 -->

Then install `ansible-navigator` using `dnf` or `yum`:

<!-- markdownlint-disable MD014 -->
```shell
$ sudo dnf install ansible-navigator
```
<!-- markdownlint-enable MD014 -->

If you'd rather like to use the latest available upstream version, you can do so with [`pip`](https://pypi.org/project/pip/):

<!-- markdownlint-disable MD014 -->
```shell
$ sudo dnf install python3-pip
$ python3 -m pip install ansible-builder --user
```
<!-- markdownlint-enable MD014 -->

For `ansible-builder` to be able to build EEs, you either need [`podman`](https://podman.io/) or [`docker`](https://www.docker.com/) on your system, as we are going to
build containers and naturally need *something* to handle them :slightly_smiling_face:. I'll use `podman` for the remainder of this post, but all commands I am going to use
with `podman` are working exactly the same when replacing `podman` with `docker`.

After all, we really only run **one** command with `podman`. The remainder will be handled by `ansible-builder` :sunglasses:.

:information_source: The upstream variant of `ansible-builder` is not exclusively available on RHEL; You can install it on a few more operating systems. Please refer to the
[installation documentation](https://ansible.readthedocs.io/projects/builder/en/latest/installation/) to get an overview of all supported operating systems.

### Getting started with `ansible-builder`

First off: I am going to talk about `ansible-builder` v3 which was released by
Red Hat with the [Ansible Automation Platform 2.4](https://access.redhat.com/support/policy/updates/ansible-automation-platform). With v3 of `ansible-builder` a new, *way* more
flexible format of writing EE definitions has been introduced. Please upgrade to `ansible-builder` v3+ if you haven't yet. It makes a **huge** difference.

Having said that, let's first look at the help output of `ansible-builder`:

```shell
$ ansible-builder --help
usage: ansible-builder [-h] [--version] {create,build,introspect} ...

Tooling to help build container images for running Ansible content. Get started by looking at the help text for one of the subcommands.

positional arguments:
  {create,build,introspect}
                        The command to invoke.
    create              Creates a build context, which can be used by podman to build an image.
    build               Builds a container image.
    introspect          Introspects collections in folder.

optional arguments:
  -h, --help            show this help message and exit
  --version             Print ansible-builder version and exit.
```

Okay, that's a *lot less* possibilities than with `ansible-navigator`. Don't let yourself fool. This is all you **need**.

Actually, we really only need `create` and `build` as commands.

`introspect` is used by `ansible-builder` itself to figure out dependencies of collections. That is something
we do not need at the moment, but surely is helpful if you are interested in finding out why a specific system or Python package has been installed.

Let me quickly show you what it looks like when I let `ansible-builder introspect` check my installed collections:

```shell
$ ansible-builder introspect ~/.ansible/collections/
# Dependency data for /home/steffen/.ansible/collections/
---
python:
  ansible.controller:
  - 'pytz  # for schedule_rrule lookup plugin'
  - 'python-dateutil>=2.7.0  # schedule_rrule'
  - 'awxkit  # For import and export modules'
  ansible.netcommon:
  - ansible-pylibssh >= 0.2.0
  - jxmlease
  - ncclient
  - netaddr
  - paramiko
  - xmltodict
  - grpcio
  - protobuf
  ansible.tower:
  - 'pytz  # for tower_schedule_rrule lookup plugin'
  - 'python-dateutil>=2.7.0  # tower_schedule_rrule'
  - 'awxkit  # For import and export modules'
  ansible.utils:
  - jsonschema
  - textfsm
  - ttp
  - xmltodict
  - netaddr
  community.crypto:
  - PyYAML
  community.docker:
  - docker
  - requests
  - paramiko
  community.routeros:
  - librouteros
  kubernetes.core:
  - kubernetes>=12.0.0
  - requests-oauthlib
  - jsonpatch
  openstack.cloud:
  - openstacksdk>=1.0.0
  redhat.openshift:
  - kubernetes>=12.0.0
  - requests-oauthlib
  redhat.satellite:
  - requests>=2.4.2
  - ipaddress; python_version < '3.3'
  - PyYAML
  sscheib.insights:
  - requests
  theforeman.foreman:
  - requests>=2.4.2
  - ipaddress; python_version < '3.3'
  - PyYAML
  zabbix.zabbix:
  - netaddr>=0.8
  - Jinja2>=3.1.2

system:
  ansible.controller:
  - python38-pytz [platform:centos-8 platform:rhel-8]
  - python38-requests [platform:centos-8 platform:rhel-8]
  - python38-pyyaml [platform:centos-8 platform:rhel-8]
  ansible.netcommon:
  - gcc-c++ [doc test platform:rpm]
  - libyaml-devel [test platform:rpm]
  - libyaml-dev [test platform:dpkg]
  - python3-devel [test platform:rpm]
  - python3 [test platform:rpm]
  - gcc [compile platform:rpm]
  - libssh-dev [compile platform:dpkg]
  - libssh-devel [compile platform:rpm]
  - python3-Cython [compile platform:fedora-35 platform:rhel-9]
  - python3-six [platform:centos-9 platform:rhel-9]
  - python39-six [platform:centos-8 platform:rhel-8]
  - python3-lxml [platform:centos-9 platform:rhel-9]
  - python39-lxml [platform:centos-8 platform:rhel-8]
  - findutils [compile platform:centos-8 platform:rhel-8]
  - gcc [compile platform:centos-8 platform:rhel-8]
  - make [compile platform:centos-8 platform:rhel-8]
  - python3-devel [compile platform:centos-9 platform:rhel-9]
  - python39-devel [compile platform:centos-8 platform:rhel-8]
  - python3-cffi [platform:centos-9 platform:rhel-9]
  - python39-cffi [platform:centos-8 platform:rhel-8]
  - python3-cryptography [platform:centos-9 platform:rhel-9]
  - python39-cryptography [platform:centos-8 platform:rhel-8]
  - python3-pycparser [platform:centos-9 platform:rhel-9]
  - python39-pycparser [platform:centos-8 platform:rhel-8]
  ansible.posix:
  - rsync [platform:redhat]
  ansible.utils:
  - gcc-c++ [doc test platform:rpm]
  - python3-devel [test platform:rpm]
  - python3 [test platform:rpm]
  community.crypto:
  - cryptsetup [platform:dpkg]
  - cryptsetup [platform:rpm]
  - openssh-client [platform:dpkg]
  - openssh-clients [platform:rpm]
  - openssl [platform:dpkg]
  - openssl [platform:rpm]
  - python3-cryptography [platform:dpkg]
  - python3-cryptography [platform:rpm]
  - python3-openssl [platform:dpkg]
  - python3-pyOpenSSL [platform:rpm !platform:rhel !platform:centos !platform:rocky]
  - python3-pyOpenSSL [platform:rhel-8]
  - python3-pyOpenSSL [platform:rhel !platform:rhel-6 !platform:rhel-7 !platform:rhel-8
    epel]
  - python3-pyOpenSSL [platform:centos-8]
  - python3-pyOpenSSL [platform:centos !platform:centos-6 !platform:centos-7 !platform:centos-8
    epel]
  - python3-pyOpenSSL [platform:rocky-8]
  - python3-pyOpenSSL [platform:rocky !platform:rocky-8 epel]
  infra.ah_configuration:
  - python >=3.5
  kubernetes.core:
  - kubernetes-client [platform:fedora]
  - openshift-clients [platform:rhel-8]
  openstack.cloud:
  - gcc [compile platform:centos-8 platform:rhel-8]
  - python38-cryptography [platform:centos-8 platform:rhel-8]
  - python38-devel [compile platform:centos-8 platform:rhel-8]
  - python38-requests [platform:centos-8 platform:rhel-8]
  redhat.rhel_system_roles:
  - openssl
  - dnf
  redhat.satellite:
  - python3-rpm [(platform:redhat platform:base-py3)]
  - rpm-python [(platform:redhat platform:base-py2)]
  - python38-requests [platform:centos-8 platform:rhel-8]
  sscheib.insights:
  - python38-requests [platform:centos-8 platform:rhel-8]
  theforeman.foreman:
  - python3-rpm [(platform:redhat platform:base-py3)]
  - rpm-python [(platform:redhat platform:base-py2)]
  - python38-requests [platform:centos-8 platform:rhel-8]
```

That's quite a bunch of system and Python packages :slightly_smiling_face:.

Again, `ansible-builder introspect` is not particularly useful, unless you debug an issue with dependencies. I just wanted you to know, that *if* there is an issue with
dependencies while building EEs, this command can be pretty handy.

Now that we have the the `introspect` out of the way, let's look at the two remaining sub-commands:

- `create`
- `build`

In a nutshell, `ansible-builder create` creates a so-called *build context* for you - remember, we want to build containers. Building containers is done via a
[`Containerfile`](https://www.mankier.com/5/Containerfile) (for `podman`) or a [`Dockerfile`](https://docs.docker.com/reference/dockerfile/) (for `docker`) which comes in a
specific format. `Containerfile` and `Dockerfile` definitions are *largely* interchangeable (there *were* differences in the past, but I don't know if it is the same today).

:information_source: Further in this post I'll use `Containerfile` as reference to make things easier. Just know that *usually* `Containerfile` and `Dockerfile` are
interchangeable.

`ansible-builder create` essentially translate an [EE definition](https://ansible.readthedocs.io/projects/builder/en/stable/definition/) into container build steps. That is
useful if you ever need to modify the Containerfile to your *very*, *very* specific needs before building the container image. *Usually*, `ansible-builder` with the new
Version 3 of the EE definition is enough for the majority of use cases.

Please bear with me a minute, I'll explain shortly what an EE definition looks like.

`ansible-builder build` on the other hand, goes one step further than `ansible-builder create`. Once the build context has been created, it will also run `podman` under the
hood to actually create and tag your image.

Again, the above explanation is very high level, but we'll go into the details later on.

### Execution Environment definition

Before we dive into it: Just a reminder, I'll be talking about *Version 3* (`ansible-builder` version > 3 is required) of the EE definition. Version 1 has limitations that have
largely been resolved by Version 3. If you have been building Version 1 EEs previously, please consider migrating to Version 3, as it eases the process a lot - especially for
complex EEs.

Let's dive into it - finally we get to define something :sunglasses:.

Execution Environment definitions essentially describe *what* you'd like to have within your EE. Typically, the *what* is a combination of these:

1. `ansible-core` and which version of it
1. `ansible-runner` and which version of it
1. Ansible collections
1. Ansible roles
1. Python packages, both as dependencies of an Ansible collection and as a "user-supplied" package
1. System packages, both as dependencies of an Ansible collection and as a "user-supplied" package
1. Additional system configurations, modifications and files, such as certificate authorities of your company or specific proxy settings

:information_source: What I mean by a "user-supplied" package (for both system packages and Python packages) is that you specifically instructed `ansible-builder` to install
these packages during the EE build.

EE definitions are written entirely in [`YAML`](https://yaml.org/) and should therefore be no trouble writing for anyone who as previously worked with `YAML` - such as
Ansible users :smile:.

Let's imagine a scenario:

Your Satellite team approached you, that they'd like to automate Red Hat Satellite and therefore need the latest versions of the collections
[`redhat.satellite`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/) and
[`redhat.satellite_operations`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite_operations/) inside the EE.
They also want to make use of the latest available `ansible-core` version, which is at the time of this writing 2.16.

That's pretty straight-forward. We have the choice to either use UBI 8 or UBI 9, and we decide to go with UBI 8.

So what we need inside the EE:

1. `ansible-core` in version 2.16
1. `ansible-runner` in version 2.16
1. [`redhat.satellite`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/) in the latest version
1. [`redhat.satellite_operations`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite_operations/) in the latest version
1. The EE should be based on UBI 8

I know, you have waited for a complex EE definition, but you'll be shocked how easy it is to define the above requirements as EE. Have a look at the EE definition
below:

```yaml
---
version: 3

images:
  base_image:
    name: 'registry.redhat.io/ansible-automation-platform/ee-minimal-rhel8:2.16'

dependencies:
  galaxy:
    collections:
      - name: 'redhat.satellite'
      - name: 'redhat.satellite_operations'

options:
  package_manager_path: '/usr/bin/microdnf'

additional_build_files:
    - src: 'ansible.cfg'
      dest: 'configs/'

additional_build_steps:
  prepend_galaxy:
    - 'COPY _build/configs/ansible.cfg /home/runner/.ansible.cfg'
...
```

That's it. Really. Before we dive into what all the options do, let me first explain what `ansible-builder` command to run and check whether all our requirements are met.

I put the above definition in a file called `execution-environment.yml`: That is the default file `ansible-builder` will try to open if you don't pass a specific EE definitions
file using the `--file` or `-f` switches.

Further, I increased the verbosity (`--verbosity` or `-v` with either 0, 1, 2 or 3 as value) to actually see what's happening during the build and lastly, I
[tagged](https://docs.podman.io/en/latest/markdown/podman-tag.1.html) (essentially assigned a name and a version to my image) my image as `ee-satellite-rhel-8:latest` using the
`--tag` or `-t` switches.

The corresponding `ansible-builder build` command is the following:

```shell
ansible-builder build --tag ee-satellite-rhel-8 --verbosity 3
```

Pretty easy, right? Once you run that command you'll see something like this as the last messages of the `ansible-builder build` command:

```plaintext
[4/4] COMMIT ee-satellite-rhel-8
--> 0ac2e9ef7216
Successfully tagged localhost/ee-satellite-rhel-8:latest
0ac2e9ef72169c7104b40e04bde34480b107b46efb5469bbb36ee883fe232e1e
```

:information_source: Please note, the above checksum you see, is the checksum of the container image and yours definitively *differs*.

You can check all your locally existing container images with `ansible-navigator images` using the TUI interface:

```plaintext
   Image                                                                                Tag          Execution environment                  Created                    Size
 0│ee-minimal-rhel8                                                                     2.16         True                                   3 weeks ago                345 MB
 1│ee-satellite-rhel-8                                                                  latest       True                                   17 minutes ago             445 MB
```

Or if you prefer, using `podman image list`:

```shell
$ podman images
REPOSITORY                                                            TAG         IMAGE ID      CREATED         SIZE
localhost/ee-satellite-rhel-8                                         latest      d9b8762094e4  15 minutes ago  445 MB
registry.redhat.io/ansible-automation-platform/ee-minimal-rhel8       2.16        74f040829a5c  3 weeks ago     345 MB
```

Now, let's check if our EE `localhost/ee-satellite-rhel-8` contains `ansible-core 2.16` and the collections `redhat.satellite` and `redhat.satellite_operations`. We are going
to use `ansible-navigator collections --execution-environment-image localhost/ee-satellite-rhel-8:latest --pull-policy never` for that:

```plaintext
  Name                                 Version    Shadowed    Type         Path
0│ansible.builtin                      2.16.3     False       contained    /usr/local/lib/python3.11/site-packages/ansible
1│redhat.satellite                     4.0.0      False       contained    /usr/share/ansible/collections/ansible_collections/redhat/satellite
2│redhat.satellite_operations          2.1.0      False       contained    /usr/share/ansible/collections/ansible_collections/redhat/satellite_operations
```

:information_source: Since we don't have a container registry on our host (which is perfectly normal!) we need to use `--pull-policy` or `--pp` with the value `never`, which
instructs `ansible-navigator` to not *pull* the EE, which would otherwise fail. This is *only* for EEs that you've built locally or pulled already from a container registry
and don't want to update it every time you run `ansible-navigator`. This helps also speed up the ramp-up time of `ansible-navigator` :sunglasses:.

:information_source: `--execution-environment-image` can be shortened with `--eei`. This will give you back plenty of time if you are such a bad typer as I am and type
`--execution-environment-image` wrong at least twice every time I try to use it:rofl:.

Okay great, we have the `ansible-core` version as well as the Ansible collections `redhat.satellite` and `redhat.satellite_operations`.

Easy, wasn't it? :slightly_smiling_face:

Let's break down this easy EE definition:

1. [`version` option](https://ansible.readthedocs.io/projects/builder/en/stable/definition/#version)

   With `version` we specify the EE definition version to use. We want this to be `3`. Version `1` looks a little different and - as said - has quite some limitations.
   Additionally version 3 is **required** when using `ansible-builder` 3.x (which we do).

2. [`images` option](https://ansible.readthedocs.io/projects/builder/en/stable/definition/#images)

   `images` is a dictionary, and the only supported attribute within `images` is `base_image`, which itself *must* have a `name` attribute which specifies the base image to use.

    Remember we talked about using existing EEs? We use these existing EEs now to add "on top" of the existing EEs our own content.

3. [`dependencies` option](https://ansible.readthedocs.io/projects/builder/en/stable/definition/#dependencies)

    I only *briefly* explain these so-called "in-line dependencies". We'll later use separate files for each of the requirement types - you understand why later on.

    Alright, so `dependencies` can have multiple attributes, but we *only* specified the `galaxy` attribute, which itself can either have a `roles` attribute or a `collections`
    attribute. The latter of which we used. In there we specify the roles and collections we want to be included in the EE.

    Again, I leave the Python package and system package dependencies out, as well as more complex definitions of collections and roles (with version and source) as we'll later
    have it in separate files.

4. [`options` option](https://ansible.readthedocs.io/projects/builder/en/stable/definition/#options)

    The `options` option can have a *bunch* of attributes, but in the context of this minimal EE definition, we simply need to set the `package_manager_path`, which defaults to
    `/usr/bin/dnf`, which is not present. Instead EEs make use of `/usr/bin/microdnf` a slimmed down version of `dnf`.

5. [`additional_build_steps` option](https://ansible.readthedocs.io/projects/builder/en/stable/definition/#additional-build-steps)

    The `additional_build_steps` option is probably **the most important** option when it comes to building complex EEs. In our example we copied the `ansible.cfg` to the
    `/home/runner` directory (which is home directory of the `ansible-runner` user) so that this user is able to reach for instance
    [Red Hat's Automation Hub](https://console.redhat.com) or your Private Automation Hub.

    We also specified a specific attribute: `prepend_galaxy`, this essentially tells `ansible-builder` to put the build steps in this section **before** the installation
    of Ansible content.

    :information_source: We'll go into much more details later on this topic.

6. [`additional_build_files` option](https://ansible.readthedocs.io/projects/builder/en/stable/definition/#additional-build-files)

    With `additional_build_files` we can copy files we have next to our `execution-environment.yml` into the build context directory. From there we can add it to the EE.
    This is useful for providing the `ansible.cfg` to the EE, but will also come in handy if we need to add other files - such as certificate authority certificates - to
    the EE.

Before we move on, let's look at what is in the `context` directory next to your `execution-environment.yml`:

```shell
$ tree context/
context/
├── _build
│   ├── configs
│   │   └── ansible.cfg
│   ├── requirements.yml
│   └── scripts
│       ├── assemble
│       ├── check_ansible
│       ├── check_galaxy
│       ├── entrypoint
│       ├── install-from-bindep
│       └── introspect.py
└── Containerfile
```

So, the `context` directory contains the `_build` directory and a `Containerfile`. Let's have a glimpse at them :sunglasses::

1. `configs`

    We introduced this directory ourselves, as we specified to copy our `ansible.cfg` to `_build/configs`. This is our `ansible.cfg` we specified to copy in the EE definition.
    If you check the content of your `ansible.cfg` and the one that ended up in the EE, you'll find that they are the same.

    You don't have to specify that config sub-directory, `_build` is enough. I personally just find it cleaner that way, so I know what comes from me in terms of additional files.

2. `requirements.yml`

    Let's quickly look what's in the `requirements.yml`:

    ```shell
    $ cat context/_build/requirements.yml 
    collections:
    - name: redhat.satellite
    - name: redhat.satellite_operations 
    ```

    Does that look familiar to you? :slightly_smiling_face:

    These are the collection we specified in our EE definition. They are simply added to a `requirements.yml` by `ansible-builder` when they are specified in-line, as we did.
    [`requirements.yml`](https://docs.ansible.com/ansible/latest/galaxy/user_guide.html#installing-roles-and-collections-from-the-same-requirements-yml-file) should be known to
    most Ansible users already: This is the standard format of providing depending collections and roles and can be read by `ansible-galaxy collection install` or
    `ansible-galaxy role install` when you pass the `--requirements-file` or, more commonly, the `-r` switch to the `ansible-galaxy command` and point it to the
    file, e.g.: `ansible-galaxy collection install -r requirements.yml`.
    This is exactly what `ansible-builder` does during the `galaxy stage` (where roles and collections are installed)

3. `Containerfile`

    This file is essentially the "translation" of the EE definition to "container build language". Of course, certain things are pre-defined, but you'll find it contains the
    options we specified.

    Let's have a look:

    ```plaintext
    ARG EE_BASE_IMAGE="registry.redhat.io/ansible-automation-platform/ee-minimal-rhel8:2.16"
    ARG PYCMD="/usr/bin/python3"
    ARG PKGMGR_PRESERVE_CACHE=""
    ARG ANSIBLE_GALAXY_CLI_COLLECTION_OPTS=""
    ARG ANSIBLE_GALAXY_CLI_ROLE_OPTS=""
    ARG PKGMGR="/usr/bin/microdnf"

    # Base build stage
    FROM $EE_BASE_IMAGE as base
    USER root
    ARG EE_BASE_IMAGE
    ARG PYCMD
    ARG PKGMGR_PRESERVE_CACHE
    ARG ANSIBLE_GALAXY_CLI_COLLECTION_OPTS
    ARG ANSIBLE_GALAXY_CLI_ROLE_OPTS
    ARG PKGMGR

    RUN $PYCMD -m ensurepip
    COPY _build/scripts/ /output/scripts/
    COPY _build/scripts/entrypoint /opt/builder/bin/entrypoint

    # Galaxy build stage
    FROM base as galaxy
    ARG EE_BASE_IMAGE
    ARG PYCMD
    ARG PKGMGR_PRESERVE_CACHE
    ARG ANSIBLE_GALAXY_CLI_COLLECTION_OPTS
    ARG ANSIBLE_GALAXY_CLI_ROLE_OPTS
    ARG PKGMGR

    COPY _build/configs/ansible.cfg /home/runner/.ansible.cfg
    RUN /output/scripts/check_galaxy
    COPY _build /build
    WORKDIR /build

    RUN ansible-galaxy role install $ANSIBLE_GALAXY_CLI_ROLE_OPTS -r requirements.yml --roles-path "/usr/share/ansible/roles"
    RUN ANSIBLE_GALAXY_DISABLE_GPG_VERIFY=1 ansible-galaxy collection install $ANSIBLE_GALAXY_CLI_COLLECTION_OPTS -r requirements.yml --collections-path "/usr/share/ansible/collections"

    # Builder build stage
    FROM base as builder
    WORKDIR /build
    ARG EE_BASE_IMAGE
    ARG PYCMD
    ARG PKGMGR_PRESERVE_CACHE
    ARG ANSIBLE_GALAXY_CLI_COLLECTION_OPTS
    ARG ANSIBLE_GALAXY_CLI_ROLE_OPTS
    ARG PKGMGR

    RUN $PYCMD -m pip install --no-cache-dir bindep pyyaml requirements-parser

    COPY --from=galaxy /usr/share/ansible /usr/share/ansible

    RUN $PYCMD /output/scripts/introspect.py introspect --sanitize --write-bindep=/tmp/src/bindep.txt --write-pip=/tmp/src/requirements.txt
    RUN /output/scripts/assemble

    # Final build stage
    FROM base as final
    ARG EE_BASE_IMAGE
    ARG PYCMD
    ARG PKGMGR_PRESERVE_CACHE
    ARG ANSIBLE_GALAXY_CLI_COLLECTION_OPTS
    ARG ANSIBLE_GALAXY_CLI_ROLE_OPTS
    ARG PKGMGR

    RUN /output/scripts/check_ansible $PYCMD

    COPY --from=galaxy /usr/share/ansible /usr/share/ansible

    COPY --from=builder /output/ /output/
    RUN /output/scripts/install-from-bindep && rm -rf /output/wheels
    RUN chmod ug+rw /etc/passwd
    RUN mkdir -p /runner && chgrp 0 /runner && chmod -R ug+rwx /runner
    WORKDIR /runner
    RUN $PYCMD -m pip install --no-cache-dir 'dumb-init==1.2.5'
    RUN rm -rf /output
    LABEL ansible-execution-environment=true
    USER 1000
    ENTRYPOINT ["/opt/builder/bin/entrypoint", "dumb-init"]
    CMD ["bash"]
    ```

    I know, if you are not familiar with building containers, most of the file looks like gibberish to you. Don't worry, with Version 3 of the EE definition, you don't need to
    touch the `Containerfile` itself - *usually*.
    There might be *very, very* specific use-case where `ansible-builder` cannot fulfill your needs, but I personally haven't encountered it (yet).

    Nevertheless, what you **need** to understand are the various stages the build passes through. This is because you can inject commands to each of those stages
    (at the very beginning and the very end of each section, essentially), and this is *exactly* what you need to realize complex EEs with proxies, certificates and so on.
    We'll cover these build stages in the next section.

### Execution Environment stages

During an EE build, we actually build *multiple* intermediate images. We call these intermediate images **stages**.

If we take the `Containerfile` from the previous section, we can see the following statements in it:

```plaintext
[..]
FROM $EE_BASE_IMAGE as base  # first image
[..]
FROM base as galaxy  # second image
[..]
FROM base as builder  # third image
[..]
FROM base as final  # fourth image
[..]
```

With these `FROM X as Y` statements we start building *a new image*.

First, we start with the `base` stage. Next, is the `galaxy` stage, followed by the `builder` stage and lastly the `final` stage.
For all these stages, you can prepend or append container build commands yourself - this makes it highly flexible.

In below table you'll see which `additional_build_steps` attribute needs to be used for which stage:

| stage                 | attribute                                    | position                |
| :-------------------- | :------------------------------------------- | :---------------------- |
| `base`                | `prepend_base`                               | beginning of `base`     |
| `base`                | `append_base`                                | end of `base`           |
| `galaxy`              | `prepend_galaxy`                             | beginning of `galaxy`   |
| `galaxy`              | `append_galaxy`                              | end of `galaxy`         |
| `builder`             | `prepend_builder`                            | beginning of `builder`  |
| `builder`             | `append_builder`                             | end of `builder`        |
| `final`               | `prepend_final`                              | beginning of `final`    |
| `final`               | `append_final`                               | end of `final`          |

To be able to use these stages effectively, we'll need to understand what each of the sections does - let's have a look.

#### `base` stage

The `base` stages begins by ensuring that `pip` is available (`RUN $PYCMD -m ensurepip`). This is important, as we'll use the intermediate image `base` as our base for the
other intermediate images - hence the name `base`.

Next, we copy the scripts we have in `_build/scripts` to `/output/scripts` and copy the `entrypoint` script to its destination at `/opt/builder/bin/entrypoint`. The
`entrypoint` script will later on be the entry point for the `final` image.

The `scripts` contain scripts which we need during the build of our intermediate containers. Usually, you don't need to look at them and understand what's happening.
For troubleshooting it's surely worthwhile to have a look :slightly_smiling_face:.

#### `galaxy` stage

Within the `galaxy` stage we'll only install all Ansible collections and roles that are asked to be installed. That's it :slightly_smiling_face:.

#### `builder` stage

The builder stage first copies what we've installed in the `galaxy` stage to `/usr/share/ansible` and then we `introspect` the content of it. Which means in other words,
we extract the dependencies that each collection requires and write them into `/tmp/src/bindep.txt` for system package dependencies or `/tmp/src/requirements.txt`
for Python packages.

:information_source:
Remember the `ansible-builder introspect` a few sections earlier? That's essentially the same.

Finally we run a mysterious script `/output/scripts/assemble`. That's one of the scripts that are in the `context/scripts` directory when you run `ansible-builder create` only.

What this script essentially does is resolving Python package dependencies and installing them to `/output`, which we need for the next stage. Additionally system packages are
installed and a list of them is collected and written for the next stage into `/output`.

We also cache what we've downloaded for Python packages, so the next stage progresses quicker :sunglasses:.

#### `final` stage

This is - who would have guessed it - the final image. Essentially we combine everything now.

First we copy from the `galaxy` stage the Ansible collections and roles to `/usr/share/ansible/`, then we copy the result of the `builder` stage from `/output` to
`/output` and re-install the cached Python packages and as well the system package dependencies.

### Complex execution environments

With all the knowledge from the previous chapters, we can finally start creating complex execution environments :sunglasses:.

I'll make up a completely random scenario where we install a bunch of collections, Python and system packages. In my imaginary scenario, we'll need to set a proxy,
a custom Python package repository and we also need to install custom certificate authority certificates. Further, we'll disable and enable repositories.

#### Specifying dependencies as files

A few sections earlier, we specified the Ansible collections we wanted to install via in-line dependencies. Above I said, we'll be using files.
The reason being, that - from my point of view - the more collections, Python and system packages you want to install, the harder to read the EE definition gets.

For this reason I've decided for myself to always use files to specify the dependencies - no matter how much I have.

We have three types of dependencies we can specify:

1. Ansible collections and roles
1. Python packages
1. System packages

They all have a very specific format; They are easy to remember formats, however. At least from my perspective.

#### Installing Ansible collections and roles

Defining Ansible collections and roles should be the easiest of all. The
[format](https://docs.ansible.com/ansible/latest/galaxy/user_guide.html#installing-roles-and-collections-from-the-same-requirements-yml-file) is probably known to most
Ansible users.

For this example, I've copied one of my own `requirements.yml` and placed it next to my `execution-environment.yml`. I use the below
`requirements.yml` for automating Satellite end-to-end. It contains a few collections, as well as some roles:

:information_source: If you don't specify a version for a collection or role, it'll default to use the latest possible version.

If you want reproducibility for your EEs (so that it builds the same *every time*, even months after you created the initial EE), you *must* specify fixed versions of
collections and roles. `>=` means "bigger than or equal to" (as seen in the code block below), which results in the installation of new versions of the
respective collections and roles if they are available.

This is only for the "user-specified" Ansible collections and roles. Once you install Ansible collections, they can come with their own dependencies, which you
cannot influence in that sense. So even if you specify fixed versions of your "user-supplied" Ansible collections, it doesn't mean that your EE is 100% reproducible.

```yaml
---
collections:
  - name: 'redhat.satellite'
    version: '>=3.10.0'

  - name: 'redhat.satellite_operations'
    version: '>=1.3.0'

  - name: 'redhat.rhel_system_roles'
    version: '>=1.21.1'

  - name: 'ansible.posix'
    version: '>=1.5.4'

  - name: 'community.crypto'
    version: '>=2.3.2'

  # NOTE: Temporarily disabling the official redhat.insights collection and adding
  #       a temporary publish of the insights collection within my own namespace
  #       which contains PR https://github.com/RedHatInsights/ansible-collections-insights/pull/90.
  #       This PR provides required functionality for the insights role (obfuscating)
  #       and I didn't want to work around this issue.
  #       Another PR was also cherry-picked to this collection to fix the Insights client
  #       registration (https://github.com/RedHatInsights/ansible-collections-insights/pull/89)
  #
  #  - name: 'redhat.insights'
  #    version: '>=1.2.0'

  - name: 'sscheib.insights'
    version: '0.0.2'

roles:
  - name: 'rhel_iso_kickstart'
    src: 'https://github.com/sscheib/ansible-role-rhel_iso_kickstart.git'
    scm: 'git'
    version: 'v2.0.6'

  - name: 'user_deployment'
    src: 'https://github.com/sscheib/ansible-role-user_deployment.git'
    scm: 'git'
    version: 'v1.0.5'

  - name: 'package_installation'
    src: 'https://github.com/sscheib/ansible-role-package_installation.git'
    scm: 'git'
    version: 'v1.0.1'

  - name: 'service_management'
    src: 'https://github.com/sscheib/ansible-role-service_management.git'
    scm: 'git'
    version: 'v1.0.2'

  - name: 'generate_ssl_key_pairs'
    src: 'https://github.com/sscheib/ansible-role-generate_ssl_key_pairs.git'
    scm: 'git'
    version: 'v1.1.5'

  - name: 'register_to_satellite'
    src: 'https://github.com/sscheib/ansible-role-register_to_satellite.git'
    scm: 'git'
    version: 'v1.0.4'

  - name: 'satellite_create_host'
    src: 'https://github.com/sscheib/ansible-role-satellite_create_host.git'
    scm: 'git'
    version: 'v1.0.2'

  - name: 'satellite_publish_promote_content_views'
    src: 'https://github.com/sscheib/ansible-role-satellite_publish_promote_content_views.git'
    scm: 'git'
    version: 'v1.0.2'

  - name: 'satellite_prepare_installation'
    src: 'https://github.com/sscheib/ansible-role-satellite_prepare_installation.git'
    scm: 'git'
    version: 'v1.0.3'

  - name: 'satellite_template_synchronization'
    src: 'https://github.com/sscheib/ansible-role-satellite_template_synchronization.git'
    scm: 'git'
    version: 'v1.0.2'

  - name: 'satellite_global_parameters'
    src: 'https://github.com/sscheib/ansible-role-satellite_global_parameters.git'
    scm: 'git'
    version: 'v1.0.1'

  - name: 'file_deployment'
    src: 'https://github.com/sscheib/ansible-role-file_deployment.git'
    scm: 'git'
    version: 'v1.0.1'
...
```

As you see, I am using both community and certified collections. The roles are all my own :slightly_smiling_face:.

To make `ansible-builder` aware of this file, we need to specify the following in our `execution-environment.yml`:

```yaml
[..]
dependencies:
  galaxy: 'requirements.yml'
[..]
```

#### Installing Python packages

Python package requirements follow also a very specific [format](https://pip.pypa.io/en/stable/reference/requirements-file-format/). Below I'll made up a random set
of Python packages to install in different versions.

:information_source: If you don't specify a version for a Python package, it'll default to use the latest possible version.

If you want reproducibility for your EEs (so that it builds the same *every time*, even months after you created the initial EE), you *must* specify fixed versions for
Python packages. `>=` means "bigger than or equal to" (as seen in the code block below), which results in the installation of new versions of the
respective Python packages if they are available and `pip` dependency resolution finds it will not cause dependency issues.

This is only for the "user-specified" Python packages. Once you install Ansible collections and Python packages, they come with their own dependencies, which you
cannot influence in that sense. So even if you specify fixed versions of your "user-supplied" Python packages, it doesn't mean that your EE is 100% reproducible.

```plaintext
pytz
python-dateutil>=2.7.0
awxkit
proxmoxer
ansible-pylibssh>=0.2.0
jxmlease
ncclient
netaddr
paramiko
xmltodict
grpcio
protobuf
jsonschema
textfsm
ttp
xmltodict
PyYAML
```

We'll place the above content into the file `requirements.txt`, next to our `execution-environment.yml`.

To make `ansible-builder` aware of this file, we need to specify the following in our `execution-environment.yml`:

```yaml
[..]
dependencies:
  python: 'requirements.txt'
[..]
```

#### Installing system packages

Last but not least, we have the system packages. Also these follow a specific [format](https://docs.opendev.org/opendev/bindep/latest/readme.html).

:information_source: If you don't specify a version for a system package, it'll default to use the latest possible version.

If you want reproducibility for your EEs (so that it builds the same *every time*, even months after you created the initial EE), you *must* specify a fixed version for
your system packages.

This is only for the "user-specified" system packages. Once you install Ansible collections, Python packages and system packages, they can come with their own
dependencies, which you cannot influence in that sense. So even if you specify fixed versions of your "user-supplied" system packages, it doesn't mean that your EE
is 100% reproducible.

I made up again a random list of system packages to install:

```plaintext
libyaml-devel [test platform:rpm]
libyaml-dev [test platform:dpkg]
python3-devel [test platform:rpm]
python3 [test platform:rpm]
gcc [compile platform:rpm]
libssh-dev [compile platform:dpkg]
libssh-devel [compile platform:rpm]
python3-Cython [compile platform:fedora-35 platform:rhel-9]
python3-six [platform:centos-9 platform:rhel-9]
python39-six [platform:centos-8 platform:rhel-8]
python3-lxml [platform:centos-9 platform:rhel-9]
python39-lxml [platform:centos-8 platform:rhel-8]
findutils [compile platform:centos-8 platform:rhel-8]
gcc [compile platform:centos-8 platform:rhel-8]
make [compile platform:centos-8 platform:rhel-8]
python3-devel [compile platform:centos-9 platform:rhel-9]
python39-devel [compile platform:centos-8 platform:rhel-8]
python3-cffi [platform:centos-9 platform:rhel-9]
jq
grep == 3.3
```

The format is the most complex of the three, and I encourage you to [read up on it](https://docs.opendev.org/opendev/bindep/latest/readme.html) to understand it. For basic
requirements, you can simply name the RPM you'd like to have installed - like I did with `jq` above.

I've also added an example for specifying a fixed version of the package `grep`.

We'll place the above content in the file `bindep.txt`, again next to our `execution-environment.yml`.

To make `ansible-builder` aware of this file, we need to specify the following in our `execution-environment.yml`:

```yaml
[..]
dependencies:
  system: 'bindep.txt'
[..]
```

#### Adding certificates to an EE

Adding certificates is essentially the same as with every RHEL-like system. In the case of EEs, we need to copy the certificates first into our `_build`
directory (as with the `ansible.cfg`).

Let's imagine we have a file called `ca.cert.pem` and another one `intermediate.cert.pem` next to our `execution-environments.yml` file.

To add these certificates to the EE, we first need to tell `ansible-builder` to copy these files to the `_build` directory:

```yaml
[..]
additional_build_files:
    - src: 'ca.cert.pem'
      dest: 'certs/'

    - src: 'intermediate.cert.pem'
      dest: 'certs/'
[..]
```

With the above, we'll copy both `ca.cert.pem` and `intermediate.cert.pem` to `_build/certs` inside the build context directory.

Next we need to copy these files actually to their correct destination. For that, we need to inject build steps into one of the stages we talked about.

Now often the question comes up: Which stage?

The answer is, as so often, it depends :rofl:.

If you need the certificates to validate your connection to your private automation hub, then you'll need them in the `galaxy` stage. Do you require certificate validation with
your proxy, and need to access the internet or some internal systems via a proxy, then you'll probably need the certificates in all stages (but the `base` stage, probably).

As you see, the answer really depends on your specific requirement.

Nevertheless, the procedure is always the same when you specify it in the EE definition:

```yaml
[..]
additional_build_steps:
  prepend_galaxy:
    - 'COPY _build/certs/ca.cert.pem /etc/pki/ca-trust/source/anchors/'
    - 'COPY _build/certs/intermediate.cert.pem /etc/pki/ca-trust/source/anchors/'
    - 'RUN update-ca-trust'
[..]
```

In the case above, I added it to the beginning of the `galaxy` stage. If you need to use it in another `stage`, simply replace `prepend_galaxy` with the corresponding
attribute for the [stage](#execution-environment-stages) required.

#### Defining a proxy

Defining a proxy really depends on your use case - again. Most commonly, you'd have to run all connections that go outside of your corporate network through the proxy.
This means, every time we install something. Which means, you'll need it in all `stages` :slightly_smiling_face:.

Further, we need to differentiate between a system proxy and a proxy only for `pip`.

Let's first go with the system proxy. A system proxy is set via the environment variables `http_proxy` and `https_proxy`. And if you want to exclude (sub-)domains,
you can make use of `no_proxy`.

If you've never encountered any of these variables before, you might want to review the [documentation](https://www.gnu.org/software/wget/manual/html_node/Proxies.html) of
them prior to using them.

In `ansible-builder` we can do it like this:

```yaml
[..]
additional_build_steps:
  prepend_builder:
    - 'ENV http_proxy=http://my.proxy.example.com:3128'
    - 'ENV https_proxy=http://my.proxy.example.com:3128'
    - 'ENV no_proxy=sub.example.org,another.domain.evil.corp'
[..]
```

When it comes to the `pip` proxy, we need to set it independently using the `pip config` command:

```yaml
[..]
additional_build_steps:
  prepend_builder:
    'RUN pip3 config --user set global.proxy http://my.proxy.example.com:3128'
[..]
```

In which stage you'll add these, is totally up to you and depends on your *specific* use case.

#### Custom Python package repository

Some users might have their Python packages scanned and published on a custom repository. It is easy to also set that in an EE definition:

```yaml
[..]
additional_build_steps:
  prepend_builder:
    - 'RUN pip3 config --user set global.index-url http://custom.repository.example.com/simple'
    - 'RUN pip3 config --user set global.trusted-host custom.repository.example.com'
[..]
```

:information_source: It is important to note, that you not only need to set the `index-url`, but also need to flag the host as trusted. Otherwise `pip` will refuse installation.

#### Enabling and disabling repositories

Sometimes you'll have the need to enable other repositories than the default UBI and RHEL repositories within the EE build context. For instance to install a custom package.

If you have read the blog post in its entirety then you know that we need to make use of `microdnf` instead of `dnf` in an EE context.

Enabling and disabling repositories is as easy as the tasks above, but this time we need to override an existing environment variable: `PKGMGR_OPTS`.

By default, the environment variable `PKGMGR_OPTS` has the following value (at the time of this writing):

```plaintext
--nodocs --setopt=install_weak_deps=0
```

We'll simply extend that by adding our `--disablerepo=` and `--enablerepo=` switches to it:

```plaintext
--nodocs --setopt=install_weak_deps=0 --disablerepo=* --enablerepo=ubi-8-*"
```

In the above code, we'll disable all repositories and only enable the UBI 8 repositories.

We'll define that like the following in the EE definition:

```yaml
additional_build_steps:
  prepend_builder:
    - 'ENV PKGMGR_OPTS="--nodocs --setopt=install_weak_deps=0 --disablerepo=* --enablerepo=ubi-8-*"'
```

#### The complete complex execution environment: Making use of advanced YAML syntax

If we combine all the previously discussed customizations into one EE definition, we'll end up with something like this:

```yaml
---
version: 3

images:
  base_image:
    name: 'registry.redhat.io/ansible-automation-platform/ee-minimal-rhel8:2.16'

dependencies:
  galaxy: 'requirements.yml'
  python: 'requirements.txt'
  system: 'bindep.txt'

options:
  package_manager_path: '/usr/bin/microdnf'

additional_build_files:
    - src: 'ansible.cfg'
      dest: 'configs/'

    - src: 'ca.cert.pem'
      dest: 'certs/'

    - src: 'intermediate.cert.pem'
      dest: 'certs/'

additional_build_steps:
  prepend_base:
    - 'COPY _build/certs/ca.cert.pem /etc/pki/ca-trust/source/anchors/'
    - 'COPY _build/certs/intermediate.cert.pem /etc/pki/ca-trust/source/anchors/'
    - 'RUN update-ca-trust'
    - 'ENV https_proxy=http://lab-development-rhel8.core.rh.scheib.me:3128'
    - 'ENV http_proxy=http://lab-development-rhel8.core.rh.scheib.me:3128'
    - 'RUN pip3 config --user set global.index-url http://lab-development-rhel8.core.rh.scheib.me/simple'
    - 'RUN pip3 config --user set global.trusted-host lab-development-rhel8.core.rh.scheib.me'
    - 'RUN pip3 config --user set global.proxy http://lab-development-rhel8.core.rh.scheib.me:3128'

  prepend_galaxy:
    - 'COPY _build/certs/ca.cert.pem /etc/pki/ca-trust/source/anchors/'
    - 'COPY _build/certs/intermediate.cert.pem /etc/pki/ca-trust/source/anchors/'
    - 'RUN update-ca-trust'
    - 'ENV https_proxy=http://lab-development-rhel8.core.rh.scheib.me:3128'
    - 'ENV http_proxy=http://lab-development-rhel8.core.rh.scheib.me:3128'
    - 'COPY _build/configs/ansible.cfg /home/runner/.ansible.cfg'

  prepend_builder:
    - 'COPY _build/certs/ca.cert.pem /etc/pki/ca-trust/source/anchors/'
    - 'COPY _build/certs/intermediate.cert.pem /etc/pki/ca-trust/source/anchors/'
    - 'ENV https_proxy=http://lab-development-rhel8.core.rh.scheib.me:3128'
    - 'ENV http_proxy=http://lab-development-rhel8.core.rh.scheib.me:3128'
    - 'RUN update-ca-trust'
    - 'RUN pip3 config --user set global.index-url http://lab-development-rhel8.core.rh.scheib.me/simple'
    - 'RUN pip3 config --user set global.trusted-host lab-development-rhel8.core.rh.scheib.me'
    - 'RUN pip3 config --user set global.proxy http://lab-development-rhel8.core.rh.scheib.me:3128'
    - 'ENV PKGMGR_OPTS="--nodocs --setopt=install_weak_deps=0 --disablerepo=* --enablerepo=ubi-8-*"'

  prepend_final:
    - 'COPY _build/certs/ca.cert.pem /etc/pki/ca-trust/source/anchors/'
    - 'COPY _build/certs/intermediate.cert.pem /etc/pki/ca-trust/source/anchors/'
    - 'ENV https_proxy=http://lab-development-rhel8.core.rh.scheib.me:3128'
    - 'ENV http_proxy=http://lab-development-rhel8.core.rh.scheib.me:3128'
    - 'RUN update-ca-trust'
    - 'RUN pip3 config --user set global.index-url http://lab-development-rhel8.core.rh.scheib.me/simple'
    - 'RUN pip3 config --user set global.trusted-host lab-development-rhel8.core.rh.scheib.me'
    - 'RUN pip3 config --user set global.proxy http://lab-development-rhel8.core.rh.scheib.me:3128'
    - 'ENV PKGMGR_OPTS="--nodocs --setopt=install_weak_deps=0 --disablerepo=* --enablerepo=ubi-8-*"'
...
```

As you surely noticed, we are doing the same things often multiple times. That's due to the different images we build.

There is a little something, that can make this a lot more tidy than it is currently: [`YAML anchors and aliases`](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_advanced_syntax.html#yaml-anchors-and-aliases-sharing-variable-values).

It is not used *that* widely with Ansible to my knowledge, as it can make things pretty quickly, pretty messy, so be careful when using it.

With YAML anchors and aliases, you can include repetitive definitions in place. This not only looks tidier, but it has the benefit that you'll only need to update *one*
reference if you, for instance, change your proxy - instead of updating all of the occurrences.

Let's look into what it looks like when using it:

```yaml
---
version: 3

images:
  base_image:
    name: 'registry.redhat.io/ansible-automation-platform/ee-minimal-rhel8:2.16'

dependencies:
  galaxy: 'requirements.yml'
  python: 'requirements.txt'
  system: 'bindep.txt'

options:
  package_manager_path: '/usr/bin/microdnf'

additional_build_files:
    - src: 'ansible.cfg'
      dest: 'configs/'

    - src: 'ca.cert.pem'
      dest: 'certs/'

    - src: 'intermediate.cert.pem'
      dest: 'certs/'

additional_build_steps:
  prepend_base:
    - &copy-certs |-
         COPY _build/certs/ca.cert.pem /etc/pki/ca-trust/source/anchors/
         COPY _build/certs/intermediate.cert.pem /etc/pki/ca-trust/source/anchors
         RUN update-ca-trust

    - &system-proxy |-
        ENV https_proxy=http://lab-development-rhel8.core.rh.scheib.me:3128
        ENV http_proxy=http://lab-development-rhel8.core.rh.scheib.me:3128
    - &pip |-
        RUN pip3 config --user set global.index-url http://lab-development-rhel8.core.rh.scheib.me/simple
        RUN pip3 config --user set global.trusted-host lab-development-rhel8.core.rh.scheib.me
        RUN pip3 config --user set global.proxy http://lab-development-rhel8.core.rh.scheib.me:3128

  prepend_galaxy:
    - *copy-certs
    - *system-proxy
    - 'COPY _build/configs/ansible.cfg /home/runner/.ansible.cfg'

  prepend_builder:
    - *copy-certs
    - *system-proxy
    - *pip
    - &disable-repos >-
       ENV PKGMGR_OPTS="--nodocs --setopt=install_weak_deps=0 --disablerepo=* --enablerepo=ubi-8-*"

  prepend_final:
    - *copy-certs
    - *system-proxy
    - *pip
    - *disable-repos
...
```

Doesn't that look **much** cleaner now? :sunglasses:.

### Using custom base images

The last topic I'd like to cover is using your very own `base` images. As you've seen in the above EE definition we often repeat the same steps. Such things as copying the
certificates, and setting the system proxy.

If you know that you **always** need certain things in your EE, we can apply a simple concept: Build your own `base` image, based on one of the official EEs.

Things that you might always need are, for instance:

- Specific system proxy
- Specific EE modifications you need to perform always
- Adding your corporate certificates

Again, this *only* applies to things that you need in *all* your EEs.

The process itself is straight forward. We'll define a EE as we used to do, but with the bare minimum of options, and leaving out *everything* that we don't need *always*.
In my case that's Python packages as well as Ansible content:

```yaml
---
version: 3

images:
  base_image:
    name: 'registry.redhat.io/ansible-automation-platform/ee-minimal-rhel8:2.16'

dependencies:
  system: 'bindep.txt'

options:
  package_manager_path: '/usr/bin/microdnf'

additional_build_files:
    - src: 'ansible.cfg'
      dest: 'configs/'

    - src: 'ca.cert.pem'
      dest: 'certs/'

    - src: 'intermediate.cert.pem'
      dest: 'certs/'

additional_build_steps:
  prepend_base:
    # copy files
    - 'COPY _build/certs/ca.cert.pem /etc/pki/ca-trust/source/anchors/'
    - 'COPY _build/certs/intermediate.cert.pem /etc/pki/ca-trust/source/anchors'
    - 'COPY _build/configs/ansible.cfg /home/runner/.ansible.cfg'
    - 'RUN update-ca-trust'

    # override pip proxy and package repository
    - 'RUN pip3 config --user set global.index-url http://lab-development-rhel8.core.rh.scheib.me/simple'
    - 'RUN pip3 config --user set global.trusted-host lab-development-rhel8.core.rh.scheib.me'
    - 'RUN pip3 config --user set global.proxy http://lab-development-rhel8.core.rh.scheib.me:3128'

    # set the system proxy
    - 'ENV https_proxy=http://lab-development-rhel8.core.rh.scheib.me:3128'
    - 'ENV http_proxy=http://lab-development-rhel8.core.rh.scheib.me:3128'

    # disable all repositories but the UBI ones
    - 'ENV PKGMGR_OPTS="--nodocs --setopt=install_weak_deps=0 --disablerepo=* --enablerepo=ubi-8-*"'
...
```

In my imaginary scenario above I *always* need:

- The certificates of my public key infrastructure inside my EE
- Proxy configuration, for both the system itself and `pip`
- We are also going to use a custom Python package repository
- I need one package, that is defined in `bindep.txt`: `socat [platform:rpm]`
- Also I want to ensure that only UBI repositories are enabled

You can also include Ansible content and as well Python packages you always want to have in *each* EE. But please remember: If you don't require it *always*,
but only in specific EEs, you should not consider adding that to your custom `base image`.

Let's build the `base` image:

```shell
$ ansible-builder build -f base.yml -t ee-base-rhel-8:latest -v3
[..]
[3/3] COMMIT ee-base-rhel-8:latest
--> ff13573ab020
Successfully tagged localhost/ee-base-rhel-8:latest
ff13573ab0209bdbc8cdcee3d0aaae4d8d643d91d87f1c38449806e983a15e46
```

Now we have the EE on our local machine and `ansible-navigator images` knows about it:

```shell
   Image                                                                            Tag         Execution environment                 Created                          Size
 1│ee-base-rhel-8                                                                   latest      True                                  About a minute ago               359 MB
 2│ee-minimal-rhel8                                                                 2.16        True                                  4 weeks ago                      345 MB
 3│ee-satellite-rhel-8                                                              latest      True                                  3 days ago                       446 MB
```

Perfect. How do we utilize it now in `ee-satellite-rhel-8`?

That's easy: Can you recall the following section of the EE definition?

```yaml
[..]
images:
  base_image:
    name: 'registry.redhat.io/ansible-automation-platform/ee-minimal-rhel8:2.16'
[..]
```

Guess what? We'll replace that now with our just created image `localhost/ee-base-rhel-8:latest` :sunglasses:.

The final `execution-environment.yml` for our EE `ee-satellite-rhel-8:latest` will look like this:

```yaml
---
version: 3

images:
  base_image:
    name: 'localhost/ee-base-rhel-8:latest'

dependencies:
  galaxy: 'requirements.yml'
  python: 'requirements.txt'
  system: 'bindep.txt'

options:
  package_manager_path: '/usr/bin/microdnf'
...
```

We handle all the necessary extra steps already (certificates, proxy, etc.) in our own `base` image, so you don't have to worry about that anymore when building your
"real" EE. This also has the benefit we only need to do it *once*, compared to the previous multiple times - depending on your use case, of course.

You can push the new `base` EE now to your Automation Hub, or any other container registry you have in your company. That way everybody can use this new corporate
base EE :slightly_smiling_face:.

Easy, wasn't it? :sunglasses:

:warning:
**A word of caution**: Since we now effectively decoupled pulling Red Hat's certified EE of our "real" EE, you **need** to update your custom `base` EE
**regularly** as well. Otherwise you don't receive updates of the certified EEs.

## Conclusion

I *think* I am done. Honestly, I kind of lost the overview with all the topics - it turned out to be longer than I anticipated :rofl:.

I hope you learned something new :slightly_smiling_face:. Please let me know if you find anything to be incorrect!

If you have any questions, just leave a comment and I might be able to incorporate your use-case in this post.

.. until next time,

Steffen

## Change log

### 2024-02-02

- Adding Section about custom `base` images
- Adding `SHA` digest image "pinning" -
  Thanks to my colleague [Michaela Lang](https://at.linkedin.com/in/michaela-lang-900603b9) for pointing this out!
- Clarification about reproducibility with regards to Ansible content, Python and system packages -
  Thanks to my colleague [Michaela Lang](https://at.linkedin.com/in/michaela-lang-900603b9) for pointing this out!
