---
title: Ansible Automation Platform - Automating the execution of job templates and leveraging the 'prompt on launch' feature
author: Steffen Scheib
last_modified_at: 2024-03-09
---
## Introduction

Just today I had an interesting conversation with a customer about the need to **not** duplicate job templates. To quickly give an overview about the problem:

- Tens of thousands of hosts split across a numerous amount of locations
- The various locations all required a specific host (e.g. "jump host") to access the specific location
- End users are making use of Ansible Tower/Ansible Automation Platform and do not know the location or even the names of their servers

With Ansible Tower, the customer solved the problem by simply setting a host variable to each and every host that would override the `ssh_extra_args` variable for each and
every host so that Ansible would chose the correct jump host magically.

This is no longer possible with Ansible Automation Platform 2.x, as the execution nodes (as the formerly known isolates nodes are now called) no longer receive a connection
via SSH, but via [receptor](https://github.com/ansible/receptor) from a controller or hop node (through the [Automation Mesh](https://www.ansible.com/products/automation-mesh)).
The most straight forward way to overcome this obstacle is to basically duplicate the job templates as many times as required, but each time assign a new instance group and the
corresponding hosts to it. Either by specifying the `Limit` or creating a
[Smart Inventory](https://docs.ansible.com/automation-controller/latest/html/userguide/inventories.html#smart-inventories) that would only contain the hosts that are reachable
by the assigned [Instance Group](https://docs.ansible.com/automation-controller/latest/html/userguide/instance_groups.html).

Unfortunately, this recommendation did not work for the customer, because the requirement of the customer was, that their internal customers do not know where the servers are
located and don't want to launch multiple job templates to achieve one simple task.

Okay, so, but what about [Workflow Templates](https://docs.ansible.com/ansible-tower/latest/html/userguide/workflows.html)? Still not good enough, because that
still would require multiple job templates to be set up to achieve the requirement of "1 Task, 1 Inventory, 1 Job Template".

## Ansible Automation Platform and the 'Prompt on launch' option

Let me introduce you to Ansible Automation Platform's "prompt able" fields. :grin:

First off: This idea is not new, so I didn't invent it. I just didn't found a lot of information on how to actually make such a scenario work at scale, so let me describe
the general idea in a bit more detail.

To recap, the requirements at this point are:

- One Job- or Workflow Template
- No splitting of inventories
- Ansible Automation Platform should "know" which instance group to use for which host

The basic idea is the same as the customer has been doing for years: Add an additional variable to each and every host. With Ansible Tower this used to be the
`ssh_extra_args` but since that's no longer a possibility, let's think of how Ansible Automation Platform could "know" which Execution Node to use when executing a Job.

The answer is simple: Instance Groups.

The Instance Groups are defined by the customer based on location. For example a data center in Munich, Germany might be named `ig-germany-munich`
(`ig`: Instance Group). At the time of importing the hosts from their inventory source, they would know the location of the server and thus could save the Instance
Group as a variable, e.g. `instance_group: ig-germany-munich`.

Now that we have the Instance Group on every host, what can we do with it?

Generally, the idea is to create a 'generic job template' that would handle all the logic of assigning the correct instance group to use when targeting a specific host and
execute the "real" job template. I called this Job template `jt-launch_job` - a fitting naming I'd say. :stuck_out_tongue_winking_eye:

But before we can actually launch the job and see the magic happening, let me quickly summarize what I have done:

1. I used Red Hat Satellite as my inventory source. In the Satellite, I added a Host Parameter for each host which I called `instance_group` and set as value the Instance Group
   I'd like to use in Ansible Automation Platform
1. I created the corresponding Instance Groups in Ansible Automation Platform
1. I set up a job template that I'd like to execute on all hosts (with different Instance Groups). In this Job Template I specified the following things:
     - The Inventory to use
     - The Project to use
     - The Credential to use
     - I checked `Prompt on launch` for both Limit and Instance Groups, but did not specify **any** value
     - Last but not least, I checked `Prevent Instance Group Fallback` to ensure that Ansible Automation Platform would **not** fall back to any other Instance Group if the
       specified one is out of capacity at the moment

## Magic

So all that was left now was to create a Job Template with the name `jt-launch_job` and write the Ansible code that would make the magic happen. The code I've written can be
seen on my [GitHub](https://github.com/sscheib/ansible-demo-promptable_job_concept) and is (mostly) straight forward.

I require a `job_template_name` to be set (that's the Job Template where we retrieve the Inventory from and which we will launch multiple times) and a Credential of the type
[Red Hat Ansible Automation Platform](https://docs.ansible.com/automation-controller/4.0.0/html/userguide/credentials.html#red-hat-ansible-automation-platform) to be assigned
to the Job Template. This credential provides all the necessary information that we need to access the Controller (Controller Host, Username, Password).

What I do in the playbook is the following:

- Ensure both the variable and the environment variables are set
- Gather information about the job template from which we will extract the inventory
- The inventory we will use to determine the actual hosts, that will be targeted
- The most challenging part was to build a dictionary of Instance Groups which each one would contain a list of hosts to target. The information was present in the `variables`
  of the host. You can take look at what I mean [here](https://github.com/sscheib/ansible-demo-promptable_job_concept/blob/main/launch_jobs.yml#L52)
- Finally, iterate over that dictionary and launch as many jobs as required :slightly_smiling_face:

For a better visualization, that dictionary would look something like this:

```plaintext
TASK [debug] ****************************************************************************************************************************************************************************
ok: [localhost] => {
    "__hosts_instance_groups": {
        "ig-germany-munich": [
            "host1.example.com",
            "host3.example.com",
            "host6.example.com",
            "host4.example.com"
        ],
        "ig-germany-berlin": [
            "host2.example.com",
            "host5.example.com"
        ]
    }
}
```

This would eventually lead to spawning two times the job template I had specified. Once with the Instance Group `ig-germany-munich` and the hosts `host1, host3, host6, host4`
and the other time with the Instance Group `ig-germany-berlin` and the hosts `host2, host5`.

That's already it - I hope it is useful for someone! :slightly_smiling_face:

## Change log

### 2024-03-09

- `markdownlint` fixes
