---
title: Execution Environments - Getting started and building complex execution environments (EEs) using ansible-builder 3
author: Steffen Scheib
---
### Preface
To understand why execution environments (EEs) are such a crucial part of modern Ansible we need to briefly look into the past.
To a time where there were no EEs, but *something* else to accomodate for certain situations.

In the following brief look into the past, I'll cover only the basics. For experienced Ansible users which are aware of Ansible's history this chapter is most likely useless -
feel free to skip to the next chapter :slightly_smiling_face:.

Okay, let's get into it :sunglasses:.

#### A quick look into the past

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
2. Install the Python packages using [`pip installs packages`, better known as `pip`](https://ianbicking.org/blog/2008/10/pyinstall-is-dead-long-live-pip.html) to install the
   Python packages either in the user context ([`--user`](https://pip.pypa.io/en/stable/user_guide/#user-installs)) or in the system context (**don't ever install
   Python packages in the system context!**)
3. Use a [`Python Virtual Environment`, better known as `venv`](https://docs.python.org/3/library/venv.html)

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

### Ansible's execution environments (EEs): An introduction
So what exactly are EEs?

First and foremost, EEs are container images. You can imagine EEs basically as packaged Ansible control nodes. Essentially, you can picture it as your development machine
or your centralized Ansible control node that runs your Ansible content in production, packaged in a container image with everything it needs to run specific Ansible content.

EEs come by default with a few things:
1. Ansible, usually `ansible-core`
2. [`ansible-runner`](https://ansible.readthedocs.io/projects/runner/en/latest/) 
3. A specific Python version that fits the Ansible version used

That's really it - for the bare minimum.

But what makes EEs so powerful and versatile is their great flexibility bundled with all the advantages of containers. If you haven't worked with containers before, you might
not see the massive benefits of EEs just yet. I understand that. Some people might even be afraid of containers as they haven't been using them until this point.

Rest assured: Containers are not bad. They are actually awesome if you understand the benefits of them and understand that you need to adapt in some ways to them.
This might sound at first like a big task, but it's not, really.

Especially in the context of EEs, containers are not just not bad, but are *the* solution to all the problems I outlined above. I know, this is might not be yet clear at this
point. You - hopefully - understand or at the very least begin to understand what I mean once you finished reading this blog post.

For those who are afraid of containers: You don't *really* need to work with containers in that sense when dealing with EEs. When building and executing EEs, you essentially
work with two 'wrappers' that take care of building and running your containers in such a way that you usually don't need to interact with container images and containers
directly. I explain all of that in greater detail a bit later.
