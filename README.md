# Services - Ansible

David Williamson @ Varilink Computing Ltd

------

## Playbooks

| Playbook            | Function                                                                                                                                                                           |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| bootstrap.yml       | Bootstraps a newly created virtual machine to prepare it for the creation of core *services* that it will provie.                                                                  |
| create-services.yml | Creates core *services* using [roles](https://github.com/varilink/libraries-ansible#roles) defined in [varilink/libraries-ansible](https://github.com/varilink/libraries-ansible). |

### bootstrap.yml

This playbook is applied to newly created *Linodes* to prepare them for subsequently applying the roles allocated to them. The steps to apply this playbook are as follows:

1. Commission the Linode making sure to enable my SSH key for access and capture the allocated root password.

2. Make an entry in the SSH config file for the new host using its *physical* hostname rather than the subsequent host alias; for example `mars` **not** `prod3`.

3. Add the host to the Ansible inventory file in the `bootstrap` group again using its *physical* hostname.

4. Run the playbook, which only acts on hosts in the `bootstrap` group.

The outcome should be that it is possible to SSH to the new host using the *physical* hostname from a logged in admin user session on the desktop and then to `sudo` on the host. You can then complete the setup for further deployment as follows:

1. Remove the host from the `bootstrap` group in the Ansible inventory.

2. Add an appropriate host alias for the host in the `external` group in the Ansible inventory.

3. Add the new host using its host alias to the relevant group(s) in the Ansible inventory according to the core *services* that it will provide.

4. Add a `host_vars` file for the new host using its host alias to the inventory.

5. Be sure to run an `apt update` on the new host so that the package cache is populated there. It's probably not a bad idea to run a package upgrade while you're there.

### create-services.yml

For a new host:

1. Add the host to the `dns_host_patterns` variable in the Ansible inventory.

2. Update the Dnsmasq configuration to include the new host in the DNS service.

```bash
ansible-playbook --limit=hub --tags=dns create-services.yml
```

If this is a host to be included in the backup.

3. Add the host to `hosts_to_roles_map` in the Ansible inventory.

4. Update the Bacula Director configuration to include the new host in the backup.

```bash
ansible-playbook --limit=hub --tags=backup create-services.yml
```

## Commissioning a new host

We operate three types hosts:

1. Internet hosts for which the operating system is installed via our hosting provider's virtual server images.

2. Office hosts running on a Raspberry Pi for which the operating system is installed via a Raspbian image.

3. Office hosts running on other types of hardware for which the operating system is installed using the Debian installer.

These differences in how the operating system is installed above in turn lead to subtle differences in the procedure to complete the commissioning of new hosts of each type. The procedure has the following steps:

Detailed instructions for each of these steps now follow.

### SSH CONFIGURATION AND SETTING OF IP ADDRESSES?

### Initial hosts inventory entry and associated setting of vars

Make a temporary entry in the hosts inventory file for the new host placing it in the `bootstrap` inventory group. There are a couple of factors that influence the form that this entry takes.

1. Where a host is a replacement for another host fulfilling one of the unique roles (`gateway`, `home` and `hub`) then you must use the hostname rather than the role name, which we would normally use as an alias for the hostname. This is because we must manage the decommissioing of the old host alongside the commissioning of the new host - see below.

2. Hosts on the office network initially join that network using a temporary IP address that has been provisioned via DHCP, the bootstrap procedure replaces this with a static IP address and associated network configuration for the office network. By contrast, Internet hosts are created with their enduring, static IP address.

Office host not fulfilling a unique role:
```ini
[bootstrap]
dev4 ansible_host=192.168.1.238
```

Office host fulfilling a unique role:
```ini
[bootstrap]
europe ansible_host=192.168.1.239
```

Internet host:
```ini
[bootstrap]
prod6 ansible_host=neptune
```

### Running the bootstrap playbook

We can now run the `bootstrap.yml` playbook to set the host up for other playbooks to complete its build. At this point the host has not been configured with `authorized_keys` for convenient SSH connections nor has it been configured for the connection user to be able to use `sudo` for privilege escalation. Setting these aspects up is done by the `bootstrap.yml` playbook itself. So, when running the `bootstrap.yml` playbook we must use the `--ask-pass` and `--ask-become-pass` options of the `ansible-playbook` command accordingly.

For example:

```bash
ansible-playbook --limit europe --ask-pass --ask-become-pass ./playbook/bootstrap.yml
```

Note that since the last step in this playbook is to set the permanent static IP address for office hosts the playbook will hang when it has done this rather than exit normally. This is of course because the IP address changes from the one that the playbook used to make SSH connections to the host. It's okay to simply <kbd>Ctrl-C</kbd> to terminate the hanging playbook.

### Post bootstrap playbook hosts inventory update

Once the `bootstrap.yml` playbook has completed move the new host in the hosts inventory file out of the `bootstrap` group and into its final group/groups. Each host must go into one of the `internal` or `external` groups and optionally into one or more other groups as appropriate.

For office hosts not fulfilling a unique role, also change the value of `ansible_host` in the hosts inventory from the temporary, starting IP address to the hosts name.

For office hosts fulfilling a unique role, just remove the setting of any value for `ansible_host` from the hosts inventory for now.

### Configure the DNS and/or backup services for the new host

Unless there is some exceptional circumstance, the DNS and backup services must both now be configured for the new host. Before executing the playbook runs to do this it is necessary to set the required variables. First add the new host to both the `dns_host_patterns` and `hosts_to_roles_map` inventory variables.

When adding the host to `hosts_to_roles_map` you can if you wish assign an empty list of roles at the point:

For example:

```yaml
europe: []
```

However, it's more correct to immediately start backing the new host according to the roles assigned to it, which are imminently going to be both the `backup_client` and `dns_client` roles.

So, for example:

```yaml
europe:
  - backup_client
  - dns_client
```

Actually, it's a good idea to combine reconfiguring the DNS and backup services with enabling the new host to be a client for both of those services. To do this requires us to configure the `backup_client_director_password`, `backup_client_monitor_password` and `debian_release` variables in a `host_vars` file for the new host.

With all that configuration in place we can now execute the playbook runs to establish DNS and backup services for the new client. The DNS service must be setup first as other services are dependent on it.

For example:

```bash
ansible-playbook --limit hub,europe --tags dns ./playbooks/install-services.yml
```

And on completion, immediately do the same for the backup service:

```bash
ansible-playbook --limit hub,europe --tags backup ./playbooks/install-services.yml
```

## Transferring unique services from old host to new host


Where a new host replaces an old host in one of the unique host roles, e.g. `gateway`, `hub`, etc., we must deal with the coexistance of both hosts for a period while we incrementally migrate services from old to new. This is why, at this point and in this instance, the new host is still for now identified by its hostname in the hosts inventory while the old host is identified by its role name.

### Install service on new host

First, temporarily alter the `install-services.yml` playbook to be able to install services on the new host.

For example, to install the calendar service on a new host with the hostname `europe`, change:

```yaml
- hosts: dns

  tasks:

    - ansible.builtin.import_role:
        name: calendar
      tags: calendar
```

To:

```yaml
- hosts: europe

  tasks:

    - ansible.builtin.import_role:
        name: calendar
      tags: calendar
```

You can then install the calendar service on the new host:

```bash
ansible-playbook --limit europe --tags calendar ./playbooks/install-services.yml
```



stop service




```bash
ansible-playbook --limit hub --tags calendar --start-at-task 'Install APT package(s) required by the calendar role' ./playbooks/install-services.yml
```

add host to dns_host_patterns and host_to_roles map (can be empty at this point)

ansible-playbook --limit hub --tags dns ./playbooks/install-services.yml

ansible-playbook --limit hub --tags backup ./playbooks/install-services.yml



