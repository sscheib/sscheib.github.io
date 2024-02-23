---
title: Execution Environments - Getting started and building complex execution environments (EEs) using ansible-builder 3
author: Steffen Scheib
---

# Preface

To understand why execution environments (EEs) are such a crucial part of modern Ansible we need to briefly look into the past.
To a time where there were no EEs, but *something* else to accomodate for certain situations.

In the following brief look into the past, I'll cover only the basics. For experienced Ansible users which are aware of Ansible's history this chapter is most likely useless -
feel free to skip to the next chapter :slightly_smiling_face:.

Okay, let's get into it :sunglasses:.

## A quick look into the past

Probably most users of Ansible encountered at some point a situation, where they needed to install a specific version of a Python module as a specific module of a collection
required it to be installed.

Let's take as an example [`community.general.proxmox`](https://docs.ansible.com/ansible/latest/collections/community/general/proxmox_module.html). The
module `proxmox` [specifies](https://docs.ansible.com/ansible/latest/collections/community/general/proxmox_module.html#requirements) the following requirements:
> The below requirements are needed on the host that executes this module.
> - proxmoxer
> - requests

Both [`proxmoxer`](https://github.com/proxmoxer/proxmoxer) and [`requests`](https://github.com/psf/requests) are
[Python packages](https://packaging.python.org/en/latest/tutorials/installing-packages/).

At this point we know, we need to install both Python packages *somehow* on our system where we execute Ansible. This leaves us with basically three choices:

1. Use the system's package manager to install the packages - if packaged versions are available
1. Install the Python packages using [`pip installs packages`, better known as `pip`](https://ianbicking.org/blog/2008/10/pyinstall-is-dead-long-live-pip.html) to install the
   Python packages either in the user context ([`--user`](https://pip.pypa.io/en/stable/user_guide/#user-installs)) or in the system context (**don't ever install
   Python packages in the system context!**)
1. Use a [`Python Virtual Environment`, better known as `venv`](https://docs.python.org/3/library/venv.html)

We can choose either of the three options above and we should be good to, right?

*Theoretically*, yes. *Practically*, not really - well, at least it comes with its challenges.

Why? Allow me to tell you little a story that might sound very familar to you :innocent:.

Let's assume we'd pick option three, the Python Virtual Environments. `venvs` are usually what most Ansible users would go with, as it 'seperates' the Python packages of the
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
basically eleminates all of the issues above. You are right - for the most part at least. There are situtations where a perfect documentation is not sufficient. Just imagine
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
This might sound at first like a big task, but it's not, really.

Especially in the context of EEs, containers are not just not bad, but are *the* solution to all the problems I outlined above. I know, this might not be clear yet at this
point. You - hopefully - understand or at the very least begin to understand, what I mean once you finished reading this blog post.

For those who are afraid of containers: You don't *really* need to work with containers in that sense when dealing with EEs. When building and executing EEs, you essentially
work with two 'wrappers' that take care of building and running your containers in such a way that you usually don't need to interact with container images and containers
directly. I explain all of that in greater detail a bit later.

With the knowledge of what EEs are, I guess everyone can easily imagine why they solve the issues `venvs` have. Essentially, you define and extend your EE as necessary while
you actively develop Ansible content. Since everything is now neatly packaged in a container, you can easily share and reuse it.

### Execution environments: Getting started

Working with EEs is pretty straight-forward once you know what you need. In the previous chapter I referred to two 'wrappers' that enable you to use and create EEs - these are:

1. [`ansible-navigator`](https://ansible.readthedocs.io/projects/navigator/)
1. [`ansible-builder`](https://ansible.readthedocs.io/projects/builder/en/latest/)

Let me give you a *very brief* overview over these tools.

#### `ansible-navigator`

`ansible-navigator` is essentially a drop-in replacement for all the `ansible-` commands (e.g. `ansible-playbook`, `ansible-inventory`, `ansible-doc`, etc.) you are used to when
working with Ansible on the command line (CLI). You *need* `ansible-navigator` to run your Ansible content via CLI inside an EE, as `ansible-playbook` is not able to run
something inside an EE.

#### `ansible-builder`

`ansible-builder` is the tool to use when building execution environments. It makes creating and extending existing EEs easy and straight-forward. Under the hood, it
essentially interacts with [`podman`](https://podman.io/) to create the images.

Now that we know the *very basics* about EEs let's actually get started. :sunglasses:

#### Existing EEs: An overview

The understand the concept behind EEs, please allow me to contextualize the Ansible ecosystem with regards to EEs a little.

There are existing EEs which you can utilize to get started. Both the Ansible community and Red Hat maintain a few EEs. The ones maintained by the
Ansible community are either hosted on [quay.io](https://quay.io/) or on GitHubs Container Registry [ghcr.io](https://ghcr.io). The ones Red Hat maintains are called *Red Hat
certified execution environments* and are provided via Red Hat's Container Registry [registry.redhat.io](https://catalog.redhat.com/).

The difference between EEs of the Ansible community (we at Red Hat, usually refer to them as *upstream*) and Red Hat's EEs (we at Red Hat, usually refer to them as
*downstream*) is that upstream EEs are *usually* based on CentOS stream, while downstream EEs are based on
[Red Hat's Universal Base Image (UBI)](https://catalog.redhat.com/software/base-images) and are *essentially* Red Hat Enterprise Linux (RHEL) in containers, if you will.

:information_source: To use Red Hat certified EEs you need to be a Red Hat subscriber If you don't own any subscriptions, you can make use of                                   [Red Hat's Developer Subscription](https://developers.redhat.com/articles/faqs-no-cost-red-hat-enterprise-linux) which is provided at no cost by Red Hat.

The major difference, however, is the level of support you'll get. Upstream in general moves really fast and does not do any
[backports](https://en.wikipedia.org/wiki/Backporting) for older Ansible versions.
In contrast, Red Hat supports their EEs for a [defined lifecycle](https://access.redhat.com/support/policy/updates/ansible-automation-platform) and backports bugfixes and/or
security fixes to older EEs which are using older `ansible-core` versions, which are still under support by Red Hat.

Please don't get confused that the [linked lifecycle](https://access.redhat.com/support/policy/updates/ansible-automation-platform) points to the Ansible Automation Platform
lifecycle. That's not a mistake. The reason being that Red Hat treats several components of the Ansible ecosystem (such as Execution and Decision Environments, `ansible-core`,
`ansible-navigator`, `ansible-builder` among other things) as one *platform*. As Ansible Automation Platform subscriber, you'll get a defined set of components in a
certain version, which ensures that all components are compatible with each other.

Upstream essentially treats various components, such as EEs, `ansible-builder`, `ansible-navigator` etc. as seperate projects and therefore you *might* encounter difficulties
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

For upstream EEs, you are probably going to choose the [AWX EE](https://quay.io/repository/ansible/awx-ee) which is meant to be used when running your code in
[AWX](https://github.com/ansible/awx) (one of Red Hat's upstream projects that are part of the Ansible Automation Platform).

In this blog post I am going to focus on using *Red Hat certified execution environments*.

Certified EEs come, at the time of this writing, in different variants:

- Either based on UBI 8 or UBI 9
- Either contain `ansible-core` *only* or `ansible-core` and a set of [*certified collections*](https://catalog.redhat.com/software/search?target_platforms=Red%20Hat%20Ansible%20Automation%20Platform)
- In a *special* case, one EE contains Ansible Engine 2.9, but this EE is going to go away in the future as Ansible Engine reached its downstream end of life in December last
  year (31.12.2023). This EE is/was provided merely for the reason to support customers in their migration from older Ansible Automation Platform versions, as Ansible itself
  has changed drastically.

*Technically* that's four variants, but UBI 8 or UBI 9 'only' change the underlying operating system :sweat_smile:.

Again, Red Hat's EEs are hosted on [registry.redhat.io](https://catalog.redhat.com) where the 'user-browsable interface' is [catalog.redhat.com](https://catalog.redhat.com).
When browsing to [registry.redhat.io](https://catalog.redhat.com) with your browser, you'll be redirected to [catalog.redhat.com](https://catalog.redhat.com) automatically, but
you'll be pulling the EEs of registry.redhat.io.

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
variant is exactly as supported as the `supported` one is, albeit it's totally understandable if you are confused :smile:.

There is also this *special* EE, which contains Ansible Engine 2.9 only: `ansible-automation-platform-24/ee-29-rhel8`. Again, this EE is/was provided merely for the reason to
support customers in their migration from older Ansible Automation Platform/Ansible versions.

Secondly, the namespace difference (`ansible-automation-platform/ee-minimal-rhel9` vs. `ansible-automation-platform-24/ee-minimal-rhel9`) is due to 'historic' reasons,
basically.

Nowadays, we encourage customers to make use of the *versionless* or *multi stream* EEs as *versioned* or *single stream* EEs are going to go away in the future.
I prefer using the terms *versionless* and *versioned* over *multi stream* and *single stream*, as it - in my opinion - more accurately describes what is meant.

*Versioned* in this specific case refers to EEs that are build for a *specific* Ansible Automation Platform **version** - such as `ansible-automation-platform-24/ee-minimal-rhel9`. From this example, the Ansible Automation Platform version that contains the EE `ee-minimal-rhel9` is 2.4. This is how EEs where distributed in the past.

Nowadays, EEs are rather independent of the Ansible Automation Platform version and are distributed '*independently*', meaning not tied to a specific Ansible Automation
Platform version in that sense. Hence: *versionless*.

The idea behind the *versionless* EEs is basically to pick and chose which `ansible-core` version you'd like to have included in your EE. The
[version tags](https://catalog.redhat.com/software/containers/ansible-automation-platform/ee-minimal-rhel8/62bd87442c0945582b2b4b37/history) of, for instance,
`ansible-automation-platform/ee-minimal-rhel8` correspond to `ansible-core` versions.

One thing important to know: Often, container images make use of the `latest` tag, which commonly - who would have guessed it - refers to the latest available version of the
container image. But when you are trying to pull `ansible-automation-platform/ee-minimal-rhel8:latest` using `podman`, you'll be greeted with an error:

```
# podman pull ansible-automation-platform/ee-minimal-rhel8:latest
Resolved "ansible-automation-platform/ee-minimal-rhel8" as an alias (/etc/containers/registries.conf.d/001-rhel-shortnames.conf)
Trying to pull registry.redhat.io/ansible-automation-platform/ee-minimal-rhel8:latest...
WARN[0001] Failed, retrying in 1s ... (1/3). Error: initializing source docker://registry.redhat.io/ansible-automation-platform/ee-minimal-rhel8:latest: reading manifest latest in registry.redhat.io/ansible-automation-platform/ee-minimal-rhel8: unsupported: This repository does not use the "latest" tag to track the most recent image and must be pulled with an explicit version or image reference. For more information, see: https://access.redhat.com/articles/4301321
```

:information_source: Don't get confused that I am using `podman` to pull the EE. You don't have to do the same. We'll be using `ansible-builder` for that. Using `podman` in
this specific case is just easier.

In [Red Hat's linked knownledgebase article](https://access.redhat.com/articles/4301321) it is explained *why* this change was introduced for a number of container images if
you are curious of the *why*. In the context of the *versionless* EEs you only need to know, that it doesn't work :slightly_smiling_face:.

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

### Using existing EEs with `ansible-navigator`

After we've learnt a lot about EEs in the previous chapters, let's actually use them :slightly_smiling_face:. To use the EEs on the CLI, let me first
introduce you to `ansible-navigator`. `ansible-navigator` is crucial for developing and testing EEs before actually putting them into production.

#### Installing `ansible-navigator`

There are multiple ways you can install `ansible-navigator`, which are detailed in the
[installation documentation](https://ansible.readthedocs.io/projects/navigator/installation/).

To install a stable version of `ansible-navigator` on RHEL (I'll refer to it as *downstream*), you need access to one of the Red Hat Ansible Automation Platform
repositories, for instance:

- Red Hat Ansible Automation Platform 2.4 for RHEL 8 x86_64 (RPMs), repository ID: `ansible-automation-platform-2.4-for-rhel-8-x86_64-rpms`
- Red Hat Ansible Automation Platform 2.4 for RHEL 9 x86_64 (RPMs), repository ID: `ansible-automation-platform-2.4-for-rhel-9-x86_64-rpms`

:information_source: Once again the hint that the above repositories require you to be a [Red Hat subscriber](#existing-ees-an-overview).

Simply enable the repository you've chosen using `subscription-manager`:

```
# sudo subscription-manager repos --enable ansible-automation-platform-2.4-for-rhel-8-x86_64-rpms
```

Then install `ansible-navigator` using `dnf` or `yum`:

```
# sudo dnf install ansible-navigator
```

If you'd rather like to use the latest available upstream version, you can do so with [`pip`](https://pypi.org/project/pip/):

```
sudo dnf install python3-pip
python3 -m pip install ansible-navigator --user
```

For `ansible-navigator` to be able to work with EEs, you either need [`podman`](https://podman.io/) or [`docker`](https://www.docker.com/) on your system, as we are going to use
containers and naturally need *something* to handle them :slightly_smiling_face:. I'll use `podman` for the remainder of this post, but all commands I am going to use with
`podman` are working exactly the same when replacing `podman` with `docker`.

Afterall, we really only run **one** command with `podman`. The remainder will be handled by `ansible-builder` :sunglasses:.

:information_source: The upstream variant of `ansible-navigator` is not exclusively available on RHEL; You can install it on a few more operating systems. Please refer to the
[installation documentation](https://ansible.readthedocs.io/projects/navigator/installation/) to get an overview of all supported operating systems.

:warning: There is one important difference in the above versions: The downstream variant of `ansible-navigator` will default to an EE that is provided and supported by Red Hat,
while the upstream variant will default to an upstream EE. At the time of this writing, the downstream variant of `ansible-navigator` of the Ansible Automation Platform 2.4
repository will pull `registry.redhat.io/ansible-automation-platform-24/ee-supported-rhel8:latest` while the upstream variant will pull `ghcr.io/ansible/creator-ee:v0.22.0`.

#### Getting started with `ansible-navigator`
