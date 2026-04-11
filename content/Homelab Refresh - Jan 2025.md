# Goals
- Create virtual machines with a defined purpose, instead of one virtual machine for all docker services. This isolates problems when a container or virtual machine becomes unresponsive.
- Move all internal services from using ISCSI to NFS
- Move Jellyfin to a docker container, I noticed that I only direct play media, I'm going to replace the Jellyfin LXC container with the GPU permissions with a docker container.
- Move all documentation to Obsidian, replacing BookStack
## Creating multiple virtual machines
Previously, I had 2 virtual machines for Docker containers: one for internal and one for external services. The main reason why I split them up is to prevent a single container from locking up all the other services. 

My internal reverse proxy was the main reason why this change was made. When I restarted my internal docker host, it would bring down all my services that were defined with a FQDN. While I could have moved back to using the HAProxy package on my PfSense box, I already had NGINX Proxy Manager with all my services defined.

I split the one host into 4 separate hosts:
1. Reverse Proxy (NGINX Reverse Proxy + Portainer)
2. Media (Jellyfin + Shoko + QBittorrent)
3. Files  (Syncthing + Bitwarden)
4. Misc (Uptime Kuma + Homepage)

I opted not to use the Terraform provider for Proxmox this time. I manually created 4 virtual machines through the GUI. I was just lazy and I didn't want to deal with state management of the virtual machines in Terraform.

![Automation](https://imgs.xkcd.com/comics/automation_2x.png)

To manage all the different hosts, I used Ansible, with the control node being my MacBook. I used Ansible here just to keep package updates easy.

I created a key that I enrolled into each virtual machine, then I populated my inventory along with the password obtained using [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/vault_encrypting_content.html)

```yaml
new-hosts:
	vars:
		ansible_ssh_private_key_file: ./keys/ansible_ed25519
	hosts:
		reverse-proxy:
		ansible_host: 10.0.10.150
		ansible_user: steven
		ansible_become_pass: !vault |
			$ANSIBLE_VAULT;1.1;AES256
			123
	media:
		ansible_host: 10.0.10.151
		ansible_user: steven
		ansible_become_pass: !vault |
			$ANSIBLE_VAULT;1.1;AES256
			123
	file-sync:
		ansible_host: 10.0.10.152
		ansible_user: steven
		ansible_become_pass: !vault |
			$ANSIBLE_VAULT;1.1;AES256
			123
	misc:
		ansible_host: 10.0.10.153
		ansible_user: steven
		ansible_become_pass: !vault |
			$ANSIBLE_VAULT;1.1;AES256
			123

```

I tested a ping to see if all the hosts were reachable:

```yaml
- name: Test connectivity to hosts
  hosts: all
  tasks:
    - name: Ping Hosts
      ansible.builtin.ping:

```

Then I went ahead and updated the repository and packages:

```yaml
- name: Update packages
  hosts: all
  become: true
  tasks:
    - name: Update APT package cache
      ansible.builtin.apt:
        update_cache: true
    - name: Upgrade packages
      ansible.builtin.apt:
        upgrade: safe

```

And then installing 2 things, the QEMU guest agent and Docker

```yaml
- name: Install qemu guest agent
  hosts: all
  become: true
  tasks:
    - name: Update APT package cache
      ansible.builtin.apt:
        update_cache: true
    - name: Install package
      ansible.builtin.package:
        name: qemu-guest-agent
        state: present
    - name: Enable and start qemu agent
      ansible.builtin.service:
        name: qemu-guest-agent
        enabled: true
        state: started

```

```yaml
- name: Install Docker
  hosts: "{{ hosts }}"
  become: true
  tasks:
    - name: Update APT package cache
      ansible.builtin.apt:
        update_cache: true

    - name: Install package
      ansible.builtin.package:
        name:
          - ca-certificates
          - curl
        state: present

    - name: Create the keyrings directory for Docker
      ansible.builtin.file:
        path: /etc/apt/keyrings
        state: directory
        mode: '0755'

    - name: Download Docker's GPG key
      ansible.builtin.get_url:
        url: https://download.docker.com/linux/ubuntu/gpg
        dest: /etc/apt/keyrings/docker.asc
        mode: '0644'

    - name: Add Docker repository to sources list
      ansible.builtin.shell:
        cmd: |
          echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
          $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" > /etc/apt/sources.list.d/docker.list
        creates: /etc/apt/sources.list.d/docker.list

    - name: Update APT cache after adding Docker repository
      ansible.builtin.apt:
        update_cache: true

    - name: Install package
      ansible.builtin.package:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-buildx-plugin
          - docker-compose-plugin
        state: present

```

After that, I ran Portainer on the reverse proxy host, and added the other hosts using the [Portainer Agents](https://docs.portainer.io/admin/environments/add/docker/agent). Using compose files (stacks on Portainer), I launched all my services and had them up pretty quickly.

### Storage for persistent docker data
I previously used an iSCSI LUN mapped to each docker host to store all my persistent data, while that was actually not an issue and generally worked well, because I split up my hosts, I wanted to give NFS another shot.

>Side note: Throughout the fall semester, I had been getting multiple emails from my TrueNAS host about file corruption, since the pool status gave me only checksum errors, I had thought that my connections to my hard drives or my HBA had gone bad. After adjusting them and continuing to get errors, I ran a [Memtest](https://www.memtest.org/) and realized that one pair of RAM sticks were broken. I went from 32GB to 16GB of RAM but I didn't bother replacing them since I didn't notice any performance degradation.  Due to this, one of my Zvols were corrupted, so I just removed it and used NFS. 

### Permissions
#### TrueNAS Dataset ACLs
Information referenced from the [ACL Primer from iXSystems](https://www.truenas.com/docs/references/aclprimer/#acl-overview). 
I decided with the NFSv4 type ACLs since I do have windows workstations and I wanted the support for the finer grain permissions. I believe that the FreeBSD based TrueNAS Core was also using it for permissions, so I was just more familiar.

### Mounting volumes for docker
Previously, I had the docker host mount a CIFS share at boot using fstab and then using local bindmounts to give containers access to the media. I wanted to change this to having the NFS mount inside the container. While I could write an Ansible playbook to setup the local mounts, I'd rather have all the container information in one place, which is inside the compose file. 

With the ACLs on TrueNAS set up, I noted down the UID of each user and mapped the UID to match in the containers. I then launched an interactive shell into each container and verified the permissions that I had set.

*The packages for NFS `nfs-common` and CIFS `cifs-utils` need to be installed on the VM or the Docker volume mounts will not work. I probably should have added these to the list of packages that Ansible installed intially.*

### Quirks with persistent data
While I could set up a SQL server for all the different applications, I thought it was easier to have each container just use their own SQLite instance. The main issue I had with a SQLite DB on a network share is the occasional errors in the DB due to locking. This wasn't actually a problem when I had the persistent data stored on the iSCSI device since the allocation of a LUN to a VM would grant only that VM access to the block level device. 

I made the change to just move all persistent data to the local VM and have Proxmox take weekly snapshots and send it to TrueNAS. This decouples my storage server to my compute. Some examples as to why this was better for me:
- If I had to do maintenance on my storage server
- When NUT shutdowns my servers I don't have to worry about the storage server shutting down before my compute
- Applications that required indexing of a large quantity of smaller files such as Shoko and Jellyfin was noticeably faster 

~~I also noticed that while the Syncthing user had complete access through ACLS to the NFS share I mounted, it won't sync unless the Syncthing user is the owner of the files. While I could change the ACLs on TrueNAS to have Syncthing own it and give full permissions to me, I just did a bandaid fix by using CIFS with the noperm flag to bypass all permission checks. (lol)~~
I found out that just had the wrong user and group ID set, the permissions were correct.
