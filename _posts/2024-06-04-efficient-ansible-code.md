---
title: Writing efficient Ansible code - An unpopular opinion
author: Steffen Scheib
last_modified_at: 2024-06-04
---
## Preface

**Writing simple Ansible code is outdated and you shouldn't do it anymore in 2024.**

I know this is a bold statement, but I guess I have your attention now :sunglasses:.

With this blog post I'd like to explain my way of writing Ansible code and why I think the days where Ansible was just *simple* are no more. I know, this is an unpopular
opinion, but hear me out first, please.

## Ansible's origin

Ansible started of as a simple, easy to use automation language, which is based on [`YAML`](https://yaml.org/) and therefore pretty much self-explanatory. In fact, Ansible
still is (mostly) the same language and is still marketed as such [^ansible_intro]:

> Ansible uses simple, human-readable scripts called playbooks to automate your tasks. You declare the desired state of a local or remote system in your playbook. Ansible
> ensures that the system remains in that state.

While this is still the case today for smaller deployments, we are in a different time. Remember, Ansible was [released](https://en.wikipedia.org/wiki/Ansible_(software)) way
back in 2012 where there were simply other needs.

Tasks would commonly be written something like this:

{% gist 51faef45c51def24084891b850417ce4 %}

:information_source: While this still works today, please do not make it a habit to write tasks like that. It's just not up-to-date anymore.

As you can imagine, with just one or two module options, this was okay back then. But as soon as you have more module options, you'll end up with a super long string that
is not easy to read or to maintain.

Nowadays, the same task above would be implemented like this:

{% gist e1789ff8327a94648f1bdf90a9cf6f2a %}

In a real-world use case you most likely would parameterize each of the available options to allow users to override them. This helps in avoiding to write the same task over
and over again - just slightly different each time (due to installing different packages for instance).

.. anyway, this is where we stand today. The way you define Ansible code today has largely remained the same.

Please don't get me wrong. Ansible itself has evolved **a lot** since it was first released. Starting from
[`roles`](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html),
to [`collections`](https://docs.ansible.com/ansible/latest/collections_guide/index.html), the introduction of
[Event-Driven Ansible](https://ansible.readthedocs.io/projects/rulebook/en/latest/introduction.html#what-is-event-driven-ansible) with
[`rulebooks`](https://ansible.readthedocs.io/projects/rulebook) to Red Hat's latest announcement
[Policy as Code](https://www.redhat.com/en/about/press-releases/red-hat-introduces-policy-code-help-address-ai-complexities-scale).

Ansible is more relevant than ever to fulfill a variety of use cases in today's ever changing IT landscape.

## The problem

In the earlier days of Ansible you'd most likely automated a few hundred or thousand systems; Today we are speaking of tens of thousands to hundred of thousands different
systems.

Not only has the number of systems dramatically increased, but also their complexity. Installing packages on systems (like in the above example) or making simple configuration
changes to a systems' configuration files is just not going to cut it anymore.

Today we are talking to complex systems via various APIs - often [REST APIs](https://www.redhat.com/en/topics/api/what-is-a-rest-api).

Such systems are for instance:

- Various cloud providers, such as [Amazon Web Services (AWS)](https://aws.amazon.com), [Microsoft Azure](https://azure.microsoft.com/) or
  [Google Cloud Platform (GCP)](https://console.cloud.google.com/), etc.
- Applications that help deploying systems, such as [VMware vCenter](https://www.vmware.com/products/vcenter.html),
  [Red Hat Satellite](https://www.redhat.com/en/technologies/management/satellite),
  [Microsoft Active Directory](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview), etc.
- Network devices such as
  [Cisco Internetworking Operating Systems (IOS)](https://www.cisco.com/c/en/us/support/docs/ios-nx-os-software/ios-software-releases-110/13327-ios-early.html),
  [Arista Extensible Operating System (EOS)](https://www.arista.com/en/products/eos),
  [Junos OS](https://www.juniper.net/us/en/products/network-operating-system/junos-os.html),
  etc.
- Virtually any system that has some sort of API to interact with

This is just to name a few to set the scene. This list can easily be extended by a few thousand other systems.

All of the named systems above have something in common, though. They are complex.

As such data retrieved from the APIs of those systems will not always return *what* you need in the *way* you need it.

To name a random made up scenario:

You retrieved a list of hosts of from your Red Hat Satellite and want to check which of those systems are on a particular hypervisor in your VMware vCenter to shut them down
to perform some activity.

:information_source: Again, this is just something made up to set the scene :rofl:.

What most people typically end up doing is something like (pseudo-code below):

{% gist ba6e7a5eb30ea98a0acc90371b494314 %}

Does that look remotely familiar to you?

Yes? **You are doing it wrong.**

I know, this is an unpopular opinion, but let me elaborate on that in the following chapter.

## The solution

How I would do it, is the following (again, pseudo-code):

{% gist 8d5b5789538aee4c5c3a118dbb5bec3e %}

If you wanted to have a list as a fact, I'd go with:

{% gist 6b19791bb6f284f6f334515cd105a554 %}

I know, it looks fancy and totally uncommon, but hear me out first :sunglasses:.

Why would you do something like this in the first place?

Well, in a recent example of one of my customers, they did it the way most people do it: Multiple `loops` and/or `include_tasks` to end up with a list of systems.

They needed to iterate over around 30000 systems. That took them around *eight hours*. I showed them my way of doing it, and they ended up with a *couple of seconds* for
the *exact* same task: Extract a list of relevant systems based on some criteria.

I'll show you below how you can try and measure the performance gain out yourself, but let's first talk about the way I write these tasks.

You need to make yourself familiar with *at least* the [`Jinja2 filters`](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_filters.html) below that will help
you to write efficient code:

- [`selectattr`](https://jinja.palletsprojects.com/en/3.1.x/templates/#jinja-filters.selectattr): Select items of a list of dictionaries
- [`rejectattr`](https://jinja.palletsprojects.com/en/3.1.x/templates/#jinja-filters.rejectattr): Reject items of a list of dictionaries
- [`select`](https://jinja.palletsprojects.com/en/3.1.x/templates/#jinja-filters.select): Select items of a list
- [`reject`](https://jinja.palletsprojects.com/en/3.1.x/templates/#jinja-filters.reject): Reject items of a list

Besides the filters included in [`ansible.builtin`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html#filter-plugins) and the ones included with
[`Jinja2`](https://jinja.palletsprojects.com/en/3.1.x/templates/#list-of-builtin-filters), there are various other filters, which are incredible handy to know about.
One example are the filters included in [`ansible.utils`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html#filter-plugins).

You also *need* to know about the concept of [`Jinja2 tests`](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_tests.html) and the various
[`test plugins`](https://docs.ansible.com/ansible/latest/plugins/test.html) you can utilize. Much like the filters, tests are available from
[`ansible.builtin`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html#test-plugins) and
[`Jinja2`](https://jinja.palletsprojects.com/en/3.1.x/templates/#list-of-builtin-tests) as well as other collections like
[`ansible.utils`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html#test-plugins).

*Ideally*, you also know about [`lookups`](https://docs.ansible.com/ansible/latest/plugins/lookup.html) to further enhance your ability to write efficient code.

There are certain obstacles when chaining filters together, however:

1. If you simply chain the filters after each other, you'll end up with a super long string that is - very much as in the beginning of Ansible - hard to read and maintain
1. The learning curve is incredible steep for newcomers and seasoned Ansible content creators alike
1. You need to wrap your head around how these filters interact with each other. I think it helped me a lot that I used to be a developer early in my career and therefore could
   relate to most of the concepts required to master this technique

Let's discuss these issues one by one.

### Syntax and coding guideline

The first issue, is easily fixed by adapting [`YAML Multiline Syntax`](https://yaml-multiline.info/). There is no reason to not make use of the `YAML Multiline Syntax` - there is nothing
to be afraid of. Sure, it takes time to get it right, but once you practice it, it becomes easy.

I decided for myself to put *every* filter on a new line and indent the code by two spaces if it is a nested expression. This greatly improves readability.

With a nested expression, I refer to something like:

{% gist 5f82b0147b7b80ae6c34bd9c10c043f1 %}

There a various ways nested expression can occur, I have some complex examples in one of my
[roles](https://github.com/sscheib/ansible-role-satellite_publish_promote_content_views) - for instance:

{% gist c96be566ba3b73665b416b71ab0fae63 %}

### Learning curve and different way of working

The second and third issue can be grouped, as they basically refers to the same thing: Learning something new.

Yes, the learning curve is pretty high. It took me quite some time to adapt to this way of writing Ansible code. In the beginning it felt very wrong and I had a hard time wrapping
my head around **how** the filters work and 'interact' with each other.

Essentially, `selectattr`, `rejectattr`, `select` and `reject` work by iterating over all items of a list and select or reject every item that matches your criteria. In a way
we are **combining** `loop` and `when` in a faster way.

For cases where you only have a dictionary available, you can make use of
[`ansible.builtin.dict2items`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/dict2items_filter.html#ansible-collections-ansible-builtin-dict2items-filter),
apply the corresponding filtering and then convert it back to a dictionary with
[`ansible.builtin.items2dict`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/items2dict_filter.html#ansible-collections-ansible-builtin-items2dict-filter).

This is *significantly* quicker than anything you can achieve with `loops`.

There are cases, however, where you *need* to use `loops`, but can still heavily benefit from the 'inline-filtering'. In the next chapter we'll pick one example where I needed
loops and 'inline-filtering' together.

### A complex example broken down

Okay, let's break down the complex example I showed you earlier:

{% gist c96be566ba3b73665b416b71ab0fae63 %}

First, I'd like to provide you some context:

I have a set of `Content Views` over which I loop and need to check if their definition (which is defined in `_satellite_content_views`) contains
any `Lifecycle Environments` the `Content View` should be `promoted` to. Additionally, I want to make sure that those `Content Views` are not `promoted` to any
`Lifecycle Environments` which are not 'allowed'. Allowed `Lifecycle Environments` are optional and specified via `__t_allowed_lifecycle_environments`.

Don't worry if that doesn't make much sense to you if you've never worked with Red Hat Satellite. We are specifically looking at the code; I just wanted to give a little context.

Let's start with the expression that gets us started:

```yaml
(
  _satellite_content_views |
  selectattr('name', 'equalto', __t_content_view.name) |
  selectattr('lifecycle_environments', 'defined') |
  length > 0
)
```

With the above we are selecting all items from `_satellite_content_views` that match `__t_content_view.name` (which is our current item's `name` attribute). Of those items that
matched the earlier expression, we select the items that have the attribute `lifecycle_environments` defined. Lastly we check with the `length` filter if there are any items left.

This expression results either in `true` or `false`.

Next up, is [`ansible.builtin.ternary`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ternary_filter.html):

```yaml
) | ansible.builtin.ternary (
```

`ansible.builtin.ternary` takes two arguments: `true_val` and `false_val`.

Essentially, we implement an `if` statement:

```plaintext
if (expression):
  do this
else:
  do that
```

The expression we are validating against is what we had in step one:

```yaml
(
  _satellite_content_views |
  selectattr('name', 'equalto', __t_content_view.name) |
  selectattr('lifecycle_environments', 'defined') |
  length > 0
)
```

Let's assume this expression evaluates to `true`, then we'll continue with:

```yaml
_satellite_content_views |
selectattr('name', 'equalto', __t_content_view.name) |
selectattr('lifecycle_environments', 'defined') |
map(attribute='lifecycle_environments') |
ansible.builtin.flatten |
```

With the above expression we are again selecting all items from `_satellite_content_views` that match `__t_content_view.name` (which is our current item's `name` attribute). Of
those items that matched the earlier expression we again select those that have the attribute `lifecycle_environments` defined. Then we go ahead and extract only the names of the
`Lifecycle Environment`. This will end up in a nested list, so we lastly `flatten` that list with `ansible.builtin.flatten`.

At this point we have data that looks something like this:

```yaml
[
  'lce-name1',
  'lce-name2',
  'lce-name3',
  'etc.'
]
```

Next up we'll use [`ansible.builtin.intersect`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/intersect_filter.html) which will return the common items
of two lists.

Remember what we want to have are the `Lifecycle Environments` defined in `_satellite_content_views` and compare that to a potentially defined list that holds the 'allowed'
`Lifecycle Environments`.

The portion of the code, I am referring to is:

```yaml
ansible.builtin.intersect(
```

Our first list contains the `Lifecycle Environments` we found out already:

```yaml
[
  'lce-name1',
  'lce-name2',
  'lce-name3',
  'etc.'
]
```

The second list we compare it to, is either defined and contains elements, or isn't: `__t_allowed_lifecycle_environments`

The statement we use to evaluate that is the following:

```yaml
(
  __t_allowed_lifecycle_environments is defined
  and __t_allowed_lifecycle_environments != None
  and __t_allowed_lifecycle_environments | length > 0
)
```

The result of that expression will either be `true` or `false` again. This result will be our input for another `if` statement (using once again `ansible.builtin.ternary`):

```yaml
) | ansible.builtin.ternary(
```

If the expression is `true`, we'll simply pass the list `__t_allowed_lifecycle_environments` as the second list for the `ansible.builtin.intersect`.

So we might end up with data like this:

```yaml
list1: [
  'lce-name1',
  'lce-name2',
  'lce-name3'
]
```

```yaml
list2: [
  'lce-name2',
  'lce-name3'
]
```

:information_source: The lists are not assigned a name `list1` or `list2`. I just use this for illustrating the different lists.

With the above data fed in, `ansible.builtin.intersect` would return a list with the following items:

```yaml
[
  'lce-name2',
  'lce-name3'
]
```

If the last expression is `false`, we basically repeat our step from earlier by selecting all `Lifecycle Environments` for the current processing `Content View`:

```yaml
_satellite_content_views |
selectattr('name', 'equalto', __t_content_view.name) |
selectattr('lifecycle_environments', 'defined') |
map(attribute='lifecycle_environments') |
ansible.builtin.flatten
```

Which would result in the following data:

```yaml
list1: [
  'lce-name1',
  'lce-name2',
  'lce-name3'
]
```

```yaml
list2: [
  'lce-name1',
  'lce-name2',
  'lce-name3'
]
```

:information_source: The lists are not assigned a name `list1` or `list2`. I just use this for illustrating the different lists.

With the above data fed in, `ansible.builtin.intersect` would return a list with the following items:

```yaml
[
  'lce-name1'
  'lce-name2',
  'lce-name3'
]
```

Which is exactly what we want, as there are no 'allowed' `Lifecycle Environments` defined :slightly_smiling_face:.

And finally, coming back to the very first expression:

```yaml
(
  _satellite_content_views |
  selectattr('name', 'equalto', __t_content_view.name) |
  selectattr('lifecycle_environments', 'defined') |
  length > 0
)
```

If that expression would result in `false`, we'd simply `omit` the parameter altogether.

I know, it's complex, but in my personal opinion definitively worth the effort when looking at the performance gain in the next chapter.

## Comparing performance

To compare a *very* simple use case, I created a playbook you can run and evaluate on your own.

In this example we are going to create a list with numbers from 1 to 1000. Then we'll randomly select three numbers and compare if looping is quicker or using `select`:

{% gist 33e2303f422ee41aab20e82c0832463e %}

To compare the performance, we are going to use [`ansible.posix.profile_tasks`](https://docs.ansible.com/ansible/latest/collections/ansible/posix/profile_tasks_callback.html),
which is a [`Callback Plugin`](https://docs.ansible.com/ansible/latest/plugins/callback.html) that shows at the end of the playbook run which task took how long.

Since we are dealing only with numbers (which is not a typical real world scenario), we need to enable `Jinja2 native` which allows for `integers` to be returned from a `Jinja2`
expression to compare against.

The options in the `ansible.cfg` would be the following:

```ini
[defaults]
jinja2_native           = true
callbacks_enabled       = ansible.posix.profile_tasks

[callback_profile_tasks]
task_output_limit       = 100
sort_order              = descending
```

Running the playbook will result in something like the following:

```plaintext
Generate a list of 1000 items --------------------------------------------------------------------------------------------------------------------------------------------- 16.03s
Loop over all items to find the 3 random numbers --------------------------------------------------------------------------------------------------------------------------- 8.27s
Use select to cut down the list -------------------------------------------------------------------------------------------------------------------------------------------- 0.13s
Choose 3 random numbers ---------------------------------------------------------------------------------------------------------------------------------------------------- 0.06s
Show random numbers chosen ------------------------------------------------------------------------------------------------------------------------------------------------- 0.05s
```

In the above example I had a performance improvement of 63x, calculated with:

> 8.27s / 0.13s = 63.615384615

:information_source: This way of 'testing' is overly simplified. It just helps to showcase the general notion that 'in-place filtering' is much quicker than looping.

Your results *will* differ, as I randomly choose numbers and shuffle those around to make it as 'random' as possible.

You'll experience the real world performance impact when you apply it to your own data you process in your playbooks and roles where you know how long it used to run.

<!-- markdownlint-disable MD026 -->
## Why no module or filter ?!
<!-- markdownlint-enable MD026 -->

Most of you probably think:

> Why does this guy even bother, when you can simply write a module or a filter to achieve the same result?

Let me ask you: Are you prepared to write, maintain and distribute tens if not hundreds of filters and modules on your own/in your company? You'd end up with a ton of modules and
filters which essentially do what can be done today using the provided filters by Ansible itself.

What I use is usually shipped with Ansible Core itself. After all, I am only transforming data.

## Conclusion

I *personally* think that this will be the future of writing Ansible code. The simple reason being that complex systems return complex data which you need to process as quickly
as possible.

I am more than happy to hear the thoughts of those who read this post. Please let me know what you think about my way of writing Ansible code and if you consider adopting it.

Until next time,

Steffen

## Change log

### 2025-03-21

- Fixing dead Ansible `rulebooks` link

### 2024-06-04

- Initial release

## Footnotes

[^ansible_intro]: [Introduction to Ansible](https://docs.ansible.com/ansible/latest/getting_started/introduction.html#introduction-to-ansible)
