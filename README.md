ansible_roles_proxmox_guest_update_with_snapshot
=========
This role relies on Proxmox API to snapshot the guest before patching it. On a defined number of VMIDs, a snapshot will be taken, a cron will be scheduled to clean up the snapshot at a set date and, the guest will be patched and optionally rebooted if the Kernel got an update. The different options are available through tags.


IMPORTANT Requirements
------------
This role requires at minimum 2 inputs.
        - The ansible inventory for the update
        - The VMIDs for the snapshot tasks

For security reasons, it's not possible to gather the VMID from the guest os. Both inventories have to be set in the playbook. There is 3 way to do this.

The easy way: 

Have a second entry in your ansible inventory with the VMID in Proxmox. You can then use the same value for both the playbook inventory and the pve_vm_ids variable.
```
        - hosts: "{{ range(100, 160) | map('string') | list }}"
          vars:
            pve_vm_ids:
              - "{{ range(100, 160) | map('string') | list }}"
          roles:
              - [...]
```

The dynamic way:

This way is more "heavy" but you can gather each host IP from their VMID through the API, add each IP to the play and run the update on this.

```
        - hosts: localhost
          gather_facts: no
          tasks:
            - name: Get Guest IP
              ignore_errors: yes
              uri:
                url: "https://<pve>.<domain.loc>:8006/api2/json/nodes/<pve>/qemu/{{ item }}/agent/network-get-interfaces"
                method: GET
                headers:
                  Content-Type: application/json
                  Authorization: PVEAPIToken=<api_user>!<api_token_id>=<api_token_secret>
                status_code: 200
                validate_certs: no
              register: guest_interfaces
              loop: "{{ range(100, 160) | map('string') | list }}"
            - name: Add ips to the play
              add_host:
                name: "{{ item.json.data.result[1]['ip-addresses'][0]['ip-address'] }}"
                group: tmp_patching_group
              loop: "{{ guest_interfaces.results }}"

        - name: Run the role with the added IPs
          hosts: tmp_patching_group
          roles:
            - [...]
```

The hard way:

Populate both inventories manually and make sure your ansible inventory list matches the IDs referenced in the pve_host_id variable.


Additional requirements
------------
The guest should be running otherwise the update task will fail. Snapshot tasks are independent of the update, even if the host is offline, the snapshot will be taken. To remove the snapshot, the same playbook and its inventory will be re-used. DO NOT remove or change the playbook path or inventory otherwise, the snapshot will not be removed. The playbook to be re-used need to be referenced in the variable: playbook_path (in defaults)


Tags
------------
Tags are mandatory to run this role otherwise no action will be taken.

--tags update

Run dnf update / apt upgrade with no exclusion. If you want to exclude some packages or limit the repos used by dnf, edit the task "Update RHEL based OS" and "Update Debian based OS". ansible-doc dnf and ansible-doc apt are your friends.

--tags kernelup

No need to have both tags kernelup and update. kernelup will run the same update tasks. Run dnf update / apt upgrade with no exclusion. If you want to exclude some packages or limit the repos used by dnf, edit the task "Update RHEL based OS" and "Update Debian based OS". ansible-doc dnf and ansible-doc apt are your friends. Scan all hosts and reboot the ones with new available kernel versions.

--tags snaprm

This one does not need to be run manually. It will be used by the cron schedule during the upgrade. The cron will be scheduled according to the value set in the defaults variables.

Defaults variables
------------
playbook_path: Reference the playbook path here. Mandatory for the snapshot removal task. pve_vm_ids: List all vmids here. It has to be a list because the role will loop into each id. You can use multiple ranges.
```
  - "{{ range(100, 160) | map('string') | list }}"
  - "{{ range(200, 880) | map('string') | list }}"
  - 999
```

snapshot_name: Guest snapshot name used at creation and removal.

snapshot_cron_file: Cron file name for the snapshot removal in /etc/cron.d

snapshot_rm_hour: Snapshot removal cron hour 

snapshot_rm_minute: Snapshot removal cron minute of the hour 

snapshot_removal_delay: Snapshot removal delay in days after patching


Vars Variables
------------
All API login info is in there. It's using the API token authentication process. You have to provide the user id, user token id and secret.

Optional dependencies
------------
This role can be combined with this other role: proxmox_offline_guest_power_on
If you have any offline guests that you would like to patch at the same time. The above role combined with this one is a great way to patch everything in a single window.

Example Playbook
------------
If the following does not make sense, please read the IMPORTANT Requirements.
The easy way:
```
        - hosts:
            - "{{ range(100, 160) | map('string') | list }}"
            - "{{ range(200, 880) | map('string') | list }}"
            - 999
          vars:
            pve_vm_ids:
              - "{{ range(100, 160) | map('string') | list }}"
              - "{{ range(200, 880) | map('string') | list }}"
              - 999
            playbook_path: /path/to/your/playbook.yml
          roles:
              - victorsierra314.ansible_role_proxmox_guest_update_with_snapshot
```

The dynamic way:
```
        - hosts: localhost
          gather_facts: no
          tasks:
            - name: Get Guest IP
              ignore_errors: yes
              uri:
                url: "https://<pve>.<domain.loc>:8006/api2/json/nodes/<pve>/qemu/{{ item }}/agent/network-get-interfaces"
                method: GET
                headers:
                  Content-Type: application/json
                  Authorization: PVEAPIToken=<api_user>!<api_token_id>=<api_token_secret>
                status_code: 200
                validate_certs: no
              register: guest_interfaces
              loop:
                - "{{ range(100, 160) | map('string') | list }}"
                - "{{ range(200, 880) | map('string') | list }}"
                - 999
            - name: Add ips to the play
              add_host:
                name: "{{ item.json.data.result[1]['ip-addresses'][0]['ip-address'] }}"
                group: tmp_patching_group
              loop: "{{ guest_interfaces.results }}"

        - name: Run the role with the added IPs
          hosts: tmp_patching_group
          vars:
            pve_vm_ids:
              - "{{ range(100, 160) | map('string') | list }}"
              - "{{ range(200, 880) | map('string') | list }}"
              - 999
            playbook_path: /path/to/your/playbook.yml
          roles:
              - victorsierra314.ansible_role_proxmox_guest_update_with_snapshot
```

The hard way:
```
       - hosts: host_group:host1:host2 
         vars: 
           pve_vm_ids: 
             - "{{ range(100, 160) | map('string') | list }}" 
             - "{{ range(200, 880) | map('string') | list }}" 
             - 999 
           playbook_path: /path/to/your/playbook.yml 
         roles: 
             - victorsierra314.ansible_role_proxmox_guest_update_with_snapshot 
```

License
--------------
BSD

Author Information
--------------
I have been working with Proxmox for quite some time now. Took me some time to get to gibhut but I'm glad I can share what I have built over the year. Enjoy.
