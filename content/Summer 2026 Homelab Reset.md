I need to clean up and pay all the infrastructure tech debt to make my services significantly easier to stand up through automation. Since I plan to spin up my Kubernetes cluster again, I can also prepare for that.
# Goals & Plans
- [x] Clean up virtual machines
  - [x] Condense all Docker virtual machines into one
  - [x] Move volume mounts into iSCSI share
- [x] Automate more
  - [x] Move compose files to Github
  - [x] Use templates for VM
    - [x] Debian cloud images
    - [x] Cloud Init
  - [x] Ansible playbooks
- [x] Prepare for Kubernetes
# Machine Modifications
For my servers, I currently have 3 machines:
## Coconut
This is my main compute. It is running [Proxmox VE](https://pve.proxmox.com/wiki/Main_Page).
The machine is running an AMD Ryzen 5700G with 128GB of DDR4 RAM.
I have added a 16GB Intel Optane for boot, leaving the existing 480GB SSD mirror for local storage.
This machine will run all my Docker Compose services on a Debian cloud VM, and eventually my Kubernetes cluster for learning.
### Networking and Storage
- I moved to a static IPv4 and IPv6 address instead of the old DHCP reservation and SLAAC.
- Mounting the VM backups and ISOs over NFSv4.2.
- Mounting all persistent VM disks over iSCSI.
## Peanut
This is my main storage. It is running [TrueNAS Scale](https://www.truenas.com/docs/).
The machine is the [SuperServer 6029P-TRT](https://www.supermicro.com/en/products/system/2u/6029/sys-6029p-trt.php).
I have added a 16GB Intel Optane for boot, moved the 4x8TB HDDs from [[#Walnut]], added a 1TB NVMe mirror, and a 240GB SSD mirror.
This machine will store:
- All my media, which will be accessible over NFS and SMB
  - Videos on the HDDs
  - Small misc files on the NVMe
- Persistent VM disks, over iSCSI
  - Docker volumes on the SSDs
### Fan Control
To quiet down the fans, I used an IPMI command to set the fan speed manually.
```sh
ip='127.0.0.1';
username='adminuser123';
password='Thisismypassword123';
ipmitool -I lanplus -H $ip -U $username -P $password -C 3 raw 0x30 0x45 0x01 0x01;
sleep 1;
ipmitool -I lanplus -H $ip -U $username -P $password -C 3 raw 0x30 0x70 0x66 0x01 0x00 0xF;
ipmitool -I lanplus -H $ip -U $username -P $password -C 3 raw 0x30 0x70 0x66 0x01 0x01 0xF
```
- The first command sets the fan controller into manual mode.
- The second and third commands set the fan speed to 15% (0xF) for zone 0 (0x0) and zone 1 (0x1).
### Networking
- I moved to a static IPv4 and IPv6 address instead of the old DHCP reservation and SLAAC.
### Encryption
- Since the parent pool is just another dataset, I decided to disable encryption on the root and cherry-pick which child datasets to encrypt.
- I had the option to use either passphrase or key encryption.
  - With key encryption, TrueNAS generates and stores the key on the boot pool so it can unlock datasets on startup.
  - With a passphrase, TrueNAS cannot unlock the datasets on boot, leaving the user to manually unlock them.
- I chose passphrase encryption since I didn't want to manage disaster recovery with a key.
### SSD
- I enabled auto TRIM for my SSD pools.
### Scrubbing
- I set the scrub schedule to `00 1 * * tue`, which triggers every Tuesday at 1 AM.
- I set the `Threshold Days` to 28. Every time the schedule triggers, it checks if 28 days have elapsed since the last scrub and decides whether to run. This means scrubs should run on Tuesdays at 1 AM, roughly once a month.
### Backups and Snapshotting
- I connected my Google Drive account and created a Cloud Sync task to mirror game saves into the cloud.
- Snapshot tasks run at 11 PM each day.
### ACLs and Sharing
- For most new datasets shared over NFS or SMB, I used the `SMB/NFSv4` `ACL Type` with `Passthrough` for the `ACL Mode`. This was [recommended by TrueNAS](https://www.truenas.com/docs/scale/datasets/permissions/configuringacls/) when using SMB shares.
- For each dataset, I left the default user and group owner as root and added my own user with `Full Control`.
- For each container running inside my Docker VM, I created a user corresponding to its function (Jellyfin, Syncthing, etc.).
- I created a unique share for each directory I wanted to share (previously I shared the root dataset).
  - For SMB shares, I set the `Purpose` to `Multi-Protocol Share`, since most shares were also accessed via NFSv4.
  - For NFS shares, I had to keep both `NFSv3` and `NFSv4` enabled under `Enabled Protocols` so that `showmount -e` (which uses NFSv3) would work for debugging.
    - Since Proxmox seems to use the root user as the connecting user, and TrueNAS has root squashing enabled by default (mapping root to the nobody user), I had to set the `maproot` user and group to root on the share. This aligns with root owning the dataset.
  - For iSCSI, I [followed the setup wizard](https://www.truenas.com/docs/scale/25.10/scaletutorials/shares/iscsi/addingiscsishares/index.html).
## Walnut
This is my backup storage. All pools from [[#Peanut]] will be sent via ZFS replication.
The machine is running an AMD Ryzen 5600G with 16GB of DDR4 RAM.
I moved the 8x4TB HDDs from [[#Peanut]] and merged them into one ZFS raidz2 pool.
### Replication
- For replication, I went through the wizard for each pool, making sure to:
  - Enable recursive so it captures snapshots of each child dataset.
  - Enable raw replication so the encrypted dataset is sent as-is without being decrypted, preserving the passphrase from the source.
  - Set the snapshot lifetime to a year to keep a longer history than my main storage.
  - Schedule it to run at 12 AM every day.
### Networking
- I moved to a static IPv4 and IPv6 address instead of the old DHCP reservation and SLAAC.
### BIOS Fan Curve
Monitoring Source: Monitor M/B

| Point | Temp | PWM |
|-------|------|-----|
| P1 | 20°C | 10% |
| P2 | 35°C | 12% |
| P3 | 45°C | 14% |
| P4 | 55°C | 35% |
| P5 | 65°C | 100% |
# Virtual Machines
## Previous Setup
I previously had separate VMs for Docker containers:
- **Reverse Proxy:** Nginx Proxy Manager
- **Media:** Jellyfin, Shoko, Qbittorrent
- **Files:** Vaultwarden, Syncthing
- **Misc:** Uptime Kuma (and some other random containers)
I was running Portainer on the reverse proxy machine and Portainer agents on the others. For persistent storage, each VM stored everything locally and I had Proxmox back up each VM onto [[#Peanut]].
## Current Setup
The plan is to have one VM for all my Docker containers, with persistent storage on an iSCSI share instead of locally, and a Docker network so I only need to expose my reverse proxy.
Some reasoning behind this:
- Single point of contact, which means no more tracking multiple IPs and reservations for Docker containers.
- Persistent storage is uncoupled from compute, allowing quick service spin-up and VM upgrades.
- Fewer VMs to manage.
## VM Templates
I used [this resource](https://www.apalrd.net/posts/2023/pve_cloud/) for creating Debian cloud image templates on Proxmox.
The Cloud-Init configuration is very basic and I chose to use Ansible for post-provisioning instead of creating custom Cloud-Init snippets in [Proxmox's Cloud-Init](https://pve.proxmox.com/wiki/Cloud-Init_Support).
The idea is to have a good base image for a VM, with my keys and networking already configured, then add the machine to my Ansible inventory and run the specific playbooks to configure it further.
Currently I have two VMs:
- Docker host
- [Pelican Wings](https://pelican.dev/docs/wings/install/) node
I also have an LXC container running the Omada SDN controller, which I haven't moved into a Docker container yet since I wanted to isolate it with its own Proxmox bridge network (see [[Omada SDN Controller Install]]).
Below is the script I used to create the template:
```sh
#!/bin/bash

function create_template() {
    echo "Creating template $2 ($1)"

    qm create $1 --name $2 --ostype l26
    qm set $1 --net0 virtio,bridge=vmbr0
    qm set $1 --serial0 socket --vga serial0
    qm set $1 --memory 1024 --cores 4 --cpu host
    qm set $1 --scsi0 ${storage}:0,import-from="$(pwd)/$3",discard=on
    qm set $1 --boot order=scsi0 --scsihw virtio-scsi-single
    qm set $1 --agent enabled=1,fstrim_cloned_disks=1
    qm set $1 --ide2 ${storage}:cloudinit
    qm set $1 --ipconfig0 "ip6=auto,ip=dhcp"
    qm set $1 --sshkeys ${ssh_keyfile}
    qm set $1 --ciuser ${username}
    qm disk resize $1 scsi0 20G
    qm template $1

    rm $3
}

export ssh_keyfile=/root/ansible_key.pub
export username=steven
export storage=coconut-storage

wget "https://cloud.debian.org/images/cloud/trixie/latest/debian-13-genericcloud-amd64.qcow2"
create_template 888 "debian-13-template" "debian-13-genericcloud-amd64.qcow2"
```
For the Docker host, I had to manually add the iSCSI share and attach it as a virtio drive, which Ansible will format (if needed) and mount.
## Ansible Automation
### Playbooks
#### docker-host.yaml
```yaml
---
- name: Get the Docker Hosts VM ready
  hosts: docker-host
  become: true
  gather_facts: true
  vars:
    mount_path: "/mnt/docker-volumes"
    target_device: "/dev/vda"
    partition_number: 1
    fstype: "ext4"
  tasks:
    - import_tasks: generic/update-all.yaml
    - import_tasks: generic/install-qemu-guest-agent.yaml
    - import_tasks: docker-host/mount-external.yaml
    - import_tasks: generic/install-docker.yaml
    - import_tasks: docker-host/copy-start-dockhand.yaml
    - import_tasks: generic/reboot.yaml
```
#### wings.yaml
```yaml
---
- name: Install Wings and Mount Remote Backup
  hosts: wings-node-0
  become: true
  gather_facts: true
  tasks:
    - import_tasks: generic/update-all.yaml
    - import_tasks: generic/install-qemu-guest-agent.yaml
    - import_tasks: generic/install-docker.yaml
    - import_tasks: wings/install-wings.yaml
    - import_tasks: wings/mount-backups.yaml
```
### Inventory
```yaml
---
servers:
  vars:
    ansible_ssh_private_key_file: ~/.ssh/temp
  hosts:
    docker-host:
      ansible_host: 10.0.10.100
      ansible_user: steven
    wings-node-0:
      ansible_host: 10.0.10.101
      ansible_user: steven
```
### Task Files
#### generic/update-all.yaml
```yaml
---
  - name: Update apt cache
    ansible.builtin.apt:
      update_cache: true
      cache_valid_time: 3600
  - name: Upgrade packages
    ansible.builtin.apt:
      upgrade: safe
  - name: Clean obsolete apt cache
    ansible.builtin.apt:
      autoclean: true
  - name: Check if reboot is required
    ansible.builtin.stat:
      path: /var/run/reboot-required
    register: reboot_required
  - name: Notify if reboot is required
    ansible.builtin.debug:
      msg: "Reboot required on {{ inventory_hostname }}"
    when: reboot_required.stat.exists
```
#### generic/install-qemu-guest-agent.yaml
```yaml
---
  - name: Install QEMU Guest Agent
    ansible.builtin.apt:
      update_cache: true
      pkg:
        - qemu-guest-agent
  - name: Start and enable QEMU Guest Agent
    ansible.builtin.systemd_service:
      name: qemu-guest-agent
      state: started
      enabled: true
```
#### generic/install-docker.yaml
```yaml
---
  - name: Create Docker configuration directory
    ansible.builtin.file:
      path: /etc/docker
      state: directory
      owner: root
      group: root
      mode: "0755"
  - name: Configure Docker daemon
    ansible.builtin.copy:
      dest: /etc/docker/daemon.json
      owner: root
      group: root
      mode: "0644"
      content: |
        {
          "userland-proxy": false,
          "log-driver": "local",
          "log-opts": {
            "max-size": "20m",
            "max-file": "5"
          }
        }
  - name: "Add Docker's official GPG key"
    block:
      - name: Install prerequisites
        ansible.builtin.apt:
          update_cache: true
          pkg:
            - ca-certificates
            - curl
  - name: Create secure apt keyrings directory
    ansible.builtin.file:
      path: /etc/apt/keyrings
      state: directory
      mode: "0755"
      owner: root
      group: root
  - name: Download Docker's official GPG key
    ansible.builtin.get_url:
      url: https://download.docker.com/linux/ubuntu/gpg
      dest: /etc/apt/keyrings/docker.asc
      mode: "0644"
      owner: root
      group: root
  - name: Ensure Docker GPG key has public read permissions
    ansible.builtin.file:
      path: /etc/apt/keyrings/docker.asc
      state: file
      mode: "0644"
  - name: Get codename
    ansible.builtin.shell: |
      . /etc/os-release
      echo "$VERSION_CODENAME"
    args:
      executable: /bin/bash
    register: os_codename
    changed_when: false
  - name: Get dpkg architecture
    ansible.builtin.command: dpkg --print-architecture
    register: dpkg_architecture
    changed_when: false
  - name: Add Docker official deb822 repository source
    ansible.builtin.deb822_repository:
      name: docker
      types: deb
      uris: https://download.docker.com/linux/debian
      suites: "{{ os_codename.stdout }}"
      components: stable
      architectures: "{{ dpkg_architecture.stdout }}"
      signed_by: /etc/apt/keyrings/docker.asc
  - name: Install Docker
    ansible.builtin.apt:
      update_cache: true
      pkg:
        - docker-ce
        - docker-ce-cli
        - containerd.io
        - docker-buildx-plugin
        - docker-compose-plugin
```
#### generic/reboot.yaml
```yaml
---
  - name: Reboot Machine
    ansible.builtin.reboot:
```
#### docker-host/mount-external.yaml
```yaml
---
  - name: Install Parted
    ansible.builtin.apt:
      update_cache: true
      pkg:
        - parted
  - name: Create GPT partition table and partition
    community.general.parted:
      device: "{{ target_device }}"
      number: "{{ partition_number }}"
      state: present
      fs_type: "{{ fstype }}"
      label: gpt
  - name: Format partition as ext4 if not already formatted
    community.general.filesystem:
      fstype: "{{ fstype }}"
      dev: "{{ target_device }}{{ partition_number }}"
  - name: Get partition UUID
    ansible.builtin.command:
      cmd: blkid -s UUID -o value {{ target_device }}{{ partition_number }}
    register: partition_uuid
    changed_when: false
  - name: Mount up device by UUID
    ansible.posix.mount:
      path: "{{ mount_path }}"
      src: "UUID={{ partition_uuid.stdout }}"
      fstype: "{{ fstype }}"
      opts: defaults,noatime
      state: mounted
```
#### docker-host/copy-start-dockhand.yaml
```yaml
---
  - name: Transfer file with permissions
    ansible.builtin.copy:
      src: "{{ playbook_dir }}/../docker/dockhand/compose.yaml"
      dest: /mnt/docker-volumes/dockhand.yaml
      owner: root
      group: root
      mode: '0644'
```
#### wings/install-wings.yaml
```yaml
---
  - name: Create Pelican Wings directories
    ansible.builtin.file:
      path: "{{ item }}"
      state: directory
      mode: "0755"
    loop:
      - /etc/pelican
      - /var/run/wings
  - name: Download Pelican Wings binary
    ansible.builtin.get_url:
      url: "https://github.com/pelican-dev/wings/releases/latest/download/wings_linux_{{ 'amd64' if ansible_facts.architecture == 'x86_64' else 'arm64' }}"
      dest: /usr/local/bin/wings
      mode: "0755"
  - name: Create Wings systemd service
    ansible.builtin.copy:
      dest: /etc/systemd/system/wings.service
      mode: "0644"
      content: |
        [Unit]
        Description=Wings Daemon
        After=docker.service
        Requires=docker.service
        PartOf=docker.service

        [Service]
        User=root
        WorkingDirectory=/etc/pelican
        LimitNOFILE=4096
        PIDFile=/var/run/wings/daemon.pid
        ExecStart=/usr/local/bin/wings
        Restart=on-failure
        StartLimitInterval=180
        StartLimitBurst=30
        RestartSec=5s

        [Install]
        WantedBy=multi-user.target
  - name: Reload systemd daemon
    ansible.builtin.systemd:
      daemon_reload: true
  - name: Enable and start Wings
    ansible.builtin.systemd:
      name: wings
      enabled: true
      state: started
```
#### wings/mount-backups.yaml
```yaml
---
  - name: Install nfs-common
    ansible.builtin.apt:
      update_cache: true
      pkg:
        - nfs-common
  - name: Create backup mount point
    ansible.builtin.file:
      path: /mnt/backups
      state: directory
      owner: root
      group: root
      mode: "0755"
  - name: Check immutable flag
    ansible.builtin.command:
      cmd: lsattr -d /mnt/backups
    register: backups_attr
    changed_when: false
  - name: Make backup mount point immutable
    ansible.builtin.command:
      cmd: chattr +i /mnt/backups
    when: "'i' not in backups_attr.stdout"
  - name: Configure Pelican backup NFS mount
    ansible.posix.mount:
      path: /mnt/backups
      src: 10.0.10.10:/mnt/Store/GameSaves/PelicanPanelBackups
      fstype: nfs
      opts: rw,hard,nofail,_netdev,nfsvers=4.2
      state: mounted
```
# Services
## Docker
Most of my apps run off Docker. All the compose files are stored in a repository so I don't have to manage the files directly.
I moved from [Portainer](https://www.portainer.io/) to [Dockhand](https://dockhand.pro/) to manage all the containers. The Dockhand compose file creates a proxy network that lets me only expose the reverse proxy. This network comes with an IPv4 and IPv6 subnet, which I needed to allowlist in the reverse proxy to allow [Gatus](https://gatus.io/) to query its targets.
## Pelican
Previously I installed Pelican following [[Pelican Panel Installation]]. However, I realized I could run the Pelican web panel inside Docker, which saves a lot of work. I set it up with the recommended defaults inside the Docker compose stack. The container also needed to run as UID 82 for file permissions for the internal user (there doesn't seem to be a way to configure the user inside the image.
## Syncthing
Had to remove the old certificate and chown the data directory to the correct user.
## Media
Just chown everything to the correct user.
