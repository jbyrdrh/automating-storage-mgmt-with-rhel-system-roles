# Automating storage management with RHEL System Roles
This demo is a RHEL 9 update of a previously posted blog on redhat.com.

- Learn how the storage system role can automate the configuration of logical volume manager (LVM) volume groups, logical volumes, and filesystems.

Automation can help increase efficiency, save time, and improve consistency, which is why Red Hat Enterprise Linux (RHEL) includes features that help automate many tasks. [RHEL System Roles](https://www.redhat.com/en/blog/rhel-system-roles) is a collection of Ansible content included in RHEL to help provide consistent workflows and streamline the execution of many manual tasks.

RHEL contains a storage system role that can automate the configuration of logical volume manager (LVM) volume groups, logical volumes, and filesystems. The storage role supports creating filesystems on unpartitioned disks without the use of LVM as well. In addition, the storage role can automate advanced storage functionality such as encryption, deduplication, compression, and RAID.

The “Managing local storage using RHEL System Roles” documentation includes information and example playbooks covering how to use the storage system role in many different scenarios.
This post will focus on using the storage role across systems where the storage device names might not always be consistent. For example, if you would like to use the storage role to implement new volume groups, logical volumes, and filesystems across many RHEL hosts, and these hosts have inconsistent storage device names.

# Environment overview
In my example environment, I have a control node system named `controlnode` and two managed nodes: `rhel9-server1` and `rhel9-server2`, all running RHEL 9.4.
These servers have the following storage devices:

- rhel9-server1
  - sda (25GB) used for OS volume group
  - sdb (20GB)
  - sdc (10GB)
  - sdd (15GB)
  - sde (10GB)
  - sdf (15GB)

- rhel9-server2
  - sda (20GB) used for OS volume group
  - sdb (15GB)
  - sdc (10GB)
  - sdd (10GB)
  - sde (15GB)

As you can see, each server has two 10GB disks and two 15 GB disks, however the device names of these storage devices are not consistent between the servers.
NOTE: Path-based addresses (e.g. `/dev/sdX`) are [not persistent per kernel design](https://access.redhat.com/solutions/3962551#root_cause), so I did have to work with the disk layout on rhel9-server1 which involved a few reboots for the kernel to discover the disks in the desired order, specifically `sdb` and `sdf`.

In this example, I would like to setup the following new volume groups, logical volumes, and filesystems:

- `web_vg` volume group, using the two 10 GB disks
  - With a web_lv logical volume, using all of the space in the volume group, mounted at /web
- `database_vg` volume group, using the two 15GB disks
  - With a database_lv logical volume, using 50% of the space in the volume group, mounted at /database
  - With a backup_lv logical volume, using 20% of the space in the volume group, mounted at /backup
  - 30% of the space in the volume group should be left free for future expansion

Normally when using the storage system role, a list of disk devices is supplied to the role. For example, a basic playbook to create the `database_vg` volume group, logical volumes, and filesystems on rhel9-server1 using the two 15GB disks would contain:

~~~
- hosts: rhel9-server1
  vars:
    storage_pools:
      - name: database_vg
        disks: 
          - sdd
          - sdf
        volumes:
          - name: database_lv
            size: 50%
            mount_point: "/database"
            state: present
          - name: backup_lv
            size: 20%
            mount_point: "/backup"
            state: present
  roles:
    - redhat.rhel_system_roles.storage
~~~

While this playbook would work on `rhel9-server1` where the 15GB disks have the `sdd` and `sdf` device names, it would fail on `rhel9-server2` as there is not a `sdf` device on this host. In addition, while the `sdd` device does exist on `rhel9-server2`, it is 10GB, and I wanted this volume group placed on the 15GB disks.

It would be possible to create a `host_vars` directory, and define variables that are specific to each host. This would allow me to specify which disks should be used on each host. However if you have more than a few hosts, manually gathering the disk information from each host and specifying it in a `host_vars` file for each host quickly becomes impractical.

In this scenario, what I would really like is to have Ansible dynamically find the disks using Ansible facts.

# Ansible facts
By default, when Ansible runs a playbook on hosts, the first task will be to gather facts from each host. These facts include a significant amount of information about each host, including information on storage devices.
On my control node, I have an inventory file named inventory.yml with the following entries:
all:
  hosts:
    rhel9-server1:
    rhel9-server2:

If using Ansible automation controller as your control node, this Inventory can be imported into Red Hat Ansible Automation Platform via an SCM project (example GitHub or GitLab) or using the awx-manage Utility as specified in the documentation.
From the controlnode, I can quickly take a look at what facts Ansible gathers from the rhel9-server1 host by running:
$ ansible rhel9-server1 -m setup -i inventory.yml


The gathered facts are displayed, including a list of ansible_devices. In this list, each disk device has an entry, such as:
            "sdc": {
                "holders": [],
                "host": "SCSI storage controller: Red Hat, Inc. Virtio SCSI (rev 01)",
                "links": {
                    "ids": [
                        "scsi-0QEMU_QEMU_HARDDISK_1184e4e2-0ac2-4aed-b162-10b1f0d0a9cc",
                        "scsi-SQEMU_QEMU_HARDDISK_1184e4e2-0ac2-4aed-b162-10b1f0d0a9cc"
                    ],
                    "labels": [],
                    "masters": [],
                    "uuids": []
                },
                "model": "QEMU HARDDISK",
                "partitions": {},
                "removable": "0",
                "rotational": "1",
                "sas_address": null,
                "sas_device_handle": null,
                "scheduler_mode": "mq-deadline",
                "sectors": "20971520",
                "sectorsize": "512",
                "serial": "1184e4e2",
                "size": "10.00 GB",
                "support_discard": "4096",
                "vendor": "QEMU",
                "virtual": 1

Included in the facts is information on the size of the disk in the size field, as well as information on holders, links, and partitions.
The holders, links, and partitions fields can be used as an indication to help determine if the disk possibly contains data. As you build a playbook to select the disks that should be used by the storage role, you might want to use these fields to help exclude existing disks that might already contain data. However, this type of logic would not be idempotent, as on the first run the storage role would configure storage on these unused devices. On subsequent runs, the playbook would no longer be able to find unused disks and would fail.
In the example presented in this blog post, the storage role will control all storage on the systems, except for the disk that the operating systems (OSs) boot from (the sda device on all managed nodes), so I am not concerned about selecting disks that might already contain data.
Note that extreme care must be taken to ensure that you don’t inadvertently identify disks for the storage role to use that contain data, which might result in data loss.
Using Ansible facts to select disks for storage role use
In this example, I would like to find and use the two 10GB disks for the web_vg volume group, and find and use the two 15GB disks for the database_vg volume group. In my environment, all storage devices start with sd, and my OS is installed on sda on all servers, so I would like to exclude sda when searching for disks to use.
Again, in this environment the storage role is managing all storage on the system other than the sda device, so I am not concerned with the playbook finding and using disks that already contain data. If your environment is not fully managed by the storage role, additional precautions should be taken to ensure the playbook doesn’t use disks that might already contain data, which could result in data loss.
Based on my environment, I can create a playbook to locate the disks with my criteria:

- hosts: all
  tasks:
    - name: Identify disks that are 10 GB for web_vg
      set_fact:
        web_vg_disks: "{{ ansible_devices | dict2items | selectattr('key', 'match', '^sd.*') | rejectattr('key', 'match', '^sda$') | selectattr('value.size', 'match', '^10.00 GB') | map(attribute='key') | list }}"

    - name: Identify disks that are 15 GB for database_vg
      set_fact:
        database_vg_disks: "{{ ansible_devices | dict2items | selectattr('key', 'match', '^sd.*') | rejectattr('key', 'match', '^sda$') | selectattr('value.size', 'match', '^15.00 GB') | map(attribute='key') | list }}"

    - name: Show value of web_vg_disks
      ansible.builtin.debug:
        var: web_vg_disks

    - name: Show value of database_vg_disks
      ansible.builtin.debug:
        var: database_vg_disks

The first task identifies ansible_devices that start with the device name sd (excluding sda), and identifies the disks that are 10GB in size. These identified disks are assigned to the web_vg_disks list variable.
The second task works in the same way, identifying 15GB disks for the database_vg volume group and stores the list of identified disks in the database_vg_disks variable.
The third and forth tasks display the contents of the web_vg_disks and database_vg_disks variables.
When run, the playbook shows:
[ansible@controlnode storage]$ ansible-playbook -i inventory.yml 1st-playbook.yml 

PLAY [all] *********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [rhel9-server2]
ok: [rhel9-server1]

TASK [Identify disks that are 10 GB for web_vg] ********************************
ok: [rhel9-server2]
ok: [rhel9-server1]

TASK [Identify disks that are 15 GB for database_vg] ***************************
ok: [rhel9-server1]
ok: [rhel9-server2]

TASK [Show value of web_vg_disks] **********************************************
ok: [rhel9-server1] => {
    "web_vg_disks": [
        "sde",
        "sdc"
    ]
}
ok: [rhel9-server2] => {
    "web_vg_disks": [
        "sdd",
        "sdc"
    ]
}

TASK [Show value of database_vg_disks] *****************************************
ok: [rhel9-server1] => {
    "database_vg_disks": [
        "sdf",
        "sdd"
    ]
}
ok: [rhel9-server2] => {
    "database_vg_disks": [
        "sdb",
        "sde"
    ]
}


The playbook correctly identified the disks I would like to use on each server.
Adding storage configuration to the playbook
Now that I have logic to identify the disks I would like to use, the next step is to utilize the storage role to implement my desired volume group, logical volume, and filesystem configuration.
I’ll add another task to the playbook to call the storage role. The complete playbook, named storage.yml is:

- hosts: all
  tasks:
    - name: Identify disks that are 10 GB for web_vg
      set_fact:
        web_vg_disks: "{{ ansible_devices | dict2items | selectattr('key', 'match', '^sd.*') | rejectattr('key', 'match', '^sda$') | selectattr('value.size', 'match', '^10.00 GB') | map(attribute='key') | list }}"

    - name: Identify disks that are 15 GB for database_vg
      set_fact:
        database_vg_disks: "{{ ansible_devices | dict2items | selectattr('key', 'match', '^sd.*') | rejectattr('key', 'match', '^sda$') | selectattr('value.size', 'match', '^15.00 GB') | map(attribute='key') | list }}"

    - name: Show value of web_vg_disks
      ansible.builtin.debug:
        var: web_vg_disks

    - name: Show value of database_vg_disks
      ansible.builtin.debug:
        var: database_vg_disks

    - name: Run storage role
      vars:
        storage_pools:
          - name: web_vg
            disks: "{{ web_vg_disks }}"
            volumes:
              - name: web_lv
                size: 100%
                mount_point: "/web"
                state: present
          - name: database_vg
            disks: "{{ database_vg_disks }}"
            volumes:
              - name: database_lv
                size: 50%
                mount_point: "/database"
                state: present
              - name: backup_lv
                size: 20%
                mount_point: "/backup"
                state: present
      ansible.builtin.include_role:
        name: redhat.rhel_system_roles.storage

The Run storage role task defines the storage_pool variable to specify my desired storage configuration. It specifies that a web_vg volume group should be created, with a web_lv logical volume utilizing 100% of the space, and with a filesystem mounted at /web. The volume group should use the disks listed in the web_vg_disks variable which the previous task defined based on the discovered disks that met the specified criteria.
It similarly specifies that the database_vg volume group should be created, with a database_lv logical volume using 50% of the space, and with a filesystem mounted at /database. There should also be a backup_lv logical volume, using 20% of the space, with a filesystem mounted at /backup. The volume group should use the disks listed in the database_vg_disks variable which the previous task defined based on the discovered disks that met the criteria.
If you are using Ansible automation controller as your control node, you can import this Ansible playbook into Red Hat Ansible Automation Platform by creating a Project, following the documentation provided here. It is very common to use Git repos to store Ansible playbooks. Ansible Automation Platform stores automation in units called Jobs which contain the playbook, credentials and inventory. Create a Job Template following the documentation here.
Running the playbook
At this point, everything is in place, and I’m ready to run the playbook. For this demonstration, I’m using a RHEL control node and will run the playbook from the command line. I’ll use the cd command to move into the storage directory, and then use the ansible-playbook command to run the playbook.
[ansible@controlnode ~]$ cd storage/
[ansible@controlnode storage]$ ansible-playbook -i inventory.yml storage.yml

I specify that the storage.yml playbook should be run and that the inventory.yml file should be used as my Ansible inventory (the -i flag).
After the playbook completes, I need to verify that there were no failed tasks:
PLAY RECAP ***********************************************************
rhel9-server1   : ok=25   changed=3    unreachable=0    failed=0    skipped=11   rescued=0    ignored=0   
rhel9-server2   : ok=25   changed=3    unreachable=0    failed=0    skipped=11   rescued=0    ignored=0
If you are using Ansible automation controller as your control node, you can launch the job from the automation controller web interface.
Validating the configuration
To validate the configuration I’ll run the lsblk command on each of the managed nodes:
[ansible@controlnode storage]$ ansible rhel9-server1 -m ansible.builtin.command -a 'lsblk' -i inventory.yml
rhel9-server1 | CHANGED | rc=0 >>
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   25G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0  200M  0 part /boot/efi
├─sda3                      8:3    0  500M  0 part /boot
└─sda4                      8:4    0  9.3G  0 part /
sdb                         8:16   0   20G  0 disk 
sdc                         8:32   0   10G  0 disk 
└─web_vg-web_lv           253:2    0   20G  0 lvm  /web
sdd                         8:48   0   15G  0 disk 
└─database_vg-database_lv 253:1    0   15G  0 lvm  /database
sde                         8:64   0   10G  0 disk 
└─web_vg-web_lv           253:2    0   20G  0 lvm  /web
sdf                         8:80   0   15G  0 disk 
└─database_vg-backup_lv   253:0    0    6G  0 lvm  /backup
sr0                        11:0    1  8.9G  0 rom  

[ansible@controlnode storage-demo]$ ansible rhel9-server2 -m ansible.builtin.command -a 'lsblk' -i inventory.yml
rhel9-server2 | CHANGED | rc=0 >>
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   20G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0  200M  0 part /boot/efi
├─sda3                      8:3    0  500M  0 part /boot
└─sda4                      8:4    0  9.3G  0 part /
sdb                         8:16   0   15G  0 disk 
└─database_vg-backup_lv   253:0    0    6G  0 lvm  /backup
sdc                         8:32   0   10G  0 disk 
└─web_vg-web_lv           253:2    0   20G  0 lvm  /web
sdd                         8:48   0   10G  0 disk 
└─web_vg-web_lv           253:2    0   20G  0 lvm  /web
sde                         8:64   0   15G  0 disk 
└─database_vg-database_lv 253:1    0   15G  0 lvm  /database

On both servers, I can see that the web_vg volume group was set up on the two 10GB disks, and the database_vg volume group was set up on the two 15GB disks. I can also validate the sizes of the logical volumes match my desired configuration.
Next Steps
This playbook is idempotent, meaning I can re-run it, and the tasks will not change anything again unless needed to bring the system to the desired state.
On the database_vg, my logical volumes only added up to use 70% of the available space in the volume group (50% went to databale_lv, and 20% went to backup_lv). This means that there is 9GB free in this volume group on each host. If I would like to increase one of these logical volumes and the corresponding filesystem, I can simply go into the playbook and update the percentage of space that should be allocated.
For example, if I increase the database_lv size from 50% to 60% in the playbook, and then re-run the playbook the database_lv and corresponding /database filesystem will be increased from 15GB to 18GB.
Conclusion
The storage RHEL System Role can help you quickly and consistently configure storage in your RHEL environment.
Red Hat offers many RHEL System Roles that can help automate other important aspects of your RHEL environment. To explore additional roles, review the list of available RHEL System Roles and start managing your RHEL servers in a more efficient, consistent and automated manner today.
Want to learn more about the Red Hat Ansible Automation Platform? Check out our e-book, The automation architect's handbook.
