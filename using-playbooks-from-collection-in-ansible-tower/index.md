# Using playbooks from collection in Ansible Tower

ksangtani

Published on November 10, 2020 / Updated on November 10, 2020

### Overview

Skill Level: Beginner

This article covers using playbooks from a collection in Ansible Tower.

### Ingredients

* Ansible (2.10)
* Ansible Tower

### Step-by-step

#### 1. Installing an Ansible Collection on your local workstation

For this article, let us consider that we need to install satellite and there is a collection already available in the community. Example of an Ansible Collection to install and tune satellite is https://github.com/jjaswanson4/install\_satellite.git.

With ansible 2.10, we can install ansible collection using git url

```
# ansible-galaxy collection install https://github.com/jjaswanson4/install_satellite.git,master
Starting galaxy collection install process
Process install dependency map
Starting collection install process
Installing ' jjaswanson4.ansible_collection_aic_v3:3.0.0' to '/root/.ansible/collections/ansible_collections/jjaswanson4/install_satellite'
Created collection for jjaswanson4. install_satellite at /root/.ansible/collections/ansible_collections/ jjaswanson4/install_satellite
jjaswanson4.install_satellite (1.0.0) was installed successfully
```

#### 2. Invoking the playbook from collections in your own playbook

In your main.yml add the following:

```
# vi my_playbook/main.yml
  ---
  - name: Install Satellite
   import_playbook: "{{ playbook_dir }}/../requirements_collections/ansible_collections/jjaswanson4/install_satellite /playbooks/install_satellite.yml"
   remote_user: 'ansible'
   become: true
```

So basically, you need to import the playbook and provide the full path of the collections playbook.

#### 3. How to use this same playbook within Tower

You need to add the collections as a requirement.

Create collections/requirements.yml as follows. This mainly for when we add the playbook in a job template on AWX.

```
# cat collections/requirements.yml
---
collections:
  - name: https://github.com/jjaswanson4/install_satellite.git
    type: git
    version: master
```

When the playbook/project is updated in Tower, it will read the collections/requirements.yml and install the collection on the Ansible control node.

#### 4. References

1.  https://docs.ansible.com/ansible/devel/dev\_guide/developing\_collections.html#developing-collections
2.  https://docs.ansible.com/ansible/latest/user\_guide/collections\_using.html
3.  https://www.ansible.com/blog/installing-and-using-collections-on-ansible-tower
4.  https://docs.ansible.com/ansible/latest/collections/ansible/builtin/import\_playbook\_module.html
