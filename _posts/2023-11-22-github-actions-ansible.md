---
title: Getting started with testing Ansible code using Ansible Lint - with or without GitHub Actions
author: Steffen Scheib
---
### Preface
A while ago I started playing around with [GitHub Actions](https://github.com/features/actions) because I really liked the idea of having Ansible code automatically checked after I pushed code to GitHub or after I merged code
from a [feature branch](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/about-branches#about-branches), e.g. `ft-my_feature` to the `main` branch.

Of course this idea isn't new, I am probably among the last ones that actively publishes Ansible code to adopt it. But the idea is simple: Ensure that whenever code gets committed and pushed or merged to a specific branch
(or really to any branch) validate the code if it follows a set of common and good practices with regards to Ansible.

With this blog post, I'd like to provide some guidance around getting started with syntactically testing Ansible code and translate the local testing to GitHub Actions.

### A brief look into history: Testing Ansible code syntactically
Let's first have a look at a very basic way of testing Ansible code. This is really only suitable for beginners, as this way of testing code is very limited, as you'll find
out in a minute.

If you've ever looked into testing an Ansible playbook, you've surely come across [`ansible-playbook --syntax-check`](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html#verifying-playbooks). This command
merely checks a given playbook for syntactical errors. Nothing more. This command is especially helpful when you first get started learning Ansible, as it provides an easy and quick way of identifying whether you have made any
syntactical errors in your playbook.

Let's look at an easy example. A common mistake is the indentation of YAML. At times you just have a space too much or too less and Ansible will not run your playbook.
```yaml
---
- name: 'Testing Ansible code'
  hosts: 'all'

  tasks:
     - name: 'Print a message'
       ansible.builtin.debug:
         msg: 'Test message'

    - name: 'Another task'
      ansible.builtin.debug:
        msg: 'Another message'
...
```

Can you spot the error already? Right, I have indented the first task by one space more than the others. Let's see what `ansible-playbook --syntax-check playbook.yml` has to say about the playbook:

```shell
$ ansible-playbook --syntax-check playbook.yml 
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'
ERROR! We were unable to read either as JSON nor YAML, these are the errors we got from each:
JSON: Expecting value: line 1 column 1 (char 0)

Syntax Error while loading YAML.
  did not find expected key

The error appears to be in '/home/sscheib/playbook.yml': line 10, column 5, but may
be elsewhere in the file depending on the exact syntax problem.

The offending line appears to be:


    - name: 'Another task'
    ^ here
[steffen@development ~]$
```

Great, it caught the error, perfect.

Let's fix the code and re-run that check:

```
$ ansible-playbook --syntax-check playbook.yml 

playbook: playbook.yml
```

Looks good :slightly_smiling_face:

As I said earlier, `ansible-playbook --syntax-check` is great for getting started, but it certainly has its limitations.

One of those, let's call it 'limitations', is the fact that it can only test *playbooks*. Nothing else. It was never designed to test anything else, hence I called it a 'limitation', but nevertheless, it's something
`ansible-playbook --syntax-check` is not capable of doing.

Let's take the simplest of all cases. I have a playbook and want to include some tasks using the `ansible.builtin.include_task` module. The file I'd like to include is `my_tasks.yml`, which looks like this:

```yaml
---
- name: 'Print yet another message'
  ansible.builtin.debug:
    msg: 'Test message'
...
```

Let's validate the syntax:

```shell
$ ansible-playbook --syntax-check my_tasks.yml 
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'
ERROR! 'ansible.builtin.debug' is not a valid attribute for a Play

The error appears to be in '/home/sscheib/my_tasks.yml': line 2, column 3, but may
be elsewhere in the file depending on the exact syntax problem.

The offending line appears to be:

---
- name: 'Print yet another message'
  ^ here
```

It does not work. At least not like that. The reason is simple: `ansible-playbook` is only meant to test, well, *playbooks*. Okay, then let's use `ansible.builtin.include_tasks` to include the tasks in `my_tasks.yml` and re-run
the syntax check:

```
$ ansible-playbook --syntax-check playbook.yml 

playbook: playbook.yml
```

Great, that looks promising! But to be sure, let's introduce an intentional issue into `my_tasks.yml` to ensure it is really checked:

```yaml
---
- name: 'Intentionally wrong'
    - name: 'Print yet another message'
      ansible.builtin.debug:
        msg: 'Test message'
...
```

Let's re-run that check:

```
$ ansible-playbook --syntax-check playbook.yml 

playbook: playbook.yml
```

Erm, wait what?! Why did it pass?

Well, there is an important distinction of `include_tasks` and `import_tasks`[^import_vs_include]. `include_tasks` will load the file `my_test.yml` when it encounters that exact task that loads the tasks. `import_tasks` on the
other hand, loads the tasks *before* the playbook run has started. Essentially you can think of it as simple adding the tasks that we are including in place of the `import_tasks` statement.

When we re-run the syntax check one last time, you'll see, it'll fail when using `import_tasks`:

```shell
$ ansible-playbook --syntax-check playbook.yml 
ERROR! We were unable to read either as JSON nor YAML, these are the errors we got from each:
JSON: Expecting value: line 1 column 1 (char 0)

Syntax Error while loading YAML.
  did not find expected key

The error appears to be in '/home/sscheib/my_tasks.yml': line 3, column 5, but may
be elsewhere in the file depending on the exact syntax problem.

The offending line appears to be:

- name: 'Intentionally wrong'
    - name: 'Print yet another message'
    ^ here
```

Okay, do I need to use now `include_tasks` over `import_tasks` and all is resolved? Well, not quite.

What about testing Ansible roles and not only playbooks? When we talk about testing roles, can we also test the variables located in `vars/` and `defaults/` within a role directory? What about `handlers/`?

I am afraid that won't work with `ansible-playbook --syntax-check` at all. Which is perfectly fine because, again, `ansible-playbook` was never meant to test anything other than *playbooks*.

But what, if I'd like to test my roles?! Keep on reading, we'll talk about it in the next chapter :sunglasses:

### Introducing Ansible Lint

#### Overview
[Ansible Lint](https://ansible.readthedocs.io/projects/lint/) is a tool that was introduced to aid Ansible playbook, role and collections developers in writing consistent good Ansible code. It basically starts where
`ansible-playbook --syntax-check` ends. Ansible Lint is *much* more sophisticated than the syntax check of the `ansible-playbook` command.
Ansible Lint does not only syntactically check playbooks, roles and collections, but also ensures common practices are followed. Ansible Lint further ensures common practices are followed for YAML by using `yamllint`.

In case you wondered whether you need to use Ansible Lint alongside `ansible-playbook --syntax-check` to have a better test coverage, then I have good news for you: Ansible Lint calls the previously discussed `ansible-playbook --syntax check` [^ansible_lint_playbook_syntax] by default. :sunglasses:

Ansible Lint works with rules. Rules can be individually enabled or disabled, but generally the rules included in Ansible Lint are well-thought out and shouldn't be disabled carelessly. These rules are developed by the 
Ansible Community on [Github](https://github.com/ansible/ansible-lint/tree/main). Ansible Lint - and the rules - are written, the same as Ansible itself, in Python. The individual rules that are included in the upstream variant of Ansible Lint (which can be installed via `pip install ansible-lint`) can also be reviewed on [GitHub](https://github.com/ansible/ansible-lint/tree/main/src/ansiblelint/rules).

For Red Hat subscribers, a RPM variant is provided by Red Hat in the Red Hat Ansible Automation Platform repository as part of the tool set for Ansible developers.

#### Getting started
So what do you need to get started with Ansible Lint?

First of, we need to install Ansible Lint. I personally use the `pip` variant, although I have access to the Ansible Automation Platform repository. I use the `pip` variant
for two reasons:
1. Working with "the next" version of Ansible Lint helps me fix issues in my code before they actually are introduced into Red Hat's downstream variant and thus I can guide
my customers (as I work at Red Hat as a Technical Account Manager for Ansible) better
2. I use [`pre-commit`](https://pre-commit.com/) for all of my git repositories - we'll come to that later - and thus need to use a Python virtual environment (`venv`) anyway
   so why not use Ansible Lint also via `pip` :slightly_smiling_face:

To configure Ansible Lint to work with a good set of rules, I'll primarily work with two configuration files (one for Ansible Lint, the other for `yamllint`), which
I personally found to be very good for all my needs.

The Ansible Lint configuration file can be put at three places in your current working directory [^ansible_lint_config_location]:
- `.ansible-lint` (which is what I use personally)
- `.config/ansible-lint.yml`
- `.config/ansible-lint.yaml`

My configuration file looks like the following:
```yaml
---
exclude_paths:
  - '.git/'
  - 'files/'

kinds:
  - tasks: 'tasks/*.{yml,yaml}'
  - vars: 'vars/*.{yml,yaml}'
  - vars: 'defaults/*.{yml,yaml}'
  - meta: 'meta/main.{yml,yaml}'
  - yaml: '.ansible-lint'
  - yaml: '.github/workflows/*.{yml,yaml}'
  - yaml: '.pre-commit-config.yaml'
  - yaml: '.yamllint'
  - yaml: 'collections/requirements.yml'

loop_var_prefix: '^(__|{role}_)'
max_block_depth: 20
offline: true
profile: 'production'
skip_action_validation: false
skip_list: []
task_name_prefix: '{stem} | '
use_default_rules: true
var_naming_pattern: '^[a-z_][a-z0-9_]*$'
warn_list:
  - 'experimental'
write_list:
  - 'none'
...
```

Let's go through the more important rules step by step:
##### `exclude_paths`
As the name says it: This will exclude paths from being processed. Anything below `.git` is not necessary to be scanned, nor do any files stored inside `files/`.

##### `kinds`
With `kinds` we tell Ansible Lint which files should be considered as what file kind.

##### `offline`
With `offline` we define that collections and roles defined in `collections/requirements.yml` should not be installed using `ansible-galaxy` *before* linting. That means
in turn, of course, that you are responsible to install the roles and collections required by your role or collection. I disabled it on purpose to speed up the linting
process a little.

##### `profile`
`profile` tells Ansible Lint which [profile](https://ansible.readthedocs.io/projects/lint/profiles/) it should use. The profile defines which rules will be loaded by Ansible
Lint and thus validated against. The profile [`production`](https://ansible.readthedocs.io/projects/lint/profiles/#production) is the most restrictive one and is intended for
code that 'meets requirements for inclusion in Ansible Automation Platform (AAP) as validated or certified content.' - you can't write any better code than with this profile, I guess :slightly_smiling_face:

##### `write_list`
Ansible Lint includes (in later versions) the possibility to [fix](https://ansible.readthedocs.io/projects/lint/autofix/) problematic code by invoking Ansible Lint via
`--fix`. I don't want Ansible Lint to touch anything actively, thus I have set this to `none` which will prevent any action.

There are many more options that can be configured in Ansible Lint, which I personally don't use, please [check them out](https://ansible.readthedocs.io/projects/lint/).

#### yamllint

So, I have spoken of two configuration files. The other one is `.yamllint` - yes, Ansible Lint invokes [`yamllint`](https://github.com/adrienverge/yamllint) as well :blush:.
`yamllint` does one thing: Validate YAML files kind of the same way as Ansible Lint does: with rules. These rules can, the same as for Ansible Lint, be configured with the
configuration file located at `.yamllint`. In other words, `yamllint` makes sure the YAML code itself is valid. It doesn't care about the Ansible portion at all, as you can
verify any YAML file with it. Conveniently, Ansible Lint includes `yamllint` :sunglasses:.

My configuration file looks like follows:
```yaml
---
extends: 'default'

rules:
  braces:
    level: 'error'
    max-spaces-inside: 1
  brackets:
    level: 'error'
    max-spaces-inside: 1
  colons:
    level: 'error'
    max-spaces-after: -1
  commas:
    level: 'error'
    max-spaces-after: -1
  comments: 'enable'
  comments-indentation: 'enable'
  document-end: 'enable'
  document-start: 'enable'
  empty-lines:
    level: 'error'
    max: 3
  empty-values: 'enable'
  float-values: 'enable'
  hyphens: 'enable'
  indentation: 'enable'
  key-duplicates: 'enable'
  key-ordering: 'disable'
  line-length:
    max: 120
  new-line-at-end-of-file: 'enable'
  new-lines:
    type: 'unix'
  octal-values: 'enable'
  quoted-strings: 'enable'
  trailing-spaces: 'enable'
  truthy: 'enable'

yaml-files:
  - '*.yaml'
  - '*.yml'
  - '.yamllint'
  - '.ansible.lint'
...
```

I have basically enabled all rules, but one: `key-ordering` (more to that in a little bit). I also extended the default line length of 80 to a more reasonable 120. I did some
research and this seems to be the "unwritten universally accepted" line length for Ansible. Anything below 120 is kind of hard to deal with, especially if you deal with long
variable names.


#### My thoughts on `key-ordering` in the context of Ansible
Okay, this is rather off topic, as this is more a very strong *personal* opinion on why this setting shouldn't be used for Ansible. If you are interested in my thoughts,
keep on reading :sunglasses:

`yamllint` has a setting which is called `key-ordering`. `key-ordering` forces you to write module options in an alphabetical order, which I don't want to do.

Here is why:

While ordering arguments alphabetically at first seems pretty useful, as it ensures you have the very same ordering for every task you write, I personally prefer to write the most important arguments of a module at the top, followed by an order that makes sense to me.

For instance, creating a simple directory with `ansible.builtin.file` will look like the following when I write it:

```yaml
- name: 'Create a directory'
  ansible.builtin.file:
    path: '/tmp/test_dir'
    state: 'directory'
    owner: 'steffen'
    group: 'steffen'
    mode: '0644'
  register: 'out'
```

With `key-ordering` enabled, I'd be forced to write it like so:

```yaml
- ansible.builtin.file:
    dest: '/tmp/test_dir'
    group: 'steffen'
    mode: '0644'
    owner: 'steffen'
    state: 'directory'
  name: 'Create a directory'
  register: 'out'
```

For me *personally* this is counter-intuitive. While it guarantees that you *always* have the very same ordering, having split `user` and `group` is not something I would do.
Of course, your mileage may vary.

Moreover, I think that it makes reading Ansible a lot harder. Maybe it's just me, but I am used to having the `name` of a task as the *very first* attribute. Not the 
module. Of course, the position of the module can change - depending on the name. :grin:

Look at the following example task:
```yaml
---
- name: 'Do something'
  theforeman.foreman.activation_key:
    another_arg: 'blubb'
    validate_certs: true
  register: 'output'
  loop: ['bla']
  loop_control:
    loop_var: '_my_var'
...
```

When enabling `key-ordering`, the following ordering is required:
```yaml
---
- loop: ['bla']
  loop_control:
    loop_var: '_my_var'
  name: Do something'
  register: 'output'
  theforeman.foreman.activation_key:
    another_arg: 'blubb'
    validate_certs: true
...
``` 

This is so far off Ansible-wise that I'd say, "No, that's not Ansible. GO AWAY!". Again, I think it makes reading Ansible *a lot* harder.

And in case you ask yourself whether the above example would technically work with Ansible: Yes it does. Of course, `theforeman.foreman.activation_key` will complain it is
missing required arguments, but Ansible accepts the above example perfectly fine.

Again, your mileage may vary, but for me personally, this shouldn't be allowed with Ansible. Really.

#### Ansible Lint summary
For beginners Ansible Lint might seem intimidating at first, but keep in mind, you don't need to start right away with the
[`production`](https://ansible.readthedocs.io/projects/lint/profiles/#production) profile. Maybe start with the
[`basic`](https://ansible.readthedocs.io/projects/lint/profiles/#basic) or the [`moderate`](https://ansible.readthedocs.io/projects/lint/profiles/#basic) profile and work your
way up to the more sophisticated profiles.

If you'd like to share your code via [Ansible Galaxy](https://galaxy.ansible.com/ui/) (I'll talk about that in a later blog post), I'd encourage you to validate your code
against the [`shared`](https://ansible.readthedocs.io/projects/lint/profiles/#shared) profile (which is also what Ansible Lint recommends) or if you are really into it, go for
the [`production`](https://ansible.readthedocs.io/projects/lint/profiles/#production) profile right away.

Better Ansible code usually encourages other people to contribute to your code, so at the end, everybody benefits :slightly_smiling_face:.

### [`pre-commit`](https://pre-commit.com/)
Having the goal in mind to have your code linted when merging a branch or pushing code to a branch, one would think the general workflow would look something like
the following:
1. Write Ansible code
2. Commit your code
3. Push your code to GitHub/GitLab/etc.
4. Get your code linted automatically
5. On linting errors, fix the Ansible code
6. Go back to 2.

You can already see how time consuming and ineffective this approach would be.

My workflow looks a little different. I start by writing Ansible Code and regularly test the code using Ansible Lint on my development machine. This helps with immediately
correcting the code before I even finished writing it.

To 'force' myself into that continuous testing, I make use of [`pre-commit`](https://pre-commit.com/). `pre-commit` basically installs a
[git hook](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) script that runs anything I define before being allowed to commit the code.
For `pre-commit` to work you need to have initialized a git repository in your current directory (`git init`) or cloned an existing git repository.
`pre-commit` can be installed via `pip install pre-commit` (please use a Python virtual environment).

`pre-commit` can be individually configured using a [configuration file](https://pre-commit.com/#2-add-a-pre-commit-configuration), which can be placed in your git initialized
directory at `.pre-commit-config.yaml`

My configuration file looks like this:
```yaml
---
repos:
  - repo: 'https://github.com/ansible/ansible-lint'
    rev: 'v6.20.3'
    hooks:
      - name: 'Ansible-lint'
        additional_dependencies:
          # https://github.com/pre-commit/pre-commit/issues/1526
          # If you want to use specific version of ansible-core or ansible, feel
          # free to override `additional_dependencies` in your own hook config
          # file.
          - 'ansible-core>=2.15'
        always_run: true
        description: 'This hook runs ansible-lint.'
        entry: 'python3 -m ansiblelint -v --force-color'
        id: 'ansible-lint'
        language: 'python'
        # do not pass files to ansible-lint, see:
        # https://github.com/ansible/ansible-lint/issues/611
        pass_filenames: false
...
```

Once you have installed `pre-commit` and put the configuration file in place, it's time to install the git hook using `pre-commit install`. The installed git hook will ensure that every time you commit code using `git commit`, the code is linted using Ansible Lint. If your code passes Ansible Lint, the commit will be performed, otherwise
Ansible Lint will let you know what's wrong and the commit is prevented. Your code remains untouched, of course (unless you run Ansible Lint with the `--fix` option that is).

I usually run `pre-commit run --all` after I installed the git hook to check if everything works as expected.

`pre-commit` has been a game changer to me as it forces me to fix faulty Ansible code before I commit it.

### Ansible Lint with GitHub Actions
Finally, we are there. I know, it's been a long post (again), but I felt this context was necessary before starting linting your code with GitHub Actions.

So how do you actually use Ansible Lint with GitHub Actions? Well, with another configuration file, of course, but it's *really* easy.

First off, we'll create a directory inside your git repository: `.github/workflows`. This is the place where GitHub looks for any workflows to run.
I have a file in `.github/workflows` that configures the Ansible Lint GitHub action (you can name it whatever you want, as long as it has either `yml` or `yaml`
as extension): `ansible-lint.yml`

My `ansible-lint.yml` looks like this:
```yaml
---
name: 'ansible-lint'
on:
  pull_request:
    branches: ['main']
  workflow_dispatch: {}
jobs:
  build:
    name: 'Ansible Lint'
    runs-on: 'ubuntu-latest'
    steps:
      - uses: 'actions/checkout@v4'
      - name: 'Run ansible-lint'
        uses: 'ansible/ansible-lint@main'
...
```

The above configuration runs the official [GitHub Action for Ansible Lint](https://github.com/marketplace/actions/run-ansible-lint) on a pull request to the `main` branch. By
specifying the attribute `workflow_dispatch` (it's on purpose an empty dictionary) you are able to run the linting on demand, even without a pull request.

### Conclusion
That's it already for this time.

I hope this post is helpful to dive into the world of linting Ansible code - with or without GitHub Actions. Happy Linting! :sunglasses:

[^import_vs_include]: [Including and Importing Task Files](https://docs.ansible.com/ansible/2.9/user_guide/playbooks_reuse_includes.html#includes-vs-imports)
[^ansible_lint_config_location]: [Ansible Lint configuration file locations](https://ansible.readthedocs.io/projects/lint/configuring/)
[^ansible_lint_playbook_syntax]: [Ansible Lint syntax-check](https://ansible.readthedocs.io/projects/lint/rules/syntax-check/#correct-code)
