---
title: Automating Red Hat Satellite 6 - End to End
author: Steffen Scheib
---
<!-- markdownlint-disable MD033 -->
{% raw %}
<style>
table th:first-of-type {
    width: 7%;
}
table th:nth-of-type(2) {
    width: 94%;
}
</style>
{% endraw %}
<!-- markdownlint-enable MD033 -->

## Preface

This blog post does not follow the style of my previous blog posts. It is more or less a step by step manual on how to use
[my repository on GitHub](https://github.com/sscheib/ansible_satellite) to automate Red Hat Satellite using Ansible.

Further, a good understanding of Satellite and, more importantly, a good understanding of the concepts around Ansible is **required**.
Especially [variable precedence](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#understanding-variable-precedence) should be well known.
Don't get me wrong, it is still a step by step guide, but nevertheless, if you want to adjust this step by step guide to *your* infrastructure (which is likely that you'd
want to do that :grin:), you **need** to understand the concepts around Ansible.

All variables we define are either [host or group variables](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html#organizing-host-and-group-variables).
This has the benefit that your inventory is not cluttered with variables, and additionally, this allows you to create different variables for each of your Satellite instances
(if you have multiple). I use group variables only for variables that are required by all my Satellite instances.

Please note, only a fraction of the available variables are used for each role. Please make sure to check out each section's documentation of the respective role for a
complete list of available variables. Further, relevant documentation is also linked in each section that complements the respective steps that are done in Ansible if
you'd like to know more about the general (manual) procedure.

In below sections I make heavy use of [GitHub `gists`](https://docs.github.com/en/get-started/writing-on-github/editing-and-sharing-content-with-gists/creating-gists#about-gists)
to show **my personal configuration** of the respective roles. You **need to modify every gist so that it works for your environment**. For this purpose, I
documented (almost) every variable in the `gists` to ensure the usage is clear. Should something be not clear, please refer to the role documentation (that is linked in every
section), which provides additional context as well as possibly other variables to use that make sense to use in your environment.

I tested all of this with Satellite 6.12, 6.13 and 6.14. It *should* work the same for other Satellite versions, but I cannot guarantee it.

:information_source: Just as a general information: Variables that are prefixed with `satellite_` are 'official' role variables of the roles we are going to use. Variables
prefixed with `sat_` are custom variables (which usually get merged into a `satellite_` variable eventually).

:information_source: I am going to use `ansible-playbook` in this blog post and will thus install all required collections and roles as the current user. Of course, you can
also make use of `ansible-navigator` with an Execution Environment that contains the collections and roles which are defined in `collections/requirements.yml`. This blog
post won't cover the procedure of how to do that, however.

:information_source: You need to be a Red Hat subscriber in order to follow this blog post. If you are not, you can use a
[no-cost Red Hat Developer Subscription](https://developers.redhat.com/articles/faqs-no-cost-red-hat-enterprise-linux), which includes the Ansible Automation Platform
subscription (required for the certified collections) and the Satellite Infrastructure subscription. Of course, you could also use the upstream projects
(e.g. `Foreman`/`Katello`) and upstream collections (e.g. `theforeman.foreman`) to follow along, but for that you need to adjust *every* playbook
(and not 'just' the GitHub `gists`) and it might not work at all, as I haven't tested it.

## Prerequisites

1. You need a host that will run Ansible and carry out the automation.

    :information_source: This has been tested on:

      - RHEL 8.7 and Ansible Core 2.14.6
      - RHEL 8.8 and Ansible Core 2.15.4
      - RHEL 8.9 and Ansible Core 2.15.4
      - RHEL 9.2 and Ansible Core 2.14.2
      - RHEL 9.3 and Ansible Core 2.14.2

    :warning: RHEL 7 is not going to work for this purpose, as only Ansible Engine 2.9 can (officially) be installed on it (which is too old for most - if not all -
    collections we are going to use).

1. Login to the host you'll be using to carry out the automation via SSH. Install `git`, check out my repository and change to the directory that has been created with
   the `git clone` command:

    <!-- markdownlint-disable MD014 -->
    ```shell
    $ sudo dnf install -y git
    $ git clone https://github.com/sscheib/ansible_satellite.git
    $ cd ansible_satellite
    ```
    <!-- markdownlint-enable MD014 -->

1. Configure Ansible that it'll be able to download
[Red Hat's Ansible Certified Content Collections](https://access.redhat.com/support/articles/ansible-automation-platform-certified-content) that are needed for the playbooks
to work from either your local Automation Hub or from [Red Hat's Automation Hub](https://console.redhat.com/ansible/automation-hub). If you don't know how to do either of
those things, please use[Red Hat's documentation](https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform/2.4/html-single/getting_started_with_automation_hub/index#proc-create-api-token)
that explains how to connect to [Red Hat's Automation Hub](https://console.redhat.com/ansible/automation-hub).
Place your Ansible configuration file either in the directory you checked out (`ansible_satellite`) as `ansible.cfg` or in your home directory with a prefixed dot: `~/.ansible.cfg`.

    Your Ansible configuration file will look similar to the example below:

    ```ini
    [galaxy]
    server_list = automation-hub,galaxy

    [galaxy_server.automation-hub]
    url=https://console.redhat.com/api/automation-hub/content/REDACTED-synclist/
    auth_url=https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token
    token=REPLACE_WITH_YOUR_TOKEN

    [galaxy_server.galaxy]
    url=https://galaxy.ansible.com
    ```

1. Install the required collections:
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-galaxy collection install -r collections/requirements.yml
    ```
    <!-- markdownlint-enable MD014 -->

1. Install the required roles:
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-galaxy role install -r collections/requirements.yml
    ```
    <!-- markdownlint-enable MD014 -->

1. Create a Manifest for Satellite - we are going to import it at a [later point](#importing-a-manifest). If you don't know how to do it, please
   follow [Red Hat's blog](https://www.redhat.com/en/blog/how-create-and-use-red-hat-satellite-manifest) that explains very well how to create a Manifest.
1. Put the `Manifest UUID` in `host_vars/<hostname>/00a_secrets.yml`. As mentioned before, we are going to need it at a [later point](#importing-a-manifest).
1. Optional (but **highly encouraged!**): You can create a [Vault password file](https://docs.ansible.com/ansible/latest/user_guide/vault.html#providing-vault-passwords) file to
   pass it to `ansible-playbook` via `--vault-pass-file`. I named mine `.vault.pass` and added it to my [`.gitignore` file](https://git-scm.com/docs/gitignore) to ensure it
   is not accidentally pushed to the repository. As you are going to need the Vault for *every* playbook of my repository, it is handy to have a Vault password file - unless
   you chose to not encrypt your secrets (**highly discouraged!**).

    :information_source: If you are not familiar with Ansible Vault, please
    [read up on it](https://docs.ansible.com/ansible/latest/vault_guide/vault_encrypting_content.html#encrypting-content-with-ansible-vault) and don't store your secrets in
    plain text in your variables files. It is really super easy to get started with Ansible Vault, I promise :slightly_smiling_face:!

## Install Satellite server

The very first step is to install the RHEL 8 host that will be our Satellite eventually.

<!-- markdownlint-disable MD024 -->
### Documentation
<!-- markdownlint-enable MD024 -->

- [Role `rhel_iso_kickstart`](https://github.com/sscheib/ansible-role-rhel_iso_kickstart)
- [Kickstart commands and options reference for RHEL 8](https://access.redhat.com/documentation/de-de/red_hat_enterprise_linux/8/html/system_design_guide/kickstart-commands-and-options-reference_system-design-guide)

<!-- markdownlint-disable MD024 -->
### Roles, variables files and playbooks
<!-- markdownlint-enable MD024 -->

:information_source: The tables below can be mapped through the numbering column (`#`).[^column_explanation]

| #   | Role/s used                                                                |
| :-- | :------------------------------------------------------------------------- |
| 1   | `rhel_iso_kickstart`                                                       |

| #   | Variable definition file/s                                                 |
| :-- | :------------------------------------------------------------------------- |
| 1   | `host_vars/<hostname>/00_kickstart.yml`                                    |

| #   | Playbook/s used                                                            |
| :-- | :------------------------------------------------------------------------- |
| 1   | `00_create_kickstart.yml`                                                  |

<!-- markdownlint-disable MD024 -->
### Procedure
<!-- markdownlint-enable MD024 -->

1. Adjust the included Kickstart (`files/satellite.ks`) to your liking.
1. Adjust the variables for the role [`rhel_iso_kickstart`](https://github.com/sscheib/ansible-role-rhel_iso_kickstart)

    :information_source: In my case, I am going to run the role [`rhel_iso_kickstart`](https://github.com/sscheib/ansible-role-rhel_iso_kickstart) on `localhost`
    (as I want to have the ISO downloaded and built on my machine), so the `host_vars` have to be placed in the directory `host_vars/localhost/`. I have chosen to place
    them in the file `00_kickstart.yml` inside that directory.

    :information_source: Make sure to read the [`README.md` of my role `rhel_iso_kickstart`](https://github.com/sscheib/ansible-role-rhel_iso_kickstart) to fully understand
    all variables.
1. Run the playbook that will download the specified RHEL ISO and build a custom RHEL ISO containing the specified Kickstart file:
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-playbook 00_create_kickstart.yml --vault-pass-file .vault.pass
    ```
    <!-- markdownlint-disable MD024 -->
1. Copy the resulting ISO file to your hypervisor and mount it to your virtual machine that should become the Satellite.
1. Start the virtual machine and wait for the installation to finish.
1. Once the installation finished, shutdown the virtual machine and unmount the ISO.

    :information_source: If you used the provided Kickstart, the virtual machine will shut down after the Kickstart finished.

## Prepare Satellite for Ansible access

Now that we have successfully installed our RHEL 8 system, we need to ensure that Ansible can access the system and **elevate its privileges**.

<!-- markdownlint-disable MD024 -->
### Procedure
<!-- markdownlint-enable MD024 -->

1. Login via SSH to the new virtual machine using `root` as username and the root password you defined in `00_kickstart.yml`

    :information_source: This step assumes that you have not already created a local user (other than `root`, of course) on the Satellite server. If you already created a
    local user as part of the Kickstart (or other means), please ensure that the user has privileged access; You can skip step **2** and **3**.

1. Create a new user for Ansible, e.g. `ansible-provisioning`.
1. Either set a password for `ansible-provisioning` or deploy your SSH key to the created user.
1. Ensure you add the user to the group `%wheel` so that you can elevate the privileges of the user.
1. Add the virtual machine host to the inventory file. Below you'll find an example:

    ```shell
    $ cat inventory
    satellite.office.int.scheib.me ansible_user=ansible-provisioning ansible_port=22
    ```

1. Ensure that your access to the virtual machine is working properly:

    ```shell
    $ ansible --become -m ping -i inventory all
    satellite.office.int.scheib.me | SUCCESS => {
        "ansible_facts": {
            "discovered_interpreter_python": "/usr/libexec/platform-python"
        },
        "changed": false,
        "ping": "pong"
    }
    ```

## Registering the system to the Red Hat Customer Portal and installing Red Hat Satellite

Now it's time to register our RHEL system to the Red Hat Customer Portal. Please ensure that the host ends up with a Satellite Infrastructure subscription (or equivalent)
as, otherwise, the Red Hat Satellite repositories cannot be found.

<!-- markdownlint-disable MD024 -->
### Documentation
<!-- markdownlint-enable MD024 -->

- [Role `redhat.rhel_system_roles.rhc`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/rhel_system_roles/docs/README_rhc/)
- [Role `redhat.satellite_operations.installer`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite_operations/content/role/installer/)
- [Registering a RHEL 8 system and managing subscriptions](https://access.redhat.com/documentation/de-de/red_hat_enterprise_linux/8/html/configuring_basic_system_settings/assembly_registering-the-system-and-managing-subscriptions_configuring-basic-system-settings)
- [Red Hat Satellite 6.12 Installation documentation](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html/installing_satellite_server_in_a_connected_network_environment/installing_server_connected_satellite#doc-wrapper)

<!-- markdownlint-disable MD024 -->
### Roles, variables files and playbooks
<!-- markdownlint-enable MD024 -->

:information_source: The tables below can be mapped through the numbering column (`#`).[^column_explanation]

| #   | Role/s used                                         |
| :-- | :-------------------------------------------------- |
| 1   | `redhat.rhel_system_roles.rhc`                      |
| 2   | `redhat.satellite_operations.installer`             |

| #   | Variable definition file/s                                                 |
| :-- | :------------------------------------------------------------------------- |
| 1   | `host_vars/<hostname>/00b_register_satellite.yml`                          |
| 2   | `host_vars/<hostname>/01a_satellite_installer_certificates.yml`            |
| ^^  | ^^ `host_vars/<hostname>/01b_satellite_firewall_rules.yml`                 |
| ^^  | ^^ `host_vars/<hostname>/01c_satellite_installer_configuration.yml`        |

| #   | Playbook/s used                                     |
| :-- | :-------------------------------------------------- |
| 1   | `01_register_satellite.yml`                         |
| 2   | `02_satellite_installer.yml`                        |

:information_source: The variable file where we put our secrets in (`host_vars/<hostname>/00a_secrets.yml`) is used in almost all roles and playbooks, and thus it isn't
added to the table above as it would need to be included in all further tables.

<!-- markdownlint-disable MD024 -->
### Procedure
<!-- markdownlint-enable MD024 -->

1. Define your secrets in `host_vars/<hostname>/00a_secrets.yml`, please see below for an example.

    :information_source: Not all variables are needed right away, but it makes sense to define all of them at once. Please read the comments in the GitHub gist below.

    {% gist 4fbc66d25da509522e16791e497a27a2 %}

1. Define the variables for the [RHEL system role `redhat.rhel_system_roles.rhc`](https://access.redhat.com/documentation/de-de/red_hat_enterprise_linux/8/html/automating_system_administration_by_using_rhel_system_roles/using-the-rhc-system-role-to-register-the-system_automating-system-administration-by-using-rhel-system-roles),
   please find below an example of my variables file (`host_vars/<hostname>/00b_register_satellite.yml`):
    {% gist 14d99b2931ce8e1bed1f6e9ac717326e %}

    :information_source: If you want to install a different Satellite version than 6.12, you need to adjust the variable `rhc_repositories` to enable the proper
    repositories for the desired Satellite version.

1. Run the playbook that registers your Satellite system to the Red Hat Content Delivery Network (`RHCDN`):
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-playbook -i inventory 01_register_satellite.yml --vault-pass-file .vault.pass
    ```
    <!-- markdownlint-enable MD014 -->
1. Next, we'll define the variables for the
   [Satellite Installer role `redhat.satellite_operations.installer`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite_operations/content/role/installer/).
   I have split the variables in three files:
    1. `host_vars/<hostname>/01a_satellite_installer_certificates.yml`: Contains variables with regards to certificates
        {% gist 7bbba923af596280573aa987180837fd %}

        :warning: When generating certificates for your Satellite, remember to set the correct `keyUsage` and `extendedKeyUsage`[^key_usage].

    1. `host_vars/<hostname>/01b_satellite_firewall_rules.yml`: Contains the ports to open on the Satellite
        {% gist 0b029863ee4e9953ab88978608bae777 %}

        :warning: Above firewall rules are specific to *my* use case. You need to adapt these based on the Satellite documentation and your used features (such as `TFTP`,
        `DHCP`, `DNS`, etc.).
    1. `host_vars/<hostname>/01c_satellite_installer_configuration.yml`: Contains all other variables for the role `redhat.satellite_operations.installer`
        {% gist c99c3008a954a4ddb81451bad99b599f %}

        :warning: If you are not planning to use certificates, please remove the certificate installer options from above example (`--certs-server-cert`,
        `--certs-server-key`, `--certs-server-ca-cert`). Don't worry, the Satellite installer will create self-signed certificates. The certificate options
        are meant for implementing custom SSL certificates that have been signed by your internal certificate authority.

1. Run the playbook to upgrade your system to the latest version (required) and install Satellite:

    :warning: This will reboot your Satellite to ensure the latest kernel is applied (as per the Satellite documentation).
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-playbook -i inventory 02_satellite_installer.yml --vault-pass-file .vault.pass
    ```
    <!-- markdownlint-enable MD014 -->
1. Verify that you can login to your Satellite instance once the playbook finished
1. Shutdown the virtual machine and take a snapshot so you can roll back to this point when you encounter any issues later on

## Define general variables

<!-- markdownlint-disable MD024 -->
### Documentation
<!-- markdownlint-enable MD024 -->

- [Satellite Collection Common Role Variables](https://github.com/theforeman/foreman-ansible-modules/blob/develop/README.md#common-role-variables)

  :information_source: The upstream common role variables are *not* used downstream in the Satellite collection. All `foreman_` variables can be substituted with
  `satellite_`, e.g. `foreman_url` -> `satellite_url`

<!-- markdownlint-disable MD024 -->
### Roles, variables files and playbooks
<!-- markdownlint-enable MD024 -->

:information_source: The tables below can be mapped through the numbering column (`#`).[^column_explanation]

| #   | Role/s used                                                                |
| :-- | :------------------------------------------------------------------------- |
| 1   | Used in almost all roles                                                   |

| #   | Variable definition file/s                                                 |
| :-- | :------------------------------------------------------------------------- |
| 1   | `host_vars/<hostname>/02_general.yml`                                      |

| #   | Playbook/s used                                                            |
| :-- | :------------------------------------------------------------------------- |
| 1   | Used in almost all playbooks                                               |

<!-- markdownlint-disable MD024 -->
### Procedure
<!-- markdownlint-enable MD024 -->

1. With this, we define variables that are reused multiple times. To avoid duplication, it is good to define them once. Please see below an example of my definition:
    {% gist 1bb3a400efe20cb7a07a7b64730ad1d2 %}

    :warning: If you know that your Linux host is slow in synchronizing the Repositories or publishing and promoting (Composite) Content Views (due to e.g. slow storage),
    consider significantly raising the value of `sat_repository_sync_retries`, `sat_content_view_publish_retries` and `sat_composite_content_view_publish_retries`, as
    otherwise the respective tasks (and thus the playbook) will fail. The status is checked every `3` seconds, so the time is (roughly) calculated as `3 * number_of_retries`.
    In this case it would be `3 * 3600 = 10800 seconds = 180 minutes = 3 hours`.

## *Optional*: Configure Satellite's Cloud Connector

:information_source: This step is entirely optional, as it merely configures Satellite's Cloud Connector.

:warning: Please make sure that you really **want** to have the Cloud Connector enabled. The Cloud Connector allows Red Hat Insights to **run remediation on your Satellite**
from the [Red Hat Hybrid Cloud Console](https://console.redhat.com).

<!-- markdownlint-disable MD024 -->
### Documentation
<!-- markdownlint-enable MD024 -->

- [Role `redhat.satellite_operations.cloud_connector`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite_operations/content/role/cloud_connector/)
- [Red Hat Insights Remediations Guide](https://access.redhat.com/documentation/en-us/red_hat_insights/2023/html-single/red_hat_insights_remediations_guide/index)
- [Enabling Cloud Connector on hosts managed by Satellite](https://access.redhat.com/documentation/en-us/red_hat_insights/2023/html-single/red_hat_insights_remediations_guide/index#configuring-satellite-cloud-connector_host-communication-with-insights)

<!-- markdownlint-disable MD024 -->
### Roles, variables files and playbooks
<!-- markdownlint-enable MD024 -->

:information_source: The tables below can be mapped through the numbering column (`#`).[^column_explanation]

| #   | Role/s used                                                                |
| :-- | :------------------------------------------------------------------------- |
| 1   | `redhat.satellite_operations.cloud_connector`                              |

| #   | Variable definition file/s                                                 |
| :-- | :------------------------------------------------------------------------- |
| 1   | `host_vars/<hostname>/03_cloud_connector.yml`                              |

| #   | Playbook/s used                                                            |
| :-- | :------------------------------------------------------------------------- |
| 1   | `03_satellite_cloud_connector.yml`                                         |

1. Define the variables for the role `redhat.satellite_operations.cloud_connector`. Please see an example below:
    {% gist 64a79193346520cb437930b4497c7f7d %}
1. Run the playbook to configure Satellite's Cloud Connector:
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-playbook -i inventory 03_satellite_cloud_connector.yml --vault-pass-file .vault.pass
    ```
    <!-- markdownlint-enable MD014 -->

## Importing a Manifest

Next up, we are going to import a Manifest into our Satellite. I have configured the role `redhat.satellite.manifest` to first download a Manifest from the Red Hat Customer
Portal and then upload it to my Satellite. If you are doing the same, please ensure that you set the correct `Manifest UUID` in `satellite_manifest_uuid`, as well as the
`satellite_rhsm_username` and `satellite_rhsm_password`. (as described in
[Registering the system to the Red Hat Customer Portal and installing Red Hat Satellite](#registering-the-system-to-the-red-hat-customer-portal-and-installing-red-hat-satellite)).

<!-- markdownlint-disable MD024 -->
### Documentation
<!-- markdownlint-enable MD024 -->

- [Role `redhat.satellite.manifest`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/manifest/)
- [How to create and use a Red Hat Satellite manifest](https://www.redhat.com/en/blog/how-create-and-use-red-hat-satellite-manifest)
- [Importing a Red Hat Subscription Manifest into Satellite Server](https://access.redhat.com/documentation/en-us/red_hat_satellite/6.14/html/managing_content/managing_red_hat_subscriptions_content-management#Importing_a_Red_Hat_Subscription_Manifest_into_Server_content-management)

<!-- markdownlint-disable MD024 -->
### Roles, variables files and playbooks
<!-- markdownlint-enable MD024 -->

:information_source: The tables below can be mapped through the numbering column (`#`).[^column_explanation]

| #   | Role/s used                                                                |
| :-- | :------------------------------------------------------------------------- |
| 1   | `redhat.satellite.manifest`                                                |

| #   | Variable definition file/s                                                 |
| :-- | :------------------------------------------------------------------------- |
| 1   | `host_vars/<hostname>/04_manifest.yml`                                     |

| #   | Playbook/s used                                                            |
| :-- | :------------------------------------------------------------------------- |
| 1   | `04_satellite_manifest.yml`                                                |

1. Adjust the variables for this step in `host_vars/<hostname>/04_manifest.yml`, please see below for an example:
    {% gist 5e8b980306c43281971f6f8976c9a351 %}
1. Run the playbook to download the manifest and import it to your Satellite:
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-playbook -i inventory 04_satellite_manifest.yml --vault-pass-file .vault.pass
    ```
    <!-- markdownlint-enable MD014 -->

## Creating Content Credentials for custom Products

Before creating custom Products and repositories, we need to create the custom Content Credentials. This is required, as we are going to use those Content Credentials to
ensure that the content we retrieve is retrieved without any modification (through a malicious actor for instance). This is only required for 3rd-party repositories, as
Red Hat repositories are validated by default.

<!-- markdownlint-disable MD024 -->
### Documentation
<!-- markdownlint-enable MD024 -->

- [Role `redhat.satellite.content_credentials`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/content_credentials/)
- [Importing Custom SSL Certificates](https://access.redhat.com/documentation/en-us/red_hat_satellite/6.14/html/managing_content/importing_content_content-management#Importing_Custom_SSL_Certificates_content-management)

<!-- markdownlint-disable MD024 -->
### Roles, variables files and playbooks
<!-- markdownlint-enable MD024 -->

:information_source: The tables below can be mapped through the numbering column (`#`).[^column_explanation]

| #   | Role/s used                                                                |
| :-- | :------------------------------------------------------------------------- |
| 1   | `redhat.satellite.content_credentials`                                     |

| #   | Variable definition file/s                                                 |
| :-- | :------------------------------------------------------------------------- |
| 1   | `host_vars/<hostname>/05_content_credentials.yml`                          |

| #   | Playbook/s used                                                            |
| :-- | :------------------------------------------------------------------------- |
| 1   | `05_satellite_content_credentials.yml`                                     |

<!-- markdownlint-disable MD024 -->
### Procedure
<!-- markdownlint-enable MD024 -->

1. Define the required Content Credentials for any custom Product in `host_vars/<hostname>/05_content_credentials.yml`. Please see an example of my Content Credentials below:
    {% gist b573c065923442511509c519938b6f89 %}

1. Run the playbook to create the Content Credentials:
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-playbook -i inventory 05_satellite_content_credentials.yml --vault-pass-file .vault.pass
    ```
    <!-- markdownlint-enable MD014 -->

## Enabling Red Hat Repositories, creating custom Products and Repositories and synchronize the Repositories

Now that we have created the Content Credentials for our custom Products, we can start creating the custom Products and their containing Repositories. Further, we are going
to enable the official Red Hat Repositories that we require. Conveniently, this can all be done by one role, so we are doing it in one go.

Since we do a couple of things with one role, I decided to split up my variables into Custom Products (`host_vars/<hostname>/06a_products.yml`) and Red Hat
Products (`host_vars/<hostname>/06a_products.yml`). Lastly, we combine the two definitions into one variable which is stored as `satellite_repositories`: That is the
variable the role `redhat.satellite.repositories` requires.

:information_source: Unfortunately, this step requires an already installed Satellite. I am not aware of any other method of finding out the Product and Repository names.
If you know another way, kindly let me know in the comments below.

<!-- markdownlint-disable MD024 -->
### Documentation
<!-- markdownlint-enable MD024 -->

- [Role `redhat.satellite.repositories`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/repositories/)
- [Introduction to Content Management](https://access.redhat.com/documentation/en-us/red_hat_satellite/6.14/html/managing_content/introduction_to_content_management_content-management)
- [Download Policies Overview](https://access.redhat.com/documentation/en-us/red_hat_satellite/6.14/html/managing_content/importing_content_content-management#Download_Policies_Overview_content-management)

<!-- markdownlint-disable MD024 -->
### Roles, variables files and playbooks
<!-- markdownlint-enable MD024 -->

:information_source: The tables below can be mapped through the numbering column (`#`).[^column_explanation]

| #   | Role/s used                                                                |
| :-- | :------------------------------------------------------------------------- |
| 1   | `redhat.satellite.repositories`                                            |

| #   | Variable definition file/s                                                 |
| :-- | :------------------------------------------------------------------------- |
| 1   | `host_vars/<hostname>/06a_products.yml`                                    |
| ^^  | ^^ `host_vars/<hostname>/06b_custom_products.yml`                          |
| ^^  | ^^ `host_vars/<hostname>/06c_combined_products.yml`                        |

| #   | Playbook/s used                                                            |
| :-- | :------------------------------------------------------------------------- |
| 1   | `06_satellite_products_and_repositories.yml`                               |
| 1   | `07_satellite_sync_repositories.yml`                                       |

<!-- markdownlint-disable MD024 -->
### Procedure
<!-- markdownlint-enable MD024 -->

1. Finding out Red Hat Repository and Product names

    This step requires an already installed Satellite (to my knowledge), because you need to find out two things:
      - The Product name a Repository is tied to
      - The Repository name itself

    The Repository and Product names can be found out via the following procedure:
    1. `Satellite WebUI` -> `Content` -> `Red Hat Repositories`
    1. Click on `Filter by Product` and select a Product, e.g. `Red Hat Enterprise Linux for x86_64` (this is the **Product Name** to use)
    1. Select the appropriate "Type" (right hand side to the `Filter by Product` drop-down), e.g. `RPM` for `RPM` repositories, `Kickstart` for Kickstart repositories
    1. Either scroll through the `Available Repositories` or narrow the list further down using the search bar at the top
    1. The search lists contains the **Repository Name** to use
    1. Each Repository has a arrow an the left hand side. When clicking on it, it reveals two things:
      - The architecture (e.g. `x86_64`)
      - The release (which is optional and thus not available for every repository), (e.g. `8`, `8.7`, etc.)
1. Define the Red Hat Repositories and Products in `host_vars/<hostname>/06a_products.yml`
    {% gist 1ab1bbd6cc74c1f1fb47ea16c5e054ef %}

    :warning: The Kickstart Repositories are set to the `download_policy` `immediate` as having them set to `on_demand` is known to cause issues during provisioning.

    :warning: I recommend setting the `download_policy` to `immediate` on custom upstream repositories, as usually the upstream repositories do **not** keep **all
    versions** of an `RPM`, which will cause issues if a client tries to install an older version. Red Hat keeps all `RPMs` in their respective repositories, that's why it is
    not required to set them on `immediate`.

1. Define the custom Products and their containing Repositories in `host_vars/<hostname>/06b_custom_products.yml`
    {% gist f483a67504e10ef76189ea5a921dcecc %}
1. The two definition will be merged in `host_vars/<hostname>/06c_combined_products.yml`
    {% gist ea886810cd22867a096eb2d3f96a48c5 %}
1. Run the playbook to enable and/or create the custom Products and Repositories
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-playbook -i inventory 06_satellite_products_and_repositories.yml --vault-pass-file .vault.pass
    ```
    <!-- markdownlint-enable MD014 -->
1. Run the playbook to synchronize the Repositories (both custom and Red Hat)

    :information_source: This might take a while.

    :warning: If you'd like to follow along with the blog post, you *have* to run this step, as otherwise the Content Views *will* be *empty*!
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-playbook -i inventory 07_satellite_sync_repositories.yml --vault-pass-file .vault.pass
    ```
    <!-- markdownlint-enable MD014 -->

## Create Sync Plans

To ensure our content is kept up to date, we are going to create Sync Plans in the next step.

<!-- markdownlint-disable MD024 -->
### Documentation
<!-- markdownlint-enable MD024 -->

- [Role `redhat.satellite.sync_plans`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/sync_plans/)
- [Creating a Sync Plan](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html-single/managing_content/index#Creating_a_Sync_Plan_content-management)
- [Assigning a Sync Plan to a Product](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html-single/managing_content/index#Assigning_a_Sync_Plan_to_a_Product_content-management)

<!-- markdownlint-disable MD024 -->
### Roles, variables files and playbooks
<!-- markdownlint-enable MD024 -->

:information_source: The tables below can be mapped through the numbering column (`#`).[^column_explanation]

| #   | Role/s used                                                                |
| :-- | :------------------------------------------------------------------------- |
| 1   | `redhat.satellite.sync_plans`                                              |

| #   | Variable definition file/s                                                 |
| :-- | :------------------------------------------------------------------------- |
| 1   | `host_vars/<hostname>/07_sync_plans.yml`                                   |

| #   | Playbook/s used                                                            |
| :-- | :------------------------------------------------------------------------- |
| 1   | `08_satellite_sync_plans.yml`                                              |

<!-- markdownlint-disable MD024 -->
### Procedure
<!-- markdownlint-enable MD024 -->

1. Define the Sync Plans in `host_vars/<hostname>/07_sync_plans.yml`
    {% gist 00a3ab9f03266ba08e80cc756d43ecc0 %}

    :information_source: I am reusing the definition of the Products we have done earlier to map all products to one Sync Plan.

1. Run the playbook to create the Sync Plans:
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-playbook -i inventory 08_satellite_sync_plans.yml --vault-pass-file .vault.pass
    ```
    <!-- markdownlint-enable MD014 -->

## Create Lifecycle Environments

When it comes to Lifecycle Environments I have a *very* simple use case. I have two Lifecycle Environments and that's it. Your use case might be much more complex, but
the example below should get you up to speed quickly.

<!-- markdownlint-disable MD024 -->
### Documentation
<!-- markdownlint-enable MD024 -->

- [Role `redhat.satellite.lifecycle_environments`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/lifecycle_environments/)
- [Managing Application Life Cycles](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html/managing_content/creating_an_application_life_cycle_content-management)

<!-- markdownlint-disable MD024 -->
### Roles, variables files and playbooks
<!-- markdownlint-enable MD024 -->

:information_source: The tables below can be mapped through the numbering column (`#`).[^column_explanation]

| #   | Role/s used                                                                |
| :-- | :------------------------------------------------------------------------- |
| 1   | `redhat.satellite.lifecycle_environments`                                  |

| #   | Variable definition file/s                                                 |
| :-- | :------------------------------------------------------------------------- |
| 1   | `host_vars/<hostname>/08_lifecycle_environments.yml`                       |

| #   | Playbook/s used                                                            |
| :-- | :------------------------------------------------------------------------- |
| 1   | `09_satellite_lifecycle_environments.yml`                                  |

<!-- markdownlint-disable MD024 -->
### Procedure
<!-- markdownlint-enable MD024 -->

1. Define the Lifecycle Environment in `host_vars/<hostname>/08_lifecycle_environments.yml`. Please see below for an example:
    {% gist 3dfd83aa5d7231d602b715316ce1f419 %}
1. Run the playbook to create the Lifecycle Environments:
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-playbook -i inventory 09_satellite_lifecycle_environments.yml --vault-pass-file .vault.pass
    ```
    <!-- markdownlint-enable MD014 -->

## Create Domains

This one is super easy for my use case as well. I have exactly **one** Domain I am using internally for my lab environment: `office.int.scheib.me`. Since this is
conveniently also the value of the initial Domain I have specified, I am just reusing it. Your use case might be much more complex; This example can easily be extended.

<!-- markdownlint-disable MD024 -->
### Documentation
<!-- markdownlint-enable MD024 -->

- [Role `redhat.satellite.domains`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/domains/)

<!-- markdownlint-disable MD024 -->
### Roles, variables files and playbooks
<!-- markdownlint-enable MD024 -->

:information_source: The tables below can be mapped through the numbering column (`#`).[^column_explanation]

| #   | Role/s used                                                                |
| :-- | :------------------------------------------------------------------------- |
| 1   | `redhat.satellite.domains`                                                 |

| #   | Variable definition file/s                                                 |
| :-- | :------------------------------------------------------------------------- |
| 1   | `host_vars/<hostname>/09_domains.yml`                                      |

| #   | Playbook/s used                                                            |
| :-- | :------------------------------------------------------------------------- |
| 1   | `10_satellite_domains.yml`                                                 |

<!-- markdownlint-disable MD024 -->
### Procedure
<!-- markdownlint-enable MD024 -->

1. Define the Domains in `host_vars/<hostname>/09_domains.yml`
    {% gist 6f33448c5ef7750f10c79d88cd301ed7 %}
1. Run the playbook to create the Domains:
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-playbook -i inventory 10_satellite_domains.yml --vault-pass-file .vault.pass
    ```
    <!-- markdownlint-enable MD014 -->

## Create Subnets

When it comes to Subnets it's the same as for the Domains: I have a super easy use case. One Subnet. That's it :grin:

Again, your use case might be much, much more complex, but the example below should get you up to speed quickly.

<!-- markdownlint-disable MD024 -->
### Documentation
<!-- markdownlint-enable MD024 -->

- [Role `redhat.satellite.subnets`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/subnets/)

<!-- markdownlint-disable MD024 -->
### Roles, variables files and playbooks
<!-- markdownlint-enable MD024 -->

:information_source: The tables below can be mapped through the numbering column (`#`).[^column_explanation]

| #   | Role/s used                                                                |
| :-- | :------------------------------------------------------------------------- |
| 1   | `redhat.satellite.subnets`                                                 |

| #   | Variable definition file/s                                                 |
| :-- | :------------------------------------------------------------------------- |
| 1   | `host_vars/<hostname>/10_subnets.yml`                                      |

| #   | Playbook/s used                                                            |
| :-- | :------------------------------------------------------------------------- |
| 1   | `11_satellite_subnets.yml`                                                 |

<!-- markdownlint-disable MD024 -->
### Procedure
<!-- markdownlint-enable MD024 -->

1. Define the Subnets in `host_vars/<hostname>/10_subnets.yml`
    {% gist 85c7a7082456159c05b6e8850f5a54a9 %}
1. Run the playbook to create the Subnets:
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-playbook -i inventory 11_satellite_subnets.yml --vault-pass-file .vault.pass
    ```
    <!-- markdownlint-enable MD014 -->

## Define the (Composite) Content Views and publish and promote them

We are now going to automate one of the more complex things: Content Views and Composite Content Views. Since Composite Content Views will contain `components`
(which are Content View Versions), they are dependent on the Content Views being present at the time we are going to create the Composite Content Views. So, the order
really matters!

As the order matters, I am splitting again the Content Views in the `host_vars` in to several files:

1. `host_vars/<hostname>/11a_content_views_custom_products.yml`: Contains Content Views with Custom Products we have created earlier
1. `host_vars/<hostname>/11b_content_views.yml`: Contains Content Views which use official Red Hat Products
1. `host_vars/<hostname>/11c_composite_content_views.yml`: Contains Composite Content Views (which depend on the creation of the earlier defined Content Views)
1. `host_vars/<hostname>/11d_combined_content_views.yml`: Lastly, I merge the definitions of the aforementioned (Composite) Content Views into one variable, which is
   used by the role `redhat.satellite.content_views`: `satellite_content_views`

<!-- markdownlint-disable MD024 -->
### Documentation
<!-- markdownlint-enable MD024 -->

- [Role `redhat.satellite.content_views`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/content_views/)
- [Managing Content Views](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html/managing_content/managing_content_views_content-management)

<!-- markdownlint-disable MD024 -->
### Roles, variables files and playbooks
<!-- markdownlint-enable MD024 -->

:information_source: The tables below can be mapped through the numbering column (`#`).[^column_explanation]

| #   | Role/s used                                                                |
| :-- | :------------------------------------------------------------------------- |
| 1   | `redhat.satellite.content_views`                                           |

| #   | Variable definition file/s                                                 |
| :-- | :------------------------------------------------------------------------- |
| 1   | `host_vars/<hostname>/11a_content_views_custom_products.yml`               |
| ^^  | ^^ `host_vars/<hostname>/11b_content_views.yml`                            |
| ^^  | ^^ `host_vars/<hostname>/11c_composite_content_views.yml`                  |
| ^^  | ^^ `host_vars/<hostname>/11d_combined_content_views.yml`                   |

| #   | Playbook/s used                                                            |
| :-- | :------------------------------------------------------------------------- |
| 1   | `12_satellite_content_views.yml`                                           |

<!-- markdownlint-disable MD024 -->
### Procedure
<!-- markdownlint-enable MD024 -->

1. Define the Content Views with custom Products in `host_vars/<hostname>/11a_content_views_custom_products.yml`
    {% gist 12fd843551041ed98632060d72b9dd60 %}
1. Define the Content Views with Red Hat Products in `host_vars/<hostname>/11b_content_views.yml`
    {% gist 50c5c2fee3238b47e2cee256ad488078 %}
1. Define the Composite Content Views in `host_vars/<hostname>/11c_composite_content_views.yml`
    {% gist 577f57b8b67b5accc6436ce21b69b62c %}
1. The definitions of 1. to 3. will be merged in `host_vars/<hostname>/11d_combined_content_views.yml`
    {% gist 8df9e83c3c21836afc83af093e34a3db %}
1. Run the playbook to create the (Composite) Content Views:
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-playbook -i inventory 12_satellite_content_views.yml --vault-pass-file .vault.pass
    ```
    <!-- markdownlint-enable MD014 -->
1. Run the playbook to publish and promote the (Composite) Content Views
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-playbook -i inventory 13_satellite_content_view_publish.yml --vault-pass-file .vault.pass
    ```
    <!-- markdownlint-enable MD014 -->

## Apply Satellite settings and enable Template Synchronization

We need to apply some settings in Satellite, as the Operating System definitions depend on Provision Templates that have to be imported prior to defining the Operating Systems.
The Host Groups and the Activation Keys (which we still need to create) rely on the definitions of the Operating System, thus we need to start with importing the Templates.

I make use of [Satellite's `TemplateSync` Plug-in](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html/managing_hosts/synchronizing_templates_repositories_managing-hosts#doc-wrapper)
which allows us to import Templates from a Source Code Management (`SCM`) tool (such as GitHub or Git Lab). `TemplateSync` will also assign the Templates to the correct Operating
System, Organization and Location, if the
[required metadata](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html/managing_hosts/synchronizing_templates_repositories_managing-hosts#Importing_Templates_managing-hosts)
is present in the Template's header.

Again, I have split my variables into several files to not have one very large confusing file. I named the files like the tabs in the
`Satellite Web UI` -> `Administer` -> `Settings`.

This chapter essentially covers two steps:

1. Set specific settings in the Satellite, especially the Template Sync settings. But since we are on it, I'll define *all* settings right away to spare me some time.
1. Run the Template Sync to import the Templates of my [Kickstart Repository on GitHub](https://github.com/sscheib/satellite_templates). The playbook will
perform [the steps that are required to synchronize Templates](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html/managing_hosts/synchronizing_templates_repositories_managing-hosts#Synchronizing_Templates_Using_the_API_managing-hosts)
from git. Please review them prior to running the playbook (`15_satellite_template_sync.yml`).

    :warning: You obviously *have* to either fork [my Satellite template repository](https://github.com/sscheib/satellite_templates) or create your own GitHub repository
    with your Templates to be able to create a GitHub Token (which will be done in this chapter).

    :information_source: Unfortunately, there is no role to configure and run the Template Sync. This will be done by a custom playbook I have written and importing the
    Templates using the Satellite API. It is a playbook, as I didn't see the need to create a role for such a basic task.

<!-- markdownlint-disable MD024 -->
### Documentation
<!-- markdownlint-enable MD024 -->

- [Role `redhat.satellite.settings`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/settings/)
- [Synchronizing Template Repositories](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html/managing_hosts/synchronizing_templates_repositories_managing-hosts#doc-wrapper)
- [GitHub: Managing your personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#fine-grained-personal-access-tokens)
- [Kickstarting Red Hat Enterprise Linux (RHEL) systems using a highly customized Kickstart with Red Hat Satellite 6](https://blog.scheib.me/2023/07/01/highly-customized-kickstart.html)

<!-- markdownlint-disable MD024 -->
### Roles, variables files and playbooks
<!-- markdownlint-enable MD024 -->

:information_source: The tables below can be mapped through the numbering column (`#`).[^column_explanation]

:information_source: `12a_settings_template_sync.yml` is added twice, as both playbooks make use of it.

| #   | Role/s used                                                                |
| :-- | :------------------------------------------------------------------------- |
| 1   | `redhat.satellite.settings`                                                |
| 2   | N/A                                                                        |

| #   | Variable definition file/s                                                 |
| :-- | :------------------------------------------------------------------------- |
| 1   | `12a_settings_template_sync.yml`                                           |
| ^^  | ^^ `12b_settings_provisioning.yml`                                         |
| ^^  | ^^ `12c_settings_remote_execution.yml`                                     |
| ^^  | ^^ `12d_settings_rh_cloud.yml`                                             |
| ^^  | ^^ `12e_settings_general.yml`                                              |
| ^^  | ^^ `12f_settings_notifications.yml`                                        |
| ^^  | ^^ `12g_merge_settings.yml`                                                |
| 2   | `12a_settings_template_sync.yml`                                           |
| ^^  | ^^ `group_vars/all/github.yml`                                             |

| #   | Playbook/s used                                                            |
| :-- | :------------------------------------------------------------------------- |
| 1   | `14_satellite_settings.yml`                                                |
| 2   | `15_satellite_template_sync.yml`                                           |

<!-- markdownlint-disable MD024 -->
### Procedure
<!-- markdownlint-enable MD024 -->

1. Define the Satellite settings in our variables files:
    1. **Tab Template Sync**: Variables file: `12a_settings_template_sync.yml`:
        {% gist bf5474d6cd2b41fcf237a515bc92a829 %}
    1. **Tab Provisioning**: Variables file: `12b_settings_provisioning.yml`:
        {% gist cb31468e436df3b8ce344d7c5376baf3 %}
    1. **Tab Remote Execution:** Variables file: `12c_settings_remote_execution.yml`:
        {% gist fe1d73bc8c895a7517370f403f4346d1 %}
    1. **Tab RH Cloud:** Variables file: `12d_settings_rh_cloud.yml`:
        {% gist 4b56a2e775b67c6d442ffc719b4b96d1 %}
    1. **Tab General**: Variables file: `12e_settings_general.yml`
        {% gist 0b57b779e5a944a77fee3ab0ebaa02a7 %}
    1. **Tab Notifications**: Variables file: `12f_settings_notifications.yml`
        {% gist 7842c40f0833abb64465061daeafa332 %}
    1. All settings are merged into one variable (`satellite_settings`) which is the variable that is used by the role `redhat.satellite.settings` in the file `12g_merge_settings.yml`:
        {% gist c9fd9deb61e6d39749ec9ef02073bf26 %}

1. Run the playbook to apply the settings:
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-playbook -i inventory 14_satellite_settings.yml --vault-pass-file .vault.pass
    ```
    <!-- markdownlint-enable MD014 -->

1. Create a
   [(Fine Grained) Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#fine-grained-personal-access-tokens)
   in GitHub with the following permissions (at least):

    - `Administration: Read and Write` (to deploy the SSH key from the Foreman user that performs the Template Synchronization)
    - `Contents: Read-only`
    - `Metadata: Read-only`

    :warning: Of course, you can deploy the SSH key manually and modify the playbook that configures the Template Sync (`15_satellite_template_sync.yml`) to not deploy the
    SSH key to GitHub if you are uncomfortable with creating a GitHub Access Token that has the *Administration* option set

1. Define the GitHub settings for the Template Sync in `group_vars/all/github.yml`
    {% gist 21475840ab5b6a90b263a87d9ee6bf70 %}
1. Run the playbook to configure the Template Sync and synchronize the Templates:
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-playbook -i inventory 15_satellite_template_sync.yml --vault-pass-file .vault.pass
    ```
    <!-- markdownlint-enable MD014 -->

## Create Operating Systems

Since the Templates are now imported, we can start creating the Operating Systems and assign the imported Templates to them.

<!-- markdownlint-disable MD024 -->
### Documentation
<!-- markdownlint-disable MD024 -->

- [Role `redhat.satellite.operatingsystems`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/operatingsystems/)
- [Creating Operating Systems](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html-single/provisioning_hosts/index#creating-operating-systems_provisioning)

<!-- markdownlint-disable MD024 -->
### Roles, variables files and playbooks
<!-- markdownlint-disable MD024 -->

:information_source: The tables below can be mapped through the numbering column (`#`).[^column_explanation]

| #   | Role/s used                                                                |
| :-- | :------------------------------------------------------------------------- |
| 1   | `redhat.satellite.operatingsystems`                                        |

| #   | Variable definition file/s                                                 |
| :-- | :------------------------------------------------------------------------- |
| 1   | `host_vars/<hostname>/13_operating_systems.yml`                            |

| #   | Playbook/s used                                                            |
| :-- | :------------------------------------------------------------------------- |
| 1   | `16_satellite_operating_systems.yml`                                       |

<!-- markdownlint-disable MD024 -->
### Procedure
<!-- markdownlint-disable MD024 -->

1. Define the Operating Systems in `host_vars/<hostname>/13_operating_systems.yml`:
    {% gist 031b18a1b51d901f088e88d8f779dc26 %}
1. Run the playbook to create and configure the Operating Systems:
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-playbook -i inventory 16_satellite_operating_systems.yml --vault-pass-file .vault.pass
    ```
    <!-- markdownlint-enable MD014 -->

## Create Activation Keys

Creating Activation Keys is now possible, as we can refer to the Operating Systems that we created in the prior chapter.

<!-- markdownlint-disable MD024 -->
### Documentation
<!-- markdownlint-enable MD024 -->

- [Role `redhat.satellite.activation_keys`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/activation_keys/)
- [Managing Activation Keys](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html/managing_content/managing_activation_keys_content-management)

<!-- markdownlint-disable MD024 -->
### Roles, variables files and playbooks
<!-- markdownlint-enable MD024 -->

:information_source: The tables below can be mapped through the numbering column (`#`).[^column_explanation]

| #   | Role/s used                                                                |
| :-- | :------------------------------------------------------------------------- |
| 1   | `redhat.satellite.activation_keys`                                         |

| #   | Variable definition file/s                                                 |
| :-- | :------------------------------------------------------------------------- |
| 1   | `host_vars/<hostname>/14_activation_keys.yml`                              |

| #   | Playbook/s used                                                            |
| :-- | :------------------------------------------------------------------------- |
| 1   | `17_satellite_activation_keys.yml`                                         |

<!-- markdownlint-disable MD024 -->
### Procedure
<!-- markdownlint-enable MD024 -->

1. Define the Activation Keys in `host_vars/<hostname>/14_activation_keys.yml`
    {% gist 585614ff9d774fb064ce6b586f48efc8 %}
1. Run the playbook to create the Activation Keys:
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-playbook -i inventory 17_satellite_activation_keys.yml --vault-pass-file .vault.pass
    ```
    <!-- markdownlint-enable MD014 -->

## Create Host Groups

Almost done! :sunglasses:

As our second to last step, we are going to create the Host Groups. Since I have
applied [my Satellite 6 concept for Host Groups](https://blog.scheib.me/2023/05/30/redhat-satellite-concept.html#host-groups-hg), I have split the Host Groups into
multiple files.

<!-- markdownlint-disable MD024 -->
### Documentation
<!-- markdownlint-enable MD024 -->

- [Role `redhat.satellite.hostgroups`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/hostgroups/)
- [Creating a Host Group](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html/managing_hosts/administering_hosts_managing-hosts#Creating_a_Host_Group_managing-hosts)

<!-- markdownlint-disable MD024 -->
### Roles, variables files and playbooks
<!-- markdownlint-enable MD024 -->

:information_source: The tables below can be mapped through the numbering column (`#`).[^column_explanation]

| #   | Role/s used                                                                |
| :-- | :------------------------------------------------------------------------- |
| 1   | `redhat.satellite.hostgroups`                                              |

| #   | Variable definition file/s                                                 |
| :-- | :------------------------------------------------------------------------- |
| 1   | `host_vars/<hostname>/15a_host_groups_base.yml`                            |
| ^^  | ^^ `host_vars/<hostname>/15b_host_groups_services.yml`                     |
| ^^  | ^^ `host_vars/<hostname>/15c_host_groups_lifecycle_environments.yml`       |
| ^^  | ^^ `host_vars/<hostname>/15d_host_groups_merged.yml`                       |

| #   | Playbook/s used                                                            |
| :-- | :------------------------------------------------------------------------- |
| 1   | `18_satellite_host_groups.yml`                                             |

<!-- markdownlint-disable MD024 -->
### Procedure
<!-- markdownlint-enable MD024 -->

1. Define the Host Groups
    1. **Base** and **Operating System level Host Groups**: `host_vars/<hostname>/15a_host_groups_base.yml`
        {% gist c0ca37c45ec9ca780b2f7eff17fef984 %}
    1. **Service level Host Groups**: `host_vars/<hostname>/15b_host_groups_services.yml`
        {% gist d06b236f5f5ff5dc15abbe54d6174ec1 %}
    1. **Lifecycle Environment level Host Groups**: `host_vars/<hostname>/15c_host_groups_lifecycle_environments.yml`
        {% gist 46776747718e35681b94681f68bfdfe9 %}
    1. Finally, merge all Host Groups into the required variable `satellite_hostgroups` for the role `redhat.satellite.hostgroups`; That is done in the
       file `host_vars/<hostname>/15d_host_groups_merged.yml`
        {% gist 9dd2cd08399d49dad14a3637de88f935 %}
1. Run the playbook to create the Host Groups:
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-playbook -i inventory 18_satellite_host_groups.yml --vault-pass-file .vault.pass
    ```
    <!-- markdownlint-enable MD014 -->

## Create Global Parameters

Now on the the Global Parameters. I will define some Global Parameters that are specific to my use case, which
[my Provisioning Templates](https://github.com/sscheib/satellite_templates) make use of as well as override some default Parameters. My parameters are denoted with a
prefixed `p-`. All Parameters which do not have that prefix are Parameters that are present by default.

Once again, I split up my variables into separate files to not have a big clunky file that is unmaintainable. :grin:

:information_source: Unfortunately, there is no role available that creates Global Parameters, but there is a module available which we can use:
`redhat.satellite.global_parameter`.

<!-- markdownlint-disable MD024 -->
### Documentation
<!-- markdownlint-enable MD024 -->

- [Module `redhat.satellite.global_parameter`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/module/global_parameter/)

<!-- markdownlint-disable MD024 -->
### Roles, variables files and playbooks
<!-- markdownlint-enable MD024 -->

:information_source: The tables below can be mapped through the numbering column (`#`).[^column_explanation]

| #   | Role/s used                                                                |
| :-- | :------------------------------------------------------------------------- |
| 1   | N/A                                                                        |

| #   | Variable definition file/s                                                 |
| :-- | :------------------------------------------------------------------------- |
| 1   | `host_vars/<hostname>/16a_global_parameters_ansible_provisioning.yml`      |
| ^^  | ^^ `host_vars/<hostname>/16b_global_parameters_remote_execution.yml`       |
| ^^  | ^^ `host_vars/<hostname>/16c_global_parameters_general.yml`                |
| ^^  | ^^ `host_vars/<hostname>/16d_merge_global_parameters.yml`                  |

| #   | Playbook/s used                                                            |
| :-- | :------------------------------------------------------------------------- |
| 1   | `19_satellite_global_parameters.yml`                                       |

<!-- markdownlint-disable MD024 -->
### Procedure
<!-- markdownlint-enable MD024 -->

1. Define the Global Parameters
    1. Global Parameters for Ansible post provisioning: `host_vars/<hostname>/16a_global_parameters_ansible_provisioning.yml`
        {% gist ccacbc6d834959ab0a18898af41c3546 %}
    1. Global Parameters for Remote Execution: `host_vars/<hostname>/16b_global_parameters_remote_execution.yml`
        {% gist 9a54f56b9a66f99473d49fe61d16e322 %}
    1. General Global Parameters: `host_vars/<hostname>/16c_global_parameters_general.yml`
        {% gist 581f59a607c385a52e7750f93261a379 %}
    1. Merge all Global Parameters: `host_vars/<hostname>/16d_merge_global_parameters.yml`
        {% gist 594c9981cfde05e3db15e24bd3cb28fc %}
1. Run the playbook to create the Global Parameters:
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-playbook -i inventory 19_satellite_global_parameters.yml --vault-pass-file .vault.pass
    ```
    <!-- markdownlint-disable MD014 -->

## *Optional:* Enable and import OpenSCAP content

If you are looking to ensure compliance on your hosts using [OpenSCAP](https://www.open-scap.org/), this chapter will be for you. Here is the thing:
Satellite comes already preloaded with some OpenSCAP content. I have written a playbook that will accommodate the two scenarios. Whether you are looking to
only enable the default OpenSCAP content or if you want to import custom OpenSCAP content, you can use this playbook.

Why writing a custom playbook and not make use of the module
[`satellite.scap_content`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/module/scap_content/)? First of all,
I haven't been able to make the module work with `validate_certs: true`. I have created a [GitHub issue](https://github.com/theforeman/foreman-ansible-modules/issues/1611)
that hasn't been resolved yet (due to the lack of interaction on my end). Secondly, I simply haven't come around to build a role of it. I probably will eventually
do - but then again it makes no sense, as there is already a module that's supposed to work, which would make the role practically obsolete.

Nevertheless, for the time being you can use the playbook :sunglasses:

<!-- markdownlint-disable MD024 -->
### Documentation
<!-- markdownlint-enable MD024 -->

- [Module `satellite.scap_content`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/module/scap_content/)
- [OpenSCAP](https://www.open-scap.org/)

<!-- markdownlint-disable MD024 -->
### Roles, variables files and playbooks
<!-- markdownlint-enable MD024 -->

:information_source: The tables below can be mapped through the numbering column (`#`).[^column_explanation]

| #   | Role/s used                                                                |
| :-- | :------------------------------------------------------------------------- |
| 1   | N/A                                                                        |

| #   | Variable definition file/s                                                 |
| :-- | :------------------------------------------------------------------------- |
| 1   | `host_vars/<hostname>/17_openscap.yml`                                     |

| #   | Playbook/s used                                                            |
| :-- | :------------------------------------------------------------------------- |
| 1   | `20_satellite_openscap.yml`                                                |

<!-- markdownlint-disable MD024 -->
### Procedure
<!-- markdownlint-enable MD024 -->

1. Define the variables for OpenSCAP
    {% gist a222ecc8c0afaaf5d8dc9fcdbc3c7fe8 %}
1. Run the playbook to create the OpenSCAP content, enable the default OpenSCAP content and enable the Foreman OpenSCAP role:
    <!-- markdownlint-disable MD014 -->
    ```shell
    $ ansible-playbook -i inventory 20_satellite_openscap.yml --vault-pass-file .vault.pass
    ```
    <!-- markdownlint-enable MD014 -->

## Closing thoughts

This concludes this blog post. There are way more things you can do with
[Red Hat's Certified Ansible Collection for Red Hat Satellite](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite) and I highly encourage you
to explore the possibilities. I haven't touch on things as Compute Resources or Host Collections and many more things you could theoretically make use of with your specific
use case.

There are still a few things left to do! I either update this blog post or might create new blog posts about the topics. The topics that I'd like to touch in the future are:

- Adding Capsules
- Create Hosts in Satellite
- Create and assign OpenSCAP policies, as well as OpenSCAP tailoring files
- Figure out a way to make it easier to find out the Product and Repository labels
- *Maybe* convert the OpenSCAP playbook into a role

*BUT* don't hold your breath on those topics. I have other blogs that I'd like to write, but I'll surely revisit this blog in **the future**.

I hope this blog post was helpful to some of you :sunglasses:

.. until next time,

Steffen

## Change log

### 2024-03-11

- `markdownlint` fixes
- Spelling fixes

### 2024-03-09

- `markdownlint` fixes
- Typo fixes
- Adding a note that RHEL 8.8 and RHEL 8.9 with Ansible Core 2.15.4 has been tested
- Adding a note that this has been tested on Satellite 6.13 and Satellite 6.14

## Footnotes

[^key_usage]:["ERR_SSL_KEY_USAGE_INCOMPATIBLE" while accessing Red Hat Satellite WebUI after configure custom SSL certificates](https://access.redhat.com/solutions/6977733)
[^column_explanation]: All tables in the **Roles, variables files and playbooks** section are set up in a way that helps to easily identify which role uses which variable
                       files and which playbooks. For example, the role that is used in `#1` in the first table (`Role/s used`), makes use of the variables that
                       are defined in table `Variable definition file/s` in column `#1` and is executed with the playbook that is defined in the table `Playbook/s used`
                       in column `#1`.
