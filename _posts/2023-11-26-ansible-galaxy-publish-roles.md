---
title: Publishing roles to Ansible Galaxy - with or without GitHub Actions
author: Steffen Scheib
---
### Preface

First off, this is going to be a small blog post. I personally found it too difficult to figure out how to publish my roles to
[Ansible Galaxy](https://galaxy.ansible.com/ui/), that's why I didn't used do it for my roles right from the start.

Instead I simply published my roles on GitHub. One downside was, the roles were
not as easy to discover as if they were published on Ansible Galaxy - the place where people usually look for existing roles.

A few weeks back, I was asked on LinkedIn whether I could publish my roles on Ansible Galaxy, so that more users could discover them and use them - and maybe contribute some
features to them :slightly_smiling_face:. That's when I started looking into this again and I have to admit, that I struggled quite a bit until I finally figured out how
to publish the roles.

At that time I thought it was only me who was not able to figure it out, but quite recently another person reached out to me on LinkedIn asking me whether I could
help getting some roles published to Ansible Galaxy.

So I decided to write this small blog post to provide a little bit of guidance. Once you know where to look, it really is easy.

### Requirements

What you'll need to follow along is the following:

1. A GitHub account. With the GitHub account you'll be able to login to Ansible Galaxy. To my knowledge there is no other way (anymore) to register or log into Ansible Galaxy
2. A role you'd like to publish. The role needs to be stored in a git repository that is publicly accessible on GitHub. Anything other than GitHub does not work (to my knowledge)
3. An Ansible Galaxy API token. You'll find it by visiting [Ansible Galaxy](https://galaxy.ansible.com/), then login via GitHub, click on `Collections` on the left hand side,
   followed by `API token`. Finally, click on `Load token`

   :warning: Be aware that loading a new token, invalidates your old token (if you had a token previously)

Please be aware, that the role needs to follow the [official role structure](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html#role-directory-structure).

Further, you'll need the role content to be at the top level of your git repository - at least I couldn't figure out a different way. What I mean by that is, that you
cannot have a `roles/` directory inside your git repository and publish any role that is inside `roles/`, but instead the role structure needs to be at the top level.

If you struggle to understand what I mean, please look at one of my roles (e.g.
[ansible-role-rhel_iso_kickstart](https://github.com/sscheib/ansible-role-rhel_iso_kickstart)) to understand the role structure I am referring to.

### Publishing roles using the command line

Once you meet all the requirements, let's look into publishing your role using the command line first.

For the purpose of this blog, I have create a [sample/dummy role](https://github.com/sscheib/ansible-role-dummy) which merely prints a given message via
`ansible.builtin.debug`. As the blog post's purpose is to provide a little help when publishing roles to Ansible Galaxy, the complexity of the role doesn't matter at all.

Once you have written your role, please lint it using Ansible Lint, as described in my blog post
[Getting started with testing Ansible code using Ansible Lint - with or without GitHub Actions](https://blog.scheib.me/2023/11/22/github-actions-ansible.html). Ansible
Galaxy will perform some form of linting, but I don't know which profile is getting applied.

Let's first look at my role structure:

```shell
$ tree ansible-role-dummy/
ansible-role-dummy/
├── defaults
│   └── main.yml
├── LICENSE
├── meta
│   └── main.yml
├── README.md
├── tasks
│   └── main.yml
└── vars
    └── main.yml

4 directories, 6 files
```

As you can see it is a very basic role without any bells and whistles.

With that being said, importing the role is actually pretty straight forward. We need to run the command `ansible-galaxy role import` and provide some command line
switches and values.

What the `ansible-galaxy role import` command needs, is the following:

1. The Galaxy server to use. In our case that's [`https://galaxy.ansible.com`](https://galaxy.ansible.com). The corresponding command line argument is `--server` or `-s`
1. The aforementioned token you generated on [`https://galaxy.ansible.com`](https://galaxy.ansible.com). The corresponding command line argument is `--token` or `--api-key`
1. Your GitHub username. This is a positional argument without any dedicated command line switch.
1. Your repository name. This is also a positional argument without any dedicated command line switch.
1. By default the repository name will be the role name. If you want a different role name to be used you need to have set that in `meta/main.yml` and additionally pass it to
   the `ansible-galaxy` command
1. Also by default, the `master` branch is going to get used of your GitHub repository. I'll use `main` as my default branch, therefore I need to pass the command line
   argument `--branch` to it and provide it with the value of `main`

Additionally, as said, your role needs to be pushed to your GitHub repository and the repository needs to be publicly accessible.

The `ansible-galaxy` command will look like following:

```shell
ansible-galaxy role import --branch <YOUR_BRANCH> --role-name <YOUR_ROLE_NAME> --server https://galaxy.ansible.com --token <YOUR_TOKEN> <YOUR_GITHUB_USERNAME> <YOUR_GITHUB_REPOSITORY_NAME>
```

:information_source: `<YOUR_GITHUB_REPOSITORY_NAME>` really refers to the **name** of the repository, not the URL you'd use to pull down your code.

To import my role `dummy` from my GitHub repository [`ansible-role-dummy`](https://github.com/sscheib/ansible-role-dummy), with my GitHub username `sscheib`, I need to run
the following command:

```shell
ansible-galaxy role import --branch main --role-name dummy --server https://galaxy.ansible.com --token MY_TOKEN sscheib ansible-role-dummy
```

The import process looks like the following:

```shell
$ ansible-galaxy role import --branch main --role-name dummy --server https://galaxy.ansible.com --token e4a4811672a51e064bb5fc97290b5d0a92f9094d sscheib ansible-role-dummy
Successfully submitted import request 2056392558890447847457884111007500265
Starting import: task_id=2056392558890447847457884111007500265, pulp_id=018c0bfb-20c0-7d1a-9559-11f84021c7e9

==== PARAMETERS ====
importer username: sscheib
matched user: sscheib id:XXXXX
github_user: sscheib
github_repo: ansible-role-dummy
github_reference: main
alternate_role_name: dummy

==== CHECK FOR MATCHING ROLE(S) ====
user:sscheib repo:ansible-role-dummy did not match any existing roles

===== CLONING REPO =====
cloning https://github.com/sscheib/ansible-role-dummy ...

===== GIT ATTRIBUTES =====
github_reference(branch): main
github_commit: b689e1ee5c8c165ceb0531b7560f234b064fd443
github_commit_message: Initial commit
github_commit_date: 2023-11-26T15:09:54+01:00

===== LOADING ROLE =====
Importing with galaxy-importer 0.4.16
Determined role name to be dummy
Linting role dummy via ansible-lint...
...ansible-lint run complete
Legacy role loading complete

===== PROCESSING LOADER RESULTS ====
enumerated role name dummy
created new role id:37383 sscheib.dummy

===== COMPUTING ROLE VERSIONS ====

==== SAVING ROLE ====

Import completed
```

Once that's completed successfully, you can now login to [`https://galaxy.ansible.com`](https://galaxy.ansible.com) and browse to your profile and see the imported role.
In my case, you can find it on my [Ansible Galaxy profile](https://galaxy.ansible.com/ui/standalone/roles/sscheib/dummy/).

#### Updating a role

I intentionally did not fully adjust the `README.md` which I copied from one of my other roles, so I'll be able to showcase what updating a role looks like.

First, I went ahead and edited my `README.md` file, as can be seen by the
[git commit](https://github.com/sscheib/ansible-role-dummy/commit/26e6b2f5396eaf4a9d73682ba657fa8ef5340fd1) I have performed. I basically removed a bunch of lines,
which I 'forgot' to remove earlier.

Once I pushed the code to GitHub, I can now trigger an import of the same role again:

```shell
ansible-galaxy role import --branch main --role-name dummy --server https://galaxy.ansible.com --token MY_TOKEN sscheib ansible-role-dummy
Successfully submitted import request 2056399856190313650539110125029237695
Starting import: task_id=2056399856190313650539110125029237695, pulp_id=018c0c57-3b99-762f-ad6b-85509cb723bf

==== PARAMETERS ====
importer username: sscheib
matched user: sscheib id:XXXXX
github_user: sscheib
github_repo: ansible-role-dummy
github_reference: main
alternate_role_name: dummy

==== CHECK FOR MATCHING ROLE(S) ====
user:sscheib repo:ansible-role-dummy matched existing role sscheib.dummy id:37383

===== CLONING REPO =====
cloning https://github.com/sscheib/ansible-role-dummy ...

===== GIT ATTRIBUTES =====
github_reference(branch): main
github_commit: 26e6b2f5396eaf4a9d73682ba657fa8ef5340fd1
github_commit_message: README.md
github_commit_date: 2023-11-26T16:52:39+01:00

===== LOADING ROLE =====
Importing with galaxy-importer 0.4.16
Determined role name to be dummy
Linting role dummy via ansible-lint...
...ansible-lint run complete
Legacy role loading complete

===== PROCESSING LOADER RESULTS ====
enumerated role name dummy

===== COMPUTING ROLE VERSIONS ====

==== SAVING ROLE ====

Import completed
```

This time the output looks a little different, as it found an existing role (which we pushed earlier) and now only updates it.

#### Tagging releases

The last thing to do is to create a new [`git tag`](https://git-scm.com/book/en/v2/Git-Basics-Tagging) to have a proper version on the role in Ansible Galaxy.

This can easily be done via `git tag v1.0.1` (of course substitute the version with yours) and by pushing the tag to GitHub using `git push origin v1.0.1` (again, please
substitute the version with yours).

This will looks something like this:

```shell
$ git tag v1.0.1
$ git push origin v1.0.1
Total 0 (delta 0), reused 0 (delta 0), pack-reused 0
To github.com:sscheib/ansible-role-dummy.git
 * [new tag]         v1.0.1 -> v1.0.1
```

Perfect. We can list the tags using `git tag`:

```shell
$ git tag
v1.0.1
```

Finally, we import the role once again:

```shell
ansible-galaxy role import --branch main --role-name dummy --server https://galaxy.ansible.com --token MY_TOKEN sscheib ansible-role-dummy
Successfully submitted import request 2056400027670397563943177324356333381
Starting import: task_id=2056400027670397563943177324356333381, pulp_id=018c0c59-65ae-7667-85db-e266b7826f45

==== PARAMETERS ====
importer username: sscheib
matched user: sscheib id:XXXXX
github_user: sscheib
github_repo: ansible-role-dummy
github_reference: main
alternate_role_name: dummy

==== CHECK FOR MATCHING ROLE(S) ====
user:sscheib repo:ansible-role-dummy matched existing role sscheib.dummy id:37383

===== CLONING REPO =====
cloning https://github.com/sscheib/ansible-role-dummy ...

===== GIT ATTRIBUTES =====
github_reference(branch): main
github_commit: 26e6b2f5396eaf4a9d73682ba657fa8ef5340fd1
github_commit_message: README.md
github_commit_date: 2023-11-26T16:52:39+01:00

===== LOADING ROLE =====
Importing with galaxy-importer 0.4.16
Determined role name to be dummy
Linting role dummy via ansible-lint...
...ansible-lint run complete
Legacy role loading complete

===== PROCESSING LOADER RESULTS ====
enumerated role name dummy

===== COMPUTING ROLE VERSIONS ====
adding new version from tag: v1.0.1
tag: v1.0.1 version: 1.0.1

==== SAVING ROLE ====
```

This time the output looks again a little different, please pay attention to the following from the above output:

```shell
[..]
===== COMPUTING ROLE VERSIONS ====
adding new version from tag: v1.0.1
tag: v1.0.1 version: 1.0.1
[..]
```

Browsing now to my role on [Ansible Galaxy](https://galaxy.ansible.com/ui/standalone/roles/sscheib/dummy/) I can see that indeed the current version is labeled as `v1.0.1`.

Mission accomplished :sunglasses:.

### Publishing roles using GitHub Actions

Lastly, I'd like to show you an automated way of publishing your roles to Ansible Galaxy by using [GitHub Actions](https://github.com/features/actions).

As in my [other blog post about Ansible Lint](https://blog.scheib.me/2023/11/22/github-actions-ansible.html#ansible-lint-with-github-actions), setting up a GitHub Action is
super easy. I am using [`@robertdebock's`](https://github.com/robertdebock) GitHub Action [`galaxy-action`](https://github.com/robertdebock/galaxy-action), which takes care
of publishing a role to Ansible Galaxy.

For this, we need two things:

1. The Ansible Galaxy token
1. A GitHub Action workflow definition file

Let's first create the workflow definition file.
For that we need to create a directory path for the Github workflows within your repository's root directory via:

```shell
mkdir -p .github/workflows
```

Next, simply place the workflow configuration file in there. I named mine `ansible-galaxy.yml` (but you can choose any name) and it contains the following:

<!-- markdownlint-disable MD003 MD022 -->
{% highlight yaml %}
{% raw %}
---
name: 'Publish latest release to Ansible Galaxy'

on:
  push:
    branches:
      - 'main'
  workflow_dispatch: {}

jobs:
  build:
    name: 'Publish to Ansible Galaxy'
    runs-on: 'ubuntu-latest'
    steps:
      - name: 'checkout'
        uses: 'actions/checkout@v2'
      - name: 'galaxy'
        uses: 'robertdebock/galaxy-action@1.2.0'
        with:
          galaxy_api_key: '${{ secrets.galaxy_api_key }}'
          git_branch: 'main'
...
{% endraw %}
{% endhighlight %}
<!-- markdownlint-enable MD003 MD022 -->

The above workflow will run on any push or merge to the `main` branch and will publish the latest code in the branch `main` to Ansible Galaxy.

There is one catch, however. The Action will need access to your Ansible Galaxy token you created earlier. Of course, you don't want to put that into the workflow definition
file in plain text, that's why we are going to use a [GitHub Repository Secret](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions), which is
referenced via {% highlight yaml %}{% raw %}${{ secrets.galaxy_api_key }}{% endraw %}{% endhighlight %} in the above workflow definition.

To create a GitHub Repository Secret for your repository, head to your GitHub repository, click on `Settings`. On the left-hand side, select `Secrets and variables`, followed
by `Actions`. Lastly click on `New repository secret` to create a new Repository Secret. Give it the name of `GALAXY_API_KEY` and put in your token you created earlier.

Finally, push the changed code to GitHub and enjoy an automated upload of your roles to Ansible Galaxy. :slightly_smiling_face:

You can always check for the status of the workflow by visiting your GitHub repository's page, click on `Actions` and check on the latest run. From there you can also
run the workflow manually again (thanks to the definition of `workflow_dispatch: {}` in the workflow file), should you have the need for it.

### Closing thoughts

A small blog post this time around, but I hope you learned something new! :slightly_smiling_face:

Until next time,

Steffen :sunglasses:
