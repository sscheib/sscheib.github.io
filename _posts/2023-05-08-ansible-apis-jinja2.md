---
title: Automating APIs with Ansible .. and Jinja2
author: Steffen Scheib
---
## Preface

The other day I wanted to automate some of my newly acquired MikroTik switches using Ansible and the excellent Ansible collection
[`community.general.routeros`](https://docs.ansible.com/ansible/latest/collections/community/routeros/index.html).

With the aforementioned collection you get all sorts of modules and plugins. One of those modules is
[`community.routeros.api_find_and_modify`](https://docs.ansible.com/ansible/latest/collections/community/routeros/api_find_and_modify_module.html#ansible-collections-community-routeros-api-find-and-modify-module)
which is - as the name suggests - a module that will find a specific path on any [`RouterOS (ROS)`](https://help.mikrotik.com/docs/display/ROS/RouterOS) capable device and
modify the value, if required. The modules depends on the Python library [`librouteros`](https://librouteros.readthedocs.io/en/3.2.1/), which talks to the `REST` API of any
`ROS` device.

<!-- markdownlint-disable MD026 -->
## Is it idempotent?!
<!-- markdownlint-enable MD026 -->

### Idempotency for newcomers in Ansible

*For those who are not new to Ansible, feel free to skip over this section; If you chose to read it: I know, this is overly simplified, but for the purpose of this blog post,
it's enough.*

Here is the thing. Good Ansible modules are [idempotent](https://docs.ansible.com/ansible/latest/reference_appendices/glossary.html#term-Idempotency), which means, Ansible
will **only perform a change if the desired state is not already present**.

Okay, for Ansible newcomers this might not mean a lot, so here is a quick example.

Imagine you want to disallow root logins via `SSH`. Naturally, you'd go to `/etc/ssh/sshd_config` adjust the option `PermitRootLogin` and restart or reload `SSHd` to pick up the
change.

The next day, you forgot whether you actually changed the value of `PermitRootLogin` and you want to ensure that you really changed it. Obviously, there a few ways to check
that, but for this example, let's assume, you'd just login via `SSH` and open up `/etc/ssh/sshd_config` and verify whether the desired value for `PermitRootLogin` is already
present.

There could be two possible outcomes for you when you open up the `SSHd` configuration file:

1. `PermitRootLogin` has the appropriate value set
1. `PermitRootLogin` has not set the appropriate value

With option number 2, you'd obviously change the value of `PermitRootLogin` and restart or reload the `SSHd`.

.. but what about option 1? Would you really change the file **again** and restart or reload the `SSHd` **again**, although you verified that the appropriate value for
`PermitRootLogin` has been set? Of course not.

This is precisely what idempotency with regards to Ansible is. Ansible compares a desired state to an existing state for each task in the play and acts accordingly. If there is
nothing to change to achieve the desired state, nothing is changed. In theory (and mostly in practice) this type of transaction (checking whether the desired state is already
present) takes up less resources and thus needs less time to apply when compared to an operation that actually changes something.

Idempotency enables you to rerun the same automation over and over again and you will always receive the same result. If something changed, you know that somebody or something
has messed around with your system and if nothing changes, you can be sure everything is exactly the way it should be.

<!-- markdownlint-disable MD026 -->
### .. now, is the `community.routeros.api_find_and_modify` module idempotent?!
<!-- markdownlint-enable MD026 -->

Apparently, it is. [A small sentence](https://docs.ansible.com/ansible/devel/collections/community/routeros/api_find_and_modify_module.html#notes) in the documentation reveals
it:
> [..] The latter case is needed for idempotency of the task: once the values have been changed, there should be no further match.

With that cleared up, I wanted to ensure that my `SSHd` port on my switches listens to a different port (default is 22). First, I logged into my switch via `SSH` and came up
with the following command to change it:

```plaintext
/ip/service/set ssh port=2222
```

Okay, that was easy. Let's put that into an Ansible playbook:

```yaml
---
- name: 'Play around with MikroTik'
  hosts: 'all'
  gather_facts: false

  tasks:
    - name: 'Configure SSH'
      community.routeros.api_find_and_modify:
        hostname: 'mikrotik-crs326.example.com'
        username: 'username'
        password: 'password'
        ca_path: 'example.com_ca_bundle.pem'
        tls: true
        validate_cert_hostname: true
        validate_certs: true
        path: 'ip service'
        find:
          name: 'ssh'
        values:
          name: 'ssh'
          port: 2222
        require_matches_min: 1
        require_matches_max: 1
      delegate_to: 'localhost'
...
```

Perfect, it works. The module reports back properly when nothing needs changing, thus it is idempotent.

As I don't want to repeat that very same task for each and every service (`HTTP/S`, `API`, `Telnet`, etc.) I decided to write a role which would configure all my "basic stuff"
(such as services).

I came up with the following task:

<!-- markdownlint-disable MD032 MD037 -->
{% highlight yaml %}
{% raw %}
- name: 'Configure service {{ __t_service.name }}'
  community.routeros.api_find_and_modify:
    hostname: '{{ _mk_hostname }}'
    username: '{{ _mk_username }}'
    password: '{{ _mk_password }}'
    port: '{{ _mk_api_port }}'
    ca_path: '{{ _mk_ca_path | default(omit) }}'
    tls: '{{ _mk_tls }}'
    validate_cert_hostname: '{{ _mk_validate_cert_hostname }}'
    validate_certs: '{{ _mk_validate_certs }}'
    path: 'ip service'
    find:
      name: '{{ __t_service.name }}'
    values:
      name: '{{ __t_service.name }}'
      disabled: '{{ __t_service.disabled | default(omit) }}'
      port: '{{ __t_service.port | default(omit) }}'
      certificate: '{{ __t_service.certificate | default(omit) }}'
      tls-version: "{{ __t_service['tls-version'] | default(omit) }}"
    require_matches_min: 1
    require_matches_max: 1
  delegate_to: 'localhost'
{% endraw %}
{% endhighlight %}
<!-- markdownlint-enable MD032 MD037 -->

I defined my variables like the following:

```yaml
---
mk_services:
  # enabled services
  - name: 'ftp'
    disabled: false
    port: 21

  - name: 'ssh'
    disabled: false
    port: 2222

  - name: 'www-ssl'
    disabled: false
    port: 443
    certificate: 'mikrotik-crs326.example.com.cert.pem'
    tls-version: 'only-1.2'

  - name: 'api-ssl'
    disabled: false
    port: 8729
    certificate: 'mikrotik-crs326.example.com.cert.pem'
    tls-version: 'only-1.2'

  - name: 'winbox'
    port: 8291
    disabled: false

  # disables services
  - name: 'telnet'
    disabled: true

  - name: 'www'
    disabled: true

  - name: 'api'
    disabled: true
...
```

In the `main.yml` of the role I would iterate over all the defined services like so:

<!-- markdownlint-disable MD032 MD037 -->
{% highlight yaml %}
{% raw %}
- name: 'Include tasks to configure IP services'
  ansible.builtin.include_tasks:
    file: 'services.yml'
  loop: '{{ _mk_services }}'
  loop_control:
    loop_var: '__t_service'
{% endraw %}
{% endhighlight %}
<!-- markdownlint-enable MD032 MD037 -->

... and that should be it. It wasn't.

## Getting mad

So whenever I ran the playbook that would include my role, it would report a changed status for the services:

- `ftp`
- `ssh`
- `www-ssl`
- `api-ssl`
- `winbox`

.. and would **not** report a changed status for:

- `www`
- `api`

I started debugging, using Ansible's [diff mode](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_checkmode.html#using-diff-mode), which was luckily possible
with this module (diff mode is optional) and it reported the following:

```plaintext
TASK [mikrotik_base : Configure service ftp] ********************************************************************************************************************************************
--- before
+++ after
@@ -6,7 +6,7 @@
             "disabled": false,
             "invalid": false,
             "name": "ftp",
-            "port": 21
+            "port": "21"
         }
     ]
 }

changed: [mikrotik-crs326.example.com -> localhost]
```

.. wait what?! Why is this a string? Okay, never mind, let's add a conversion to `int` using `| int` for the `port` value:
{% highlight yaml %}
{% raw %}
port: '{{ __t_service.port | int }}'
{% endraw %}
{% endhighlight %}
That will do it :sunglasses:

... and it didn't do it.

I started debugging with the `ansible.builtin.debug` module:

<!-- markdownlint-disable MD032 -->
{% highlight yaml %}
{% raw %}

- ansible.builtin.debug:
    var: __t_service.port

- ansible.builtin.debug:
    msg: '{{ __t_service.port | type_debug }}'

- ansible.builtin.debug:
    msg: '{{ __t_service.port | int }}'

- ansible.builtin.assert:
    that:
      - __t_service.port is number

- ansible.builtin.debug:
    msg: '{{ __t_service.port | int | type_debug }}'

- ansible.builtin.meta: end_play

{% endraw %}
{% endhighlight %}

.. and I couldn't understand the result:

```plaintext
TASK [mikrotik_base : ansible.builtin.debug] ********************************************************************************************************************************************
ok: [mikrotik-crs326.example.com] => {
    "__t_service.port": "21"
}

TASK [mikrotik_base : ansible.builtin.debug] ********************************************************************************************************************************************
ok: [mikrotik-crs326.example.com] => {
    "msg": "int"
}

TASK [mikrotik_base : ansible.builtin.debug] ********************************************************************************************************************************************
ok: [mikrotik-crs326.example.com] => {
    "__t_service.port | int": "21"
}

TASK [mikrotik_base : ansible.builtin.assert] *******************************************************************************************************************************************
ok: [mikrotik-crs326.example.com] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [mikrotik_base : ansible.builtin.debug] ********************************************************************************************************************************************
ok: [mikrotik-crs326.example.com] => {
    "msg": "int"
}

TASK [mikrotik_base : ansible.builtin.meta] *********************************************************************************************************************************************

PLAY RECAP ******************************************************************************************************************************************************************************
mikrotik-crs326.example.com : ok=13   changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

All of the tests clearly indicated that {% raw %}`{{ __t_service.port }}`{% endraw %} is an integer even without `| int`.

## Meet `jinja2_native`

While I almost lost any hope and was *that* close to either open an issue in the
[`community.routeros`' Github](https://github.com/ansible-collections/community.routeros/issues) (which I hesitated because I didn't want to look like an idiot) or simply live
with the non-idempotency, I decided to give it a last try after I left that topic alone for a few days.

After a **ton** of looking around the internet, I found a [pull request](https://github.com/ansible/ansible/pull/68560) for Ansible that
[fixed](https://github.com/ansible/ansible/issues/46169) the conversion of a `JSON` string to a `dict`, where the conversion would corrupt that `dict`. After looking at the
[change log](https://docs.ansible.com/ansible/latest/porting_guides/porting_guide_core_2.11.html#playbook), I realized my issue:
> The `jinja2_native` setting now does not affect the template module which **implicitly returns strings**. For the template lookup there is a new argument `jinja2_native`
> (off by default) to control that functionality. The rest of the `Jinja2` expressions still operate based on the `jinja2_native` setting.

I looked through the closed pull requests to find the one which introduced `jinja2_native`, and there it was:
[Allow config to enable native `jinja` types](https://github.com/ansible/ansible/pull/32738). The
[documentation](https://docs.ansible.com/ansible/latest/reference_appendices/config.html#default-jinja2-native) describes `jinja2_native` as follows:
> This option preserves variable types during template operations.

Surely, that **must** be it! I went ahead and added `jinja2_native` to the `[defaults]` section of my `ansible.cfg` and re-ran my playbook.

***It worked. Finally.***

For why `jinja2_native` is off by default, I can only speculate and would guess for backwards compatibility reasons. Not a bad reasoning, but in my case, it would have been a
lot less painful if it was turned on by default ..

.. oh well, the next challenge surely won't wait for long. :smile:

## Change log

### 2024-04-01

- Spelling fix

### 2024-03-17

- Fixing incorrectly rendered code block

### 2024-03-11

- `markdownlint` fixes
- correcting spelling errors

### 2024-03-10

- `markdownlint` fixes
