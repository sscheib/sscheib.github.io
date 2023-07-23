---
title: TODO
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
TODO:
**general**
- Specify which variables need to be changed
- Add to each section that all variables are found on online AH for role XY
- Comment variables in each code block
- Link object in Satellite to documentation
- ^ add link to both role documentation and Satellite documentation

**satellite install/configuration**
- Remember: Remove cert params from satellite installer if sat_deploy_certificates
- Firewall rules: Add hint to documentation (because personal ports)
- To the latest version: To the latest RHEL version available

**configure content credentials**
- Note: There is no elastic 9
- Add specific example for EPEL 8 and 9; :warning: if used will download a lot of shit

**enable RHEL repos**
- information_source: immediate on Kickstart
- Add comments to variables

**sync plans**
- Move before enable RHEL repos as sync_plan is specified

**template sync**
- Link to docu of template sync


This blog post does not follow the style of my previous blog posts. It is more or less a step by step manual on how to use [my repository on GitHub](https://github.com/sscheib/ansible_satellite) to automate Red Hat Satellite using Ansible.

Further, a good understanding of Satellite and, more importantly, a good understanding of the concepts around Ansible is **required**. Especially [variable precedence](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#understanding-variable-precedence) should be well known. Don't get me wrong, it is still a step by step guide, but nevertheless, if you want to adjust this step by step guide to *your* infrastructure (which is likely that you'd want to do that :grin:), you **need** to understand the concepts around Ansible.

All variables we define are either [host or group variables](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html#organizing-host-and-group-variables). This has the benefit that your inventory is not cluttered with variables, and additionally, this allows you to create different variables for each of your Satellite instances (if you have multiple). I use group variables only for variables that are required by all my Satellite instances.

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
ansible-playbook 00_create_kickstart.yml --vault-pass-file .vault.pass
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

## Registering the system to the Red Hat Customer Portal and installing Satellite

### Documentation
* [Role `redhat.rhel_system_roles.rhc`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/rhel_system_roles/docs/README_rhc/)
* [Role `redhat.satellite_operations.installer`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite_operations/content/role/installer/)
* [Registering a RHEL 8 system and managing subscriptions](https://access.redhat.com/documentation/de-de/red_hat_enterprise_linux/8/html/configuring_basic_system_settings/assembly_registering-the-system-and-managing-subscriptions_configuring-basic-system-settings)

:information_source: Only a fraction of the available variables are used for each role. Please make sure to check out above documentation of the respective role for a complete list of available variables.

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

1. Define your secrets in `host_vars/<hostname>/00a_secrets.yml`, please see below for an example.

    :information_source: The variables are not required right away, but it makes sense to define all of them at once.

    {% gist 4fbc66d25da509522e16791e497a27a2 %}

2. Define the variables for the [RHEL system role `redhat.rhel_system_roles.rhc`](https://access.redhat.com/documentation/de-de/red_hat_enterprise_linux/8/html/automating_system_administration_by_using_rhel_system_roles/using-the-rhc-system-role-to-register-the-system_automating-system-administration-by-using-rhel-system-roles) {% raw %}<br />{% endraw %}
    **Variables file**: `host_vars/<hostname>/00b_register_satellite.yml`
    {% gist 14d99b2931ce8e1bed1f6e9ac717326e %}
  
3. Run the playbook that registers your Satellite system to the Red Hat Content Delivery Network (RHCDN): 
    ```
    ansible-playbook -i inventory 01_register_satellite.yml --vault-pass-file .vault.pass
    ```
4. Define the variables for the [Satellite Installer role `redhat.satellite_operations.installer`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite_operations/content/role/installer/)
    
    **Variables file (certificates):** `host_vars/<hostname>/01a_satellite_installer_certificates.yml`

    **Note:** When generating certificates for your Satellite, remember to set the correct `keyUsage` and `extendedKeyUsage`[^key_usage]

    {% gist 7bbba923af596280573aa987180837fd %}

    **Variables file (firewall rules):** `host_vars/<hostname>/01b_satellite_firewall_rules.yml`

    {% gist 0b029863ee4e9953ab88978608bae777 %}

    **Variables file (Satellite installer):** `host_vars/<hostname>/01c_satellite_installer_configuration.yml`

    {% gist c99c3008a954a4ddb81451bad99b599f %}

5. Run the playbook to upgrade your system to the latest version (required) and install Satellite:
    
    :warning: This will reboot your Satellite to ensure the latest kernel is applied.
    ```
    ansible-playbook -i inventory 02_satellite_installer.yml --vault-pass-file .vault.pass
    ```
6. Verify that you can login to your Satellite instance once the playbook finished
7. Shutdown the VM and take a snapshot so you can roll back to this point when you encounter any issues later on

## Define general variables

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

1. Define variables that are reused multiple times

    **Variables file:** `host_vars/<hostname>/02_general.yml`
    {% gist 1bb3a400efe20cb7a07a7b64730ad1d2 %}

## *Optional*: Configure Satellite's Cloud Connector

:information_source: This step is entirely optional, as it merely configures [Satellite's Cloud Connector](https://access.redhat.com/documentation/en-us/red_hat_insights/2023/html-single/red_hat_insights_remediations_guide/index#assembly-configuring-satellite-cloud-connector_host-communication-with-insights).

:warning: Please make sure that you really **want** to have the Cloud Connector enabled. The Cloud Connector allows Insight to **run remediation on you Satellite**.

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

1. Define the variables

    **Variables file**: `host_vars/<hostname>/03_cloud_connector.yml`
    {% gist 64a79193346520cb437930b4497c7f7d %}

2. Run the playbook to configure Satellite's Cloud Connector:
    ```
    ansible-playbook -i inventory 03_satellite_cloud_connector.yml --vault-pass-file .vault.pass
    ```

## Importing a Manifest

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

3. Adjust the variables for this step in `host_vars/<hostname>/04_manifest.yml`, please see below for an example:
    {% gist 5e8b980306c43281971f6f8976c9a351 %}
4. Run the playbook to download the manifest and import it to your Satellite:
    ```
    ansible-playbook -i inventory 04_satellite_manifest.yml --vault-pass-file .vault.pass 
    ``` 

## Creating Content Credentials for custom Products

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

1. Define [Content Credentials](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html-single/managing_content/index#Importing_Custom_SSL_Certificates_content-management) for any custom Product in `host_vars/<hostname>/05_content_credentials.yml`. All available variables can be found in the role [`redhat.satellite.content_credentials`](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/satellite/content/role/content_credentials/):
    {% gist b573c065923442511509c519938b6f89 %} 

2. Run the playbook to create the Content Credentials:
    ```
    ansible-playbook -i inventory 05_satellite_content_credentials.yml --vault-pass-file .vault.pass
    ```

## Enabling Red Hat repositories, creating custom Products and Repositories and synchronize the Repositories

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
1. Finding out Red Hat Repository and Product names

    This step requires an already installed Satellite (to my knowledge), because you need to find out two things:
      * The Product name a repository is tied to
      * The Repository name itself

    The Repository and Product names can be found out via:
    1. `Satellite WebUI` -> `Content` -> `Red Hat Repositories`
    2. Click on `Filter by Product` and select a Product, e.g. `Red Hat Enterprise Linux for x86_64` (this is the **Product Name** to use)
    3. Select the appropriate "Type" (right hand side to the `Filter by Product` dropdown), e.g. `RPM` for RPM repositories, `Kickstart` for Kickstart repositories
    4. Either scroll through the `Available Repositories` or narrow the list further down using the search bar at the top
    5. The search lists contains the **Repository Name** to use
    6. Each Repository has a arrow an the left hand side. When clicking on it, it reveals two things:
      * The architecture (e.g. `x86_64`)
      * The release (which is optional and thus not available for every repository), (e.g. `8`, `8.7`, etc.)
2. Define the Red Hat Repositories and Products in `host_vars/<hostname>/06a_products.yml`
    {% gist 1ab1bbd6cc74c1f1fb47ea16c5e054ef %}
3. Define the custom Products and Repositories in `host_vars/<hostname>/06b_custom_products.yml`
    *Note:* Custom Repositories have to be tied to Products. So the logical step is to create custom Products and tie the custom Repositories to them
    {% gist f483a67504e10ef76189ea5a921dcecc %} 
4. The two definition will be merged in `host_vars/<hostname>/06c_combined_products.yml`
    {% gist ea886810cd22867a096eb2d3f96a48c5 %}
5. Run the playbook to enable and/or create the custom Products and Repositories
    ```
    ansible-playbook -i inventory 06_satellite_repositories.yml --vault-pass-file .vault.pass
    ```
6. Run the playbook to synchronize the Repositories (both custom and Red Hat)
    :information_source: This might take a while.
    ```
    ansible-playbook -i inventory 08_satellite_sync_repositories.yml --vault-pass-file .vault.pass
    ```

## Create Sync Plans

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
1. Define the Sync Plans in `host_vars/<hostname>/07_sync_plans.yml`
    {% gist 00a3ab9f03266ba08e80cc756d43ecc0 %}
2. Run the playbook to create the Sync Plans:
    ```
    ansible-playbook -i inventory 07_satellite_sync_plans.yml --vault-pass-file .vault.pass
    ```

## Create Lifecycle Environments

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
1. Define the Lifecycle Environment in `host_vars/<hostname>/08_lifecycle_environments.yml`
    {% gist 3dfd83aa5d7231d602b715316ce1f419 %}
2. Run the playbook to create the Lifecycle Environments:
    ```
    ansible-playbook -i inventory 09_satellite_lifecycle_environments.yml --vault-pass-file .vault.pass
    ```

## Create Domains

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
1. Define the Domains in `host_vars/<hostname>/09_domains.yml`
    {% gist 6f33448c5ef7750f10c79d88cd301ed7 %}
2. Run the playbook to create the Domains:
    ```
    ansible-playbook -i inventory 10_satellite_domains.yml --vault-pass-file .vault.pass
    ```

## Create Subnets

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
1. Define the Subnets in `host_vars/<hostname>/10_subnets.yml`
    {% gist 85c7a7082456159c05b6e8850f5a54a9 %}
2. Run the playbook to create the Subnets:

    ```
    ansible-playbook -i inventory 11_satellite_subnets.yml --vault-pass-file .vault.pass
    ```

## Define the (Composite-) Content Views and publish and promote the (Composite-) Content Views

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
*Note*: The order is important, as the Composite Content Views depend on the Content Views being created before the creation of the Composite Content Views.
1. Define the Content Views with custom Products in `host_vars/<hostname>/11a_content_views_custom_products.yml`
    {% gist 12fd843551041ed98632060d72b9dd60 %}
2. Define the Content Views with Red Hat Products in `host_vars/<hostname>/11b_content_views.yml`
    {% gist 50c5c2fee3238b47e2cee256ad488078 %}
3. Define the Composite Content Views in `host_vars/<hostname>/11c_composite_content_views.yml`
    {% gist 577f57b8b67b5accc6436ce21b69b62c %}
4. The definitions of 1. to 3. will be merged in `host_vars/<hostname>/11d_combined_content_views.yml`
    {% gist 8df9e83c3c21836afc83af093e34a3db %}
5. Run the playbook to create the (Composite-) Content Views:

    ```
    ansible-playbook -i inventory 12_satellite_content_views.yml --vault-pass-file .vault.pass
    ```

6. Run the playbook to publish and promote the (Composite-) Content Views

    ```
    ansible-playbook -i inventory 13_satellite_content_view_publish.yml --vault-pass-file .vault.pass 
    ```

## Apply Satellite settings and enable Template Synchronization

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
First, we need to apply some settings in Satellite, as the Operating System definitions depend on Provision Templates that should be imported prior to defining the Operating Systems. The Host Groups rely on the definitions of the Operating System, thus we need to first start with the Templates.

1. Define the Satellite settings, I have split mine into several files to not have one very large confusing file. I named the files like the tabs in the Satellite Web UI -> Administer -> Settings.
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
    ansible-playbook -i inventory 14_satellite_settings.yml --vault-pass-file .vault.pass
    ```
3. Create a [(Fine Grained) Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#fine-grained-personal-access-tokens) in GitHub with the following permissions (at least):
    * Administration: Read and Write (to deploy the SSH key from the Foreman user that performs the Template Synchronization)
    * Content: Read
    * Metadata: Read

   Of course, you can deploy the SSH key manually and modify the playbook that configures the Template Sync (`15_satellite_template_sync.yml`) to not deploy the SSH key to GitHub if you are uncomfortable with creating a GitHub Access Token that 
   has the *Administration* option set
4. Define the GitHub settings for the Template Sync in `group_vars/all/github.yml`
    {% gist 21475840ab5b6a90b263a87d9ee6bf70 %}
5. Run the playbook to configure the Template Sync and synchronize the Templates:
    ```
    ansible-playbook -i inventory 15_satellite_template_sync.yml --vault-pass-file .vault.pass
    ```

## Create Operating Systems

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
1. Define the Operating Systems in `host_vars/<hostname>/13_operating_systems.yml`:
    {% gist 031b18a1b51d901f088e88d8f779dc26 %}
2. Run the playbook to create and configure the Operating Systems:
    ```
    ansible-playbook -i inventory 16_satellite_operating_systems.yml --vault-pass-file .vault.pass
    ```

## Create Activation Keys
1. Define the Activation Keys in `host_vars/<hostname>/14_activation_keys.yml`
    {% gist 585614ff9d774fb064ce6b586f48efc8 %}
2. Run the playbook to create the Activation Keys:
    ```
    ansible-playbook -i inventory 17_satellite_activation_keys.yml --vault-pass-file .vault.pass
    ```

## Create Host Groups

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
Since I have applied [my Satellite 6 concept for Host Groups](https://blog.scheib.me/2023/05/30/redhat-satellite-concept.html#host-groups-hg), I have split the Host Groups into multiple files.
1. Define the Host Groups
    1. Base and Operating System level Host Groups: Variables files: `15a_host_groups_base.yml`
        {% gist c0ca37c45ec9ca780b2f7eff17fef984 %}
    2. Service level Host Groups: Variables files: `15b_host_groups_services.yml`
        {% gist d06b236f5f5ff5dc15abbe54d6174ec1 %}
    3. Lifecycle Environment level Host Groups: Variables files: `15c_host_groups_lifecycle_environments.yml`
        {% gist 46776747718e35681b94681f68bfdfe9 %}
    4. Merge all Host Groups into the required variable `satellite_hostgroups` for the role `redhat.satellite.hostgroups`
        {% gist 9dd2cd08399d49dad14a3637de88f935 %}
2. Run the playbook to create the Host Groups:
    ```
    ansible-playbook -i inventory 18_satellite_host_groups.yml --vault-pass-file .vault.pass
    ```

## Create Global Parameters

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
Again, I split the Global Parameters in smaller files to not have a big confusing file.
1. Define the Global Parameters
    1. Global Parameters for Ansible post provisioning
        {% gist ccacbc6d834959ab0a18898af41c3546 %}
    2. Global Parameters for Remote Execution
        {% gist 9a54f56b9a66f99473d49fe61d16e322 %}
    3. General Global Parameters
        {% gist 581f59a607c385a52e7750f93261a379 %}
    4. Merge all Global Parameters
        {% gist 594c9981cfde05e3db15e24bd3cb28fc %}
2. Run the playbook to create the Global Parameters:
    ```
    ansible-playbook -i inventory 19_satellite_global_parameters.yml --vault-pass-file .vault.pass
    ```
# Footnootes
[^key_usage]:["ERR_SSL_KEY_USAGE_INCOMPATIBLE" while accessing Red Hat Satellite WebUI after configure custom SSL certificates](https://access.redhat.com/solutions/6977733)
[^column_explanation]: All tables in the **Roles, variables files and playbooks** section are set up in a way that helps to easily identify which role uses which variable files and which playbooks. For example, the role that is used in `#1` in the first table (`Role/s used`), makes use of the variables that are defined in table `Variable definition file/s` in column `#1` and is executed with the playbook that is defined in the table `Playbook/s used` in column `#1`.
