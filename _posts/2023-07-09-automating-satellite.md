---
title: Automating Red Hat Satellite 6 - End to End
author: Steffen Scheib
---
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

## Preface
This blog post does not follow the style of my previous blog posts. It is more or less a step by step manual on how to use [my repository on GitHub](https://github.com/sscheib/ansible_satellite) to automate Red Hat Satellite using Ansible.

Further, a good understanding of Satellite and, more importantly, a good understanding of the concepts around Ansible is **required**. Especially [variable precedence](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#understanding-variable-precedence) should be well known. Don't get me wrong, it is still a step by step guide, but nevertheless, if you want to adjust this step by step guide to *your* infrastructure (which is likely that you'd want to do that :grin:), you **need** to understand the concepts around Ansible.

All variables we define are either [host or group variables](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html#organizing-host-and-group-variables). This has the benefit that your inventory is not cluttered with variables, and additionally, this allows you to create different variables for each of your Satellite instances (if you have multiple). I use group variables only for variables that are required by all my Satellite instances.

Please note, only a fraction of the available variables are used for each role. Please make sure to check out each section's documentation of the respective role for a complete list of available variables. Further, relevant documentation is also linked in each section that complements the respective steps that are done in Ansible if you'd like to know more about the general (manual) procedure.

In below sections I make heavy use of [GitHub gists](https://docs.github.com/en/get-started/writing-on-github/editing-and-sharing-content-with-gists/creating-gists#about-gists) to show **my personal configuration** of the respective roles. You **need to modify every gist so that it works for your environment**. For this purpose, I documented (almost) every variable in the gists to ensure the usage is clear. Should something be not clear, please refer to the role documentation (that is linked in every section), which provides additional context as well as possibly other variables to use that make sense to use in your environment.

:information_source: Just as a general information: Variables that are prefixed with `satellite_` are 'official' role variables of the roles we are going to use. Variables prefixed with `sat_` are custom variables (which usually get merged into a `satelite_` variable eventually).

:information_source: I am going to use `ansible-playbook` in this blog post and will thus install all required collections and roles as the current user. Of course, you can also make use of `ansible-navigator` with an Execution Environment that contains the collections and roles which are defined in `collections/requirements.yml`. This blog post won't cover the procedure of how to do that, however.


## Prerequisites
1. You need a host that will run Ansible and carry out the automation.

    :information_source: I have tested all of this on a RHEL 8.7 system with Ansible Core 2.14.6. RHEL 9 *should* work (untested) and Ansible Core versions starting from 2.12 *should* work.

    :warning: RHEL 7 is not going to work for this purpose, as only Ansible Engine 2.9 can (officially) be installed on it (which is too old for most - if not all - collections we are going to use).

2. Login to the host you'll be using to carry out the automation via SSH. Install `git`, check out my repository and change to the directory that has been created with the `git clone` command:
    ```shell
    $ sudo dnf install -y git
    $ git clone https://github.com/sscheib/ansible_satellite.git
    $ cd ansible_satellite
    ```
3. Configure Ansible that it'll be able to download [Red Hat's Ansible Certified Content Collections](https://access.redhat.com/support/articles/ansible-automation-platform-certified-content) that are needed for the playbooks to work from either your local Automation Hub or from [Red Hat's Automation Hub](https://console.redhat.com/ansible/automation-hub). If you don't know how to do either of those things, please use [Red Hat's documentation](https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform/2.4/html/getting_started_with_automation_hub/con-create-api-token) that explains how to connect to [Red Hat's Automation Hub](https://console.redhat.com/ansible/automation-hub).
Place your Ansible configuration file either in the directory you checked out (`ansible_satellite`) as `ansible.cfg` or in your home directory with a prefixed dot: `~/.ansible.cfg`.

    Your Ansible configuration file will look similar to below example:
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
4. Install the required collections:
    ```shell
    $ ansible-galaxy collection install -r collections/requirements.yml
    ```
5. Install the required roles:
    ```shell
    $ ansible-galaxy role install -r collections/requirements.yml
    ```
6. Create a Manifest for Satellite - we are going to import it at a [later point](#importing-a-manifest). If you don't know how to do it, please follow [Red Hat's blog](https://www.redhat.com/en/blog/how-create-and-use-red-hat-satellite-manifest) that explains very well how to create a Manifest.
7. Put the `Manifest UUID` in `host_vars/<hostname>/00a_secrets.yml`. As said, we are going to need it [later](#importing-a-manifest).
8. Optional (but **highly encouraged!**): You can create a [Vault password file](https://docs.ansible.com/ansible/2.8/user_guide/vault.html#providing-vault-passwords) file to pass it to `ansible-playbook` via `--vault-pass-file`. I named mine `.vault.pass` and added it to my [`.gitignore` file](https://git-scm.com/docs/gitignore) to ensure it is not accidentally pushed to the repository. As you are going to need the Vault for *every* playbook of my repository, it is handy to have a Vault password file - unless you chose to not encrypt your secrets (**highly discouraged!**).

    :information_source: If you are not familiar with Ansible Vault, please [read up on it](https://docs.ansible.com/ansible/latest/vault_guide/vault_encrypting_content.html#encrypting-content-with-ansible-vault) and don't store your secrets in plain text in your variables files. It is really super easy to get started with Ansible Vault, I promise :slightly_smiling_face:!

## Install Satellite server
The very first step is to install the RHEL 8 host that will be our Satellite eventually.

### Documentation
* [Role `rhel_iso_kickstart`](https://github.com/sscheib/ansible-role-rhel_iso_kickstart)
* [Kickstart commands and options reference for RHEL 8](https://access.redhat.com/documentation/de-de/red_hat_enterprise_linux/8/html/system_design_guide/kickstart-commands-and-options-reference_system-design-guide)

### Roles, variables files and playbooks

:information_source: The tables below can be mapped through the numbering column (`#`).[^column_explanation]

| #   | Role/s used                                                                |
| :-- | :------------------------------------------------------------------------- |
| 1   | `rhel_iso_kickstart`                                                       |

| #   | Variable definition file/s                                                 |
| :-- | :------------------------------------------------------------------------- |
| 1   | `host_vars/<hostname>/00_kickstart.yml`

| #   | Playbook/s used                                                            |
| :-- | :------------------------------------------------------------------------- |
| 1   | `00_create_kickstart.yml`                                                  |

### Procedure
1. Adjust the included Kickstart (`files/satellite.ks`) to your liking.
2. Adjust the variables for the role [`rhel_iso_kickstart`](https://github.com/sscheib/ansible-role-rhel_iso_kickstart)

    :information_source: In my case, I am going to run the role [`rhel_iso_kickstart`](https://github.com/sscheib/ansible-role-rhel_iso_kickstart) on `localhost` (as I want to have the ISO downloaded and built on my machine), so the `host_vars` have to be placed in the directory `host_vars/localhost/`. I have chosen to place them in the file `00_create_kickstart.yml` inside that directory.

    :information_source: Make sure to read the [README.md of my role `rhel_iso_kickstart`](https://github.com/sscheib/ansible-role-rhel_iso_kickstart) to fully understand all variables.
3. Run the playbook that will download 
```
$ ansible-playbook 00_create_kickstart.yml --vault-pass-file .vault.pass
```
3. Copy the resulting ISO file to your hypervisor and mount it to your VM that should become the Satellite. 
4. Start the VM and wait for the installation to finish.
5. Once the installation finished, shutdown the VM and unmount the ISO.

    :information_source: If you used the provided Kickstart, the VM will shut down after the Kickstart finished.

## Prepare Satellite for Ansible access
Now that we have successfully installed our RHEL 8 system, we need to ensure that Ansible can access the system and **elevate its privileges**.

### Procedure
1. Login via SSH to the new VM using `root` as username and the root password you defined in `00_create_kickstart.yml`

    :information_source: This step assumes that you have not already created a local user (other than `root`, of course) on the Satellite server. If you already created a local user as part of the Kickstart (or other means), please ensure that the user has privileged access; You can skip step **2** and **3**.
 
2. Create a new user for Ansible, e.g. `ansible-provisioning`.
3. Either set a password for `ansible-provisioning` or deploy your SSH key to the created user.
4. Ensure you add the user to the group `%wheel` so that you can elevate the privileges of the user.
6. Add the VM host to the inventory file. Below you'll find an example:
    ```shell
    $ cat inventory 
    satellite.office.int.scheib.me ansible_user=ansible-provisioning ansible_port=22
    ```
7. Ensure that your access to the VM is working properly:
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
Now it's time to register our RHEL system to the Red Hat Customer Portal. Please ensure that the host ends up with a Satellite Infrastructure subscription (or equivalent) as, otherwise, the Red Hat Satellite repositories cannot be found.

### Documentation
* [Role `redhat.rhel_system_roles.rhc`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/rhel_system_roles/docs/README_rhc/)
* [Role `redhat.satellite_operations.installer`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite_operations/content/role/installer/)
* [Registering a RHEL 8 system and managing subscriptions](https://access.redhat.com/documentation/de-de/red_hat_enterprise_linux/8/html/configuring_basic_system_settings/assembly_registering-the-system-and-managing-subscriptions_configuring-basic-system-settings)
* [Red Hat Satellite 6.12 Installation documentation](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html/installing_satellite_server_in_a_connected_network_environment/installing_server_connected_satellite#doc-wrapper)

### Roles, variables files and playbooks

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

### Procedure
1. Define your secrets in `host_vars/<hostname>/00a_secrets.yml`, please see below for an example.

    :information_source: Not all variables are needed right away, but it makes sense to define all of them at once. Please read the comments in the GitHub gist below.

    {% gist 4fbc66d25da509522e16791e497a27a2 %}

2. Define the variables for the [RHEL system role `redhat.rhel_system_roles.rhc`](https://access.redhat.com/documentation/de-de/red_hat_enterprise_linux/8/html/automating_system_administration_by_using_rhel_system_roles/using-the-rhc-system-role-to-register-the-system_automating-system-administration-by-using-rhel-system-roles)

    Below you'll find an example of my variables file (`host_vars/<hostname>/00b_register_satellite.yml`):
    {% gist 14d99b2931ce8e1bed1f6e9ac717326e %}
  
3. Run the playbook that registers your Satellite system to the Red Hat Content Delivery Network (RHCDN): 
    ```
    $ ansible-playbook -i inventory 01_register_satellite.yml --vault-pass-file .vault.pass
    ```
4. Next, we'll define the variables for the [Satellite Installer role `redhat.satellite_operations.installer`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite_operations/content/role/installer/).
   I have split the variables in three files:
    1. `host_vars/<hostname>/01a_satellite_installer_certificates.yml`: Contains variables with regards to certificates
        {% gist 7bbba923af596280573aa987180837fd %}

        :warning: When generating certificates for your Satellite, remember to set the correct `keyUsage` and `extendedKeyUsage`[^key_usage].

    2. `host_vars/<hostname>/01b_satellite_firewall_rules.yml`: Contains the ports to open on the Satellite
        {% gist 0b029863ee4e9953ab88978608bae777 %}

        :warning: Above firewall rules are specific to *my* use case. You need to adapt these based on the Satellite documentation and your used features (such as TFTP, DHCP, DNS, etc.).
    3. `host_vars/<hostname>/01c_satellite_installer_configuration.yml`: Contains all other variables for the role `redhat.satellite_operations.installer`
        {% gist c99c3008a954a4ddb81451bad99b599f %}

        :warning: If you are not planning to use certificates, please remove the certificate installer options from above example. Don't worry, the Satellite installer will create self-signed certificates. The certificate options are
                  are meant for implementing custom SSL certificates that have been signed by your internal certificate authority. 

5. Run the playbook to upgrade your system to the latest version (required) and install Satellite:
    
    :warning: This will reboot your Satellite to ensure the latest kernel is applied (as per the Satellite documentation).
    ```
    $ ansible-playbook -i inventory 02_satellite_installer.yml --vault-pass-file .vault.pass
    ```
6. Verify that you can login to your Satellite instance once the playbook finished
7. Shutdown the VM and take a snapshot so you can roll back to this point when you encounter any issues later on

## Define general variables
### Documentation
* [Satellite Collection Common Role Variables](https://github.com/theforeman/foreman-ansible-modules/blob/develop/README.md#common-role-variables)

  :information_source: The upstream common role variables are *not* used downstream in the Satellite collection. All `foreman_` variables can be substituted with `satellite_`, e.g. `foreman_url` -> `satellite_url`

### Roles, variables files and playbooks

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

### Procedure
1. With this, we define variables that are reused multiple times. To avoid duplication, it is good to define them once. Please see below an example of my definition:
    {% gist 1bb3a400efe20cb7a07a7b64730ad1d2 %}

## *Optional*: Configure Satellite's Cloud Connector

:information_source: This step is entirely optional, as it merely configures Satellite's Cloud Connector.

:warning: Please make sure that you really **want** to have the Cloud Connector enabled. The Cloud Connector allows Red Hat Insights to **run remediation on you Satellite** from the [Red Hat Hybrid Cloud Console](https://console.redhat.com).

### Documentation
* [Role `redhat.satellite_operations.cloud_connector`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite_operations/content/role/cloud_connector/)
* [Red Hat Insights Remediations Guide](https://access.redhat.com/documentation/en-us/red_hat_insights/2023/html-single/red_hat_insights_remediations_guide/index)
* [Enabling Cloud Connector on hosts managed by Satellite](https://access.redhat.com/documentation/en-us/red_hat_insights/2023/html-single/red_hat_insights_remediations_guide/index#configuring-satellite-cloud-connector_host-communication-with-insights)

### Roles, variables files and playbooks

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
2. Run the playbook to configure Satellite's Cloud Connector:
    ```
    $ ansible-playbook -i inventory 03_satellite_cloud_connector.yml --vault-pass-file .vault.pass
    ```

## Importing a Manifest
Next up, we are going to import a Manifest into our Satellite. I have configured the role `redhat.satellite.manifest` to first download a Manifest from the Red Hat Customer Portal and then upload it to my Satellite. If you are doing the same Please ensure you set the correct Manifest UUID in `satellite_manifest_uuid`, as well as the `rhsm_username` and `rhsm_password`. (as described in [Registering the system to the Red Hat Customer Portal and installing Red Hat Satellite](#registering-the-system-to-the-red-hat-customer-portal-and-installing-red-hat-satellite)).

### Documentation
* [Role `redhat.satellite.manifest`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/manifest/)
* [How to create and use a Red Hat Satellite manifest](https://www.redhat.com/en/blog/how-create-and-use-red-hat-satellite-manifest)
* [Importing a Red Hat Subscription Manifest into Satellite Server](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html/managing_content/managing_red_hat_subscriptions_content-management#Importing_a_Red_Hat_Subscription_Manifest_into_Server_content-management)

### Roles, variables files and playbooks

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

3. Adjust the variables for this step in `host_vars/<hostname>/04_manifest.yml`, please see below for an example:
    {% gist 5e8b980306c43281971f6f8976c9a351 %}
4. Run the playbook to download the manifest and import it to your Satellite:
    ```
    $ ansible-playbook -i inventory 04_satellite_manifest.yml --vault-pass-file .vault.pass 
    ``` 

## Creating Content Credentials for custom Products
Before creating custom Products and repositories, we need to create the custom Content Credentials. This is required, as we are going to use those Content Credentials to ensure that the content we retrieve is retrieved without any modification (through a malicious actor for instance). This is only required for 3rd-party repositories, as Red Hat repositories are validated by default.

## Documentation
* [Role `redhat.satellite.content_credentials`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/content_credentials/)
* [Importing Custom SSL Certificates](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html-single/managing_content/index#Importing_Custom_SSL_Certificates_content-management)

### Roles, variables files and playbooks

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

### Procedure

1. Define the required Content Credentials for any custom Product in `host_vars/<hostname>/05_content_credentials.yml`. Please see an example of my Content Credentials below:
    {% gist b573c065923442511509c519938b6f89 %} 

2. Run the playbook to create the Content Credentials:
    ```
    $ ansible-playbook -i inventory 05_satellite_content_credentials.yml --vault-pass-file .vault.pass
    ```

## Enabling Red Hat Repositories, creating custom Products and Repositories and synchronize the Repositories
Now that we have created the Content Credentials for our custom Products, we can start creating the custom Products and their containing Repositories. Further, we are going to enable the official Red Hat Repositories that we require. Conveniently, this can all be done by one role, so we are doing it in one go.

Since we do a couple of things with one role, I decided to split up my variables into Custom Products (`host_vars/<hostname>/06a_products.yml`) and Red Hat Products (`host_vars/<hostname>/06a_products.yml`). Lastly, we combine the two definitions into one variable which is stored as `satellite_repositories`: That is the variable the role `redhat.satellite.repositories` requires.

:information_source: Unfortunately, this step requires an already installed Satellite. I am not aware of any other method of finding out the Product and Repository names. If you know another way, kindly let me know in the comments below.

## Documentation
* [Role `redhat.satellite.repositories`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/repositories/)
* [Introduction to Content Management](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html-single/managing_content/index#Introduction_to_Content_Management_content-management)
* [Download Policies Overview](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html-single/managing_content/index#Download_Policies_Overview_content-management)

### Roles, variables files and playbooks

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

### Procedure
1. Finding out Red Hat Repository and Product names

    This step requires an already installed Satellite (to my knowledge), because you need to find out two things:
      * The Product name a Repository is tied to
      * The Repository name itself

    The Repository and Product names can be found out via the following procedure:
    1. `Satellite WebUI` -> `Content` -> `Red Hat Repositories`
    2. Click on `Filter by Product` and select a Product, e.g. `Red Hat Enterprise Linux for x86_64` (this is the **Product Name** to use)
    3. Select the appropriate "Type" (right hand side to the `Filter by Product` drop-down), e.g. `RPM` for RPM repositories, `Kickstart` for Kickstart repositories
    4. Either scroll through the `Available Repositories` or narrow the list further down using the search bar at the top
    5. The search lists contains the **Repository Name** to use
    6. Each Repository has a arrow an the left hand side. When clicking on it, it reveals two things:
      * The architecture (e.g. `x86_64`)
      * The release (which is optional and thus not available for every repository), (e.g. `8`, `8.7`, etc.)
2. Define the Red Hat Repositories and Products in `host_vars/<hostname>/06a_products.yml`
    {% gist 1ab1bbd6cc74c1f1fb47ea16c5e054ef %}

    :warning: The Kickstart Repositories are set to the `download_policy` `immediate` as having them set to `on_demand` is known to cause issues during provisioning.

3. Define the custom Products and their containing Repositories in `host_vars/<hostname>/06b_custom_products.yml`
    {% gist f483a67504e10ef76189ea5a921dcecc %} 
4. The two definition will be merged in `host_vars/<hostname>/06c_combined_products.yml`
    {% gist ea886810cd22867a096eb2d3f96a48c5 %}
5. Run the playbook to enable and/or create the custom Products and Repositories
    ```
    $ ansible-playbook -i inventory 06_satellite_products_and_repositories.yml --vault-pass-file .vault.pass
    ```
6. Run the playbook to synchronize the Repositories (both custom and Red Hat)

    :information_source: This might take a while.

    :warning: If you'd like to follow along with the blog post, you *have* to run this step, as otherwise the Content Views *will* be *empty*!

    ```
    $ ansible-playbook -i inventory 08_satellite_sync_repositories.yml --vault-pass-file .vault.pass
    ```

## Create Sync Plans
To ensure our content is kept up to date, we are going to create Sync Plans in the next step.

## Documentation
* [Role `redhat.satellite.sync_plans`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/sync_plans/)
* [Creating a Sync Plan](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html-single/managing_content/index#Creating_a_Sync_Plan_content-management)
* [Assigning a Sync Plan to a Product](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html-single/managing_content/index#Assigning_a_Sync_Plan_to_a_Product_content-management)

### Roles, variables files and playbooks

:information_source: The tables below can be mapped through the numbering column (`#`).[^column_explanation]

| #   | Role/s used                                                                |
| :-- | :------------------------------------------------------------------------- |
| 1   | `redhat.satellite.sync_plans`                                              |

| #   | Variable definition file/s                                                 |
| :-- | :------------------------------------------------------------------------- |
| 1   | `host_vars/<hostname>/07_sync_plans.yml`                                   |

| #   | Playbook/s used                                                            |
| :-- | :------------------------------------------------------------------------- |
| 1   | `07_satellite_sync_plans.yml`                                              |

### Procedure
1. Define the Sync Plans in `host_vars/<hostname>/07_sync_plans.yml`
    {% gist 00a3ab9f03266ba08e80cc756d43ecc0 %}

    :information_source: I am reusing the definition of the Products we have done earlier to map all products to one Sync Plan.

2. Run the playbook to create the Sync Plans:
    ```
    $ ansible-playbook -i inventory 07_satellite_sync_plans.yml --vault-pass-file .vault.pass
    ```

## Create Lifecycle Environments
When it comes to Lifecycle Environments I have a *very* simple use case. I have two Lifecycle Environments and that's it. Your use case might be much more complex, but the example below should get you started quickly.

### Documentation
* [Role `redhat.satellite.lifecycle_environments`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/lifecycle_environments/)
* [Managing Application Life Cycles](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html/managing_content/creating_an_application_life_cycle_content-management)

### Roles, variables files and playbooks

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

### Procedure
1. Define the Lifecycle Environment in `host_vars/<hostname>/08_lifecycle_environments.yml`. Please see below for an example:
    {% gist 3dfd83aa5d7231d602b715316ce1f419 %}
2. Run the playbook to create the Lifecycle Environments:
    ```
    $ ansible-playbook -i inventory 09_satellite_lifecycle_environments.yml --vault-pass-file .vault.pass
    ```

## Create Domains
This one is super easy for my use case as well. I have exactly **one** Domain I am using internally for my lab environment: `office.int.scheib.me`. Since this is conveniently also the value of the initial Domain I have specified, I am just reusing it. Your use case might be much more complex; This example can easily be extended.

### Documentation
* [Role `redhat.satellite.domains`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/domains/)

### Roles, variables files and playbooks

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

### Procedure
1. Define the Domains in `host_vars/<hostname>/09_domains.yml`
    {% gist 6f33448c5ef7750f10c79d88cd301ed7 %}
2. Run the playbook to create the Domains:
    ```
    $ ansible-playbook -i inventory 10_satellite_domains.yml --vault-pass-file .vault.pass
    ```

## Create Subnets
When it comes to Subnets it's the same as for the Domains: I have a super easy use case. One Subnet. That's it :grin:

Again, your use case might be much, much more complex, but the example below should get you started.

### Documentation
* [Role `redhat.satellite.subnets`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/subnets/)

### Roles, variables files and playbooks

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

### Procedure
1. Define the Subnets in `host_vars/<hostname>/10_subnets.yml`
    {% gist 85c7a7082456159c05b6e8850f5a54a9 %}
2. Run the playbook to create the Subnets:
    ```
    $ ansible-playbook -i inventory 11_satellite_subnets.yml --vault-pass-file .vault.pass
    ```

## Define the (Composite) Content Views and publish and promote them
We are now going to automate one of the more complex things: Content Views and Composite Content Views. Since Composite Content Views will contain `components` (which are Content View Versions), they are dependent on the Content Views being present at the time we are going to create the Composite Content Views. So, the order really matters!

As the order matters, I am splitting again the Content Views in the `host_vars` in to several files:
1. `host_vars/<hostname>/11a_content_views_custom_products.yml`: Contains Content Views with Custom Products we have created earlier
2. `host_vars/<hostname>/11b_content_views.yml`: Contains Content Views which use official Red Hat Products
3. `host_vars/<hostname>/11c_composite_content_views.yml`: Contains Composite Content Views (which depend on the creation of the earlier defined Content Views)
4. `host_vars/<hostname>/11d_combined_content_views.yml`: Lastly, I merge the definitions of the aforementioned (Composite) Content Views into one variable, which is used by the role `redhat.satellite.content_views`: `satellite_content_views`

### Documentation
* [Role `redhat.satellite.content_views`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/content_views/)
* [Managing Content Views](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html/managing_content/managing_content_views_content-management)

### Roles, variables files and playbooks

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

### Procedure
1. Define the Content Views with custom Products in `host_vars/<hostname>/11a_content_views_custom_products.yml`
    {% gist 12fd843551041ed98632060d72b9dd60 %}
2. Define the Content Views with Red Hat Products in `host_vars/<hostname>/11b_content_views.yml`
    {% gist 50c5c2fee3238b47e2cee256ad488078 %}
3. Define the Composite Content Views in `host_vars/<hostname>/11c_composite_content_views.yml`
    {% gist 577f57b8b67b5accc6436ce21b69b62c %}
4. The definitions of 1. to 3. will be merged in `host_vars/<hostname>/11d_combined_content_views.yml`
    {% gist 8df9e83c3c21836afc83af093e34a3db %}
5. Run the playbook to create the (Composite) Content Views:
    ```
    $ ansible-playbook -i inventory 12_satellite_content_views.yml --vault-pass-file .vault.pass
    ```
6. Run the playbook to publish and promote the (Composite) Content Views
    ```
    $ ansible-playbook -i inventory 13_satellite_content_view_publish.yml --vault-pass-file .vault.pass 
    ```

## Apply Satellite settings and enable Template Synchronization
We need to apply some settings in Satellite, as the Operating System definitions depend on Provision Templates that have to be imported prior to defining the Operating Systems. The Host Groups and the Activation Keys (which we still need to create) rely on the definitions of the Operating System, thus we need to start with importing the Templates.

I make use of [Satellite's TemplateSync Plug-in](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html/managing_hosts/synchronizing_templates_repositories_managing-hosts#doc-wrapper) which allows us to import Templates from a Source Code Management (SCM) tool (such as GitHub or Git Lab). TemplateSync will also assign the Templates to the correct Operating System, Organization and Location, if the [required metadata](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html/managing_hosts/synchronizing_templates_repositories_managing-hosts#Importing_Templates_managing-hosts) is present in the Template's header.

Again, I have split my variables into several files to not have one very large confusing file. I named the files like the tabs in the `Satellite Web UI` -> `Administer` -> `Settings`.

This chapter essentially covers two steps:
1. Set specific settings in the Satellite, especially the Template Sync settings. But since we are on it, I'll define *all* settings right away to spare me some time.
2. Run the Template Sync to import the Templates of my [Kickstart Repository on GitHub](https://github.com/sscheib/satellite_templates). The playbook will perform [the steps that are required to synchronize Templates](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html/managing_hosts/synchronizing_templates_repositories_managing-hosts#Synchronizing_Templates_Using_the_API_managing-hosts) from git. Please review them prior to running the playbook (`15_satellite_template_sync.yml`).

    :information_source: Unfortunately, there is no role to configure and run the Template Sync. This will be done by a custom playbook I have written and importing the Templates using the Satellite API. It is a playbook, as I didn't see the need to create a role for such a basic task.

### Documentation
* [Role `redhat.satellite.settings`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/settings/)
* [Synchronizing Template Repositories](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html/managing_hosts/synchronizing_templates_repositories_managing-hosts#doc-wrapper)
* [GitHub: Managing your personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#fine-grained-personal-access-tokens)
* [Kickstarting Red Hat Enterprise Linux (RHEL) systems using a highly customized Kickstart with Red Hat Satellite 6](https://blog.scheib.me/2023/07/01/highly-customized-kickstart.html)

### Roles, variables files and playbooks

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

### Procedure

1. Define the Satellite settings in our variables files:
    1. **Tab Template Sync**: Variables file: `12a_settings_template_sync.yml`:
        {% gist bf5474d6cd2b41fcf237a515bc92a829 %}
    2. **Tab Provisioning**: Variables file: `12b_settings_provisioning.yml`:
        {% gist cb31468e436df3b8ce344d7c5376baf3 %}
    3. **Tab Remote Execution:** Variables file: `12c_settings_remote_execution.yml`:
        {% gist fe1d73bc8c895a7517370f403f4346d1 %}
    4. **Tab RH Cloud:** Variables file: `12d_settings_rh_cloud.yml`:
        {% gist 4b56a2e775b67c6d442ffc719b4b96d1 %}
    5. **Tab General**: Variables file: `12e_settings_general.yml`
        {% gist 0b57b779e5a944a77fee3ab0ebaa02a7 %}
    6. **Tab Notifications**: Variables file: `12f_settings_notifications.yml`
        {% gist 7842c40f0833abb64465061daeafa332 %}
    7. All settings are merged into one variable (`satellite_settings`) which is the variable that is used by the role `redhat.satellite.settings` in the file `12g_merge_settings.yml`:
        {% gist c9fd9deb61e6d39749ec9ef02073bf26 %}

2. Run the playbook to apply the settings:
    ```
    $ ansible-playbook -i inventory 14_satellite_settings.yml --vault-pass-file .vault.pass
    ```
3. Create a [(Fine Grained) Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#fine-grained-personal-access-tokens) in GitHub with the following permissions (at least):
    * `Administration: Read and Write` (to deploy the SSH key from the Foreman user that performs the Template Synchronization)
    * `Content: Read`
    * `Metadata: Read`

    :warning: Of course, you can deploy the SSH key manually and modify the playbook that configures the Template Sync (`15_satellite_template_sync.yml`) to not deploy the SSH key to GitHub if you are uncomfortable with creating a GitHub Access Token that has the *Administration* option set

4. Define the GitHub settings for the Template Sync in `group_vars/all/github.yml`
    {% gist 21475840ab5b6a90b263a87d9ee6bf70 %}
5. Run the playbook to configure the Template Sync and synchronize the Templates:
    ```
    $ ansible-playbook -i inventory 15_satellite_template_sync.yml --vault-pass-file .vault.pass
    ```

## Create Operating Systems
Since the Templates are now imported, we can start creating the Operating Systems and assign the imported Templates to them.

### Documentation
* [Role `redhat.satellite.operatingsystems`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/operatingsystems/)
* [Creating Operating Systems](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html-single/provisioning_hosts/index#creating-operating-systems_provisioning)

### Roles, variables files and playbooks

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

### Procedure
1. Define the Operating Systems in `host_vars/<hostname>/13_operating_systems.yml`:
    {% gist 031b18a1b51d901f088e88d8f779dc26 %}
2. Run the playbook to create and configure the Operating Systems:
    ```
    $ ansible-playbook -i inventory 16_satellite_operating_systems.yml --vault-pass-file .vault.pass
    ```

## Create Activation Keys
Creating Activation Keys is now possible, as we can refer to the Operating Systems that we created in the prior chapter.

### Documentation
* [Role `redhat.satellite.activation_keys`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/activation_keys/)
* [Managing Activation Keys](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html/managing_content/managing_activation_keys_content-management)

### Roles, variables files and playbooks

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

### Procedure
1. Define the Activation Keys in `host_vars/<hostname>/14_activation_keys.yml`
    {% gist 585614ff9d774fb064ce6b586f48efc8 %}
2. Run the playbook to create the Activation Keys:
    ```
    $ ansible-playbook -i inventory 17_satellite_activation_keys.yml --vault-pass-file .vault.pass
    ```

## Create Host Groups
Almost done! :sunglasses: 

As our second to last step, we are going to create the Host Groups. Since I have applied [my Satellite 6 concept for Host Groups](https://blog.scheib.me/2023/05/30/redhat-satellite-concept.html#host-groups-hg), I have split the Host Groups into multiple files.

### Documentation
* [Role `redhat.satellite.hostgroups`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/hostgroups/)
* [Creating a Host Group](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html/managing_hosts/administering_hosts_managing-hosts#Creating_a_Host_Group_managing-hosts)

### Roles, variables files and playbooks

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

### Procedure
1. Define the Host Groups
    1. **Base** and **Operating System level Host Groups**: `host_vars/<hostname>/15a_host_groups_base.yml`
        {% gist c0ca37c45ec9ca780b2f7eff17fef984 %}
    2. **Service level Host Groups**: `host_vars/<hostname>/15b_host_groups_services.yml`
        {% gist d06b236f5f5ff5dc15abbe54d6174ec1 %}
    3. **Lifecycle Environment level Host Groups**: `host_vars/<hostname>/15c_host_groups_lifecycle_environments.yml`
        {% gist 46776747718e35681b94681f68bfdfe9 %}
    4. Finally, merge all Host Groups into the required variable `satellite_hostgroups` for the role `redhat.satellite.hostgroups`; That is done in the file `host_vars/<hostname>/15d_host_groups_merged.yml`
        {% gist 9dd2cd08399d49dad14a3637de88f935 %}
2. Run the playbook to create the Host Groups:
    ```
    $ ansible-playbook -i inventory 18_satellite_host_groups.yml --vault-pass-file .vault.pass
    ```

## Create Global Parameters
Now on the the Global Parameters. I will define some Global Parameters that are specific to my use case, which [my Provisioning Templates](https://github.com/sscheib/satellite_templates) make use of as well as override some default Parameters. My parameters are denoted with a prefixed `p-`. All Parameters which do not have that prefix are Parameters that are present by default.

Once again, I split up my variables into separate files to not have a big clunky file that is unmaintainable. :grin:

:information_source: Unfortunately, there is no role available that creates Global Parameters, but there is a module available which we can use: `redhat.satellite.global_parameter`.

### Documentation
* [Module `redhat.satellite.global_parameter`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/module/global_parameter/)

### Roles, variables files and playbooks

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

### Procedure
1. Define the Global Parameters
    1. Global Parameters for Ansible post provisioning: `host_vars/<hostname>/16a_global_parameters_ansible_provisioning.yml`
        {% gist ccacbc6d834959ab0a18898af41c3546 %}
    2. Global Parameters for Remote Execution: `host_vars/<hostname>/16b_global_parameters_remote_execution.yml`
        {% gist 9a54f56b9a66f99473d49fe61d16e322 %}
    3. General Global Parameters: `host_vars/<hostname>/16c_global_parameters_general.yml`
        {% gist 581f59a607c385a52e7750f93261a379 %}
    4. Merge all Global Parameters: `host_vars/<hostname>/16d_merge_global_parameters.yml`
        {% gist 594c9981cfde05e3db15e24bd3cb28fc %}
2. Run the playbook to create the Global Parameters:
    ```
    $ ansible-playbook -i inventory 19_satellite_global_parameters.yml --vault-pass-file .vault.pass
    ```

## *Optional:* Enable and import OpenSCAP content
If you are looking to ensure compliance on your hosts using [OpenSCAP](https://www.open-scap.org/), this chapter will be for you. Here is the thing: Satellite comes already pre-loaded with some OpenSCAP content. I have written a playbook that will accommodate the two scenarios. Whether you are looking to only enable the default OpenSCAP content or if you want to import custom OpenSCAP content, you can use this playbook.

Why writing a custom playbook and not make use of the module [`satellite.scap_content`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/module/scap_content/)? First of all, I haven't been able to make the module work with `validate_certs: true`. I have created a [GitHub issue](https://github.com/theforeman/foreman-ansible-modules/issues/1611) that hasn't been resolved yet (due to the lack of interaction on my end). Secondly, I simply haven't come around to build a role of it. I probably will eventually do - but then again it makes no sense, as there is already a module that's supposed to work, which would make the role practically obsolete.

Nevertheless, for the time being you can use the playbook :sunglasses:

### Documentation
* [Module satellite.scap_content](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/module/scap_content/)
* [OpenSCAP](https://www.open-scap.org/)

### Roles, variables files and playbooks

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

### Procedure
1. Define the variables for OpenSCAP
    {% gist a222ecc8c0afaaf5d8dc9fcdbc3c7fe8 %}
2. Run the playbook to create the OpenSCAP content, enable the default OpenSCAP content and enable the Foreman OpenSCAP role:
    ```
    $ ansible-playbook -i inventory 20_satellite_openscap.yml --vault-pass-file .vault.pass
    ```

# Closing thoughts
This concludes this blog post. There are way more things you can do with [Red Hat's Certified Ansible Collection for Red Hat Satellite](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite) and I highly encourage you to explore the possibilities. I haven't touch on things as Compute Resources or Host Collections and many more things you could theoretically make use of with your specific use case.

I hope this blog post was helpful to some of you :sunglasses:

.. until next time

# Footnotes
[^key_usage]:["ERR_SSL_KEY_USAGE_INCOMPATIBLE" while accessing Red Hat Satellite WebUI after configure custom SSL certificates](https://access.redhat.com/solutions/6977733)
[^column_explanation]: All tables in the **Roles, variables files and playbooks** section are set up in a way that helps to easily identify which role uses which variable files and which playbooks. For example, the role that is used in `#1` in the first table (`Role/s used`), makes use of the variables that are defined in table `Variable definition file/s` in column `#1` and is executed with the playbook that is defined in the table `Playbook/s used` in column `#1`.
[^secrets]:[Registering the system to the Red Hat Customer Portal and installing Red Hat Satellite](#registering-the-system-to-the-red-hat-customer-portal-and-installing-red-hat-satellite)
