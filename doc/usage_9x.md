
Usage for MOS 9.x
=================

1. Preparations
---------------

For the first step you should make a perform a preparation playbook:
```
ansible-playbook playbooks/mos9_prepare.yml -e '{"env_id":<env_id>}'
```

2. Check configuration customizations
-------------------------------------

Run the configuration check on your environment using Noop run:
```
fuel2 env redeploy --noop <env_id>
```
To verify the deployment task status, run **fuel2 task show <TASK_ID>**.
The task ID is specified in the output of the configuration check command.

Then verify the task summary:
```
fuel2 task history show <TASK_ID> --include-summary -f yaml | python ./tool/parse_report.py
```

The Noop run reports are stored on each OpenStack node in the
/var/lib/puppet/reports/<NODE-FQDN>/<TIMESTAMP>.yaml directory.

**Warning**

All configuration and arhcitecture environment customizations might be lost
during the update process.
Therefore, use the customization detection reports made by Noop run to make
a decision on whether it is worth proceeding with the update.


3. Gathering code customizations
--------------------------------
```
ansible-playbook playbooks/gather_customizations.yml -e '{"env_id":<env_id>}'
```
More details you can find here:
[gather_customizations.yml](doc/architecture.md#gather_customizationsyml)

During performing all steps you can use the flags for step management:

[Apply MU steps](playbooks/vars/steps/apply_mu.yml)

For example you can **skip health check step, repeat generation of APT files
or gathering customizations one more time**.
```
ansible-playbook playbooks/gather_customizations.yml -e '{"env_id":<env_id>,"gather_customizations":true}'
```
Sometimes playbooks can stop with fail and recommend to use
some flags for the solving the situation, for example, when patch
were applied not on all nodes.

4. Verify patches
-----------------

Then check that all customizations are applied on new versions:
```
ansible-playbook playbooks/verify_patches.yml -e '{"env_id":<env_id>}'
```
More details you can find here:
[verify_patches.yml](doc/architecture.md#verify_patchesyml)

After that please go to the nodes **/root/mos_mu/verification/** and make sure
that all patches are applied correctly.

It is also strongly recommended to identify and copy original patches to
**patches** folder on Fuel and disable **use_current_customization** flag and
manage patches to successfully execute previous **verify_patches.yml** step.

5. Update Fuel node
-------------------

Update the Fuel Master node packages, services, and configuration:
```
ansible-playbook playbooks/update_fuel.yml
```
More details you can find here:
[update_fuel.yml](doc/architecture.md#update_fuelyml)

**Warning**

During the update procedure, the Fuel Master node services will be restarted
automatically.

6. Update environment
---------------------
Update the Fuel Slave nodes using the command below:
```
fuel2 update --env <ENV_ID> install
```
Optionally, add the --restart-rabbit and --restart-mysql arguments to the
command to restart RabbitMQ and MySQL automatically. Otherwise, these services
will not be restarted unless their configurations change during the update.

To verify the update progress, use the Fuel web UI Dashboard tab or run
`fuel2 task show <TASK_ID>`. The task ID is specified in the output of the
`fuel2 update –env <ENV_ID>` install command.

7. Apply patches
----------------
```
ansible-playbook playbooks/mos9_apply_patches.yml -e '{"env_id":<env_id>}'
```
More details you can find here:
[apply_mu.yml](doc/architecture.md#apply_muyml)

This playbook apply gathered customizations and patches from **patches** folder
from Fuel on each node and then restarts OpenStack services.

Full restart
------------

For the applying updates for some services like CEPH, QEMU and etc we would
recommend to restart them manually and at the same time monitor thier status.

Which services were updated you can find in output in section
**Show upgradable packages**

Please also read the upgrade section in documentation for these services.

Rollback
--------

Rollback is not possible for MOS 9.x releases.

Local repos mirrors
-------------------

If nodes don't have access to http://mirror.fuel-infra.org/ you can create and sync
local mirrors:
```
ansible-playbook playbooks/create_mirrors.yml -e '{"env_id":<env_id>}'
```

Then for the using them you to add **fuel_url** external variable:
```
ansible-playbook playbooks/apply_mu.yml -e '{"env_id":<env_id>,"fuel_url":"http://<FUEL_IP>:8080"}'
```

