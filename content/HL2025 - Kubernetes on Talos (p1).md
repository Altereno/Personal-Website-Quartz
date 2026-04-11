This is part 1 of [[Homelab Plans - Summer 2025]]
# Closing Notes
Summary of the modifications as the project progresses:
- This page is only to create the underlying VMs and the bare Kubernetes cluster on Talos.
- Networking and storage have been moved to [[HL2025 - Kubernetes on Talos (p2)]] since I felt that it was more appropriate to consolidate there.
# Introduction
I wanted to have the infrastructure and Kubernetes automated by using Terraform. This way I can easily spin up a cluster that should have everything I need to explore Kubernetes.
# Notes
This would technically be my first Terraform project that has some sort of substance(?), so I tried my best to organize everything. Below is my file structure:
```
.
├── modules
│   ├── cilium
│   │   ├── configs
│   │   │   └── cilium.yaml
│   │   └── main.tf
│   ├── longhorn
│   │   └── main.tf
│   ├── proxmox
│   │   ├── main.tf
│   │   └── variables.tf
│   └── talos
│       ├── configs
│       │   ├── control.tpl
│       │   └── worker.yaml
│       ├── main.tf
│       ├── outputs.tf
│       └── variables.tf
├── main.tf
├── terraform.tfvars
└── variables.tf
```
**Updated: I have removed the Cilium and Longhorn modules, as they don't necessarily pertain to the infrastructure, I'd rather have them all grouped up separately so everything is centralized**
This is the new file structure:
```
.
├── modules
│   ├── proxmox
│   │   ├── main.tf
│   │   └── variables.tf
│   └── talos
│       ├── configs
│       │   ├── control.tpl
│       │   └── worker.yaml
│       ├── main.tf
│       ├── outputs.tf
│       └── variables.tf
├── main.tf
├── terraform.tfvars
└── variables.tf
```

Ideally each module is designed to be separate, but I ran into some issues with dependent providers.
Here are some of the issues I ran into / some things I didn't like about my implementation:
- The modules don't have their own `terraform.tfvars` file, so I had to define the `variables.tf` in the modules, then copy it all into the root `variables.tf` file. I also had to set the variables in the root `main.tf`.
- ~~I wanted to split up modules such that each module would be somewhat self-contained. An example would be having the Cilium module write what was necessary to the Talos machine configurations and then run the installation through Helm. This didn't really work out since I bootstrap the Talos cluster and I don't really know the behavior of machine configuration patches after bootstrapping.~~
- ~~I had to manually set the [`inlineManifests`](https://www.talos.dev/v1.10/reference/configuration/v1alpha1/config/#Config.cluster.inlineManifests.) in the Talos control plane machine configuration files because:~~
	- ~~When setting up Longhorn, it requires a [privileged namespace](https://longhorn.io/docs/1.7.0/advanced-resources/os-distro-specific/talos-linux-support/#pod-security), which I could not get working by setting an exemption in the [Pod Security section](https://www.talos.dev/v1.10/kubernetes-guides/configuration/pod-security/#configuration) in the Talos machine configuration.~~
	- ~~The other way was to create the namespace using the Kubernetes provider. While not in my configuration files anymore, the Kubernetes provider depended on the Talos provider to export the kubeconfig. This was an issue with the Kubernetes provider and not the Helm provider, the Kubernetes provider would complain that the kubeconfig file did not exist while the Helm provider didn't.~~
- ~~Since bootstrapping the Talos cluster took some time, I added a [`null_resource`](https://registry.terraform.io/providers/hashicorp/null/latest/docs/resources/resource) with a [`local_exec`](https://developer.hashicorp.com/terraform/language/resources/provisioners/local-exec) to poll the Kubernetes API endpoint with `kubectl`. This was just to prevent Helm from erroring out when trying to install a chart when the Kubernetes endpoint was inaccessible.~~
**Looking back, I think that I should have just had the Proxmox and Talos providers. Setting up networking and storage should have been moved to their own separate process.**
^ I did exactly this a week later.
# Prerequisites
To access everything remotely, I am using a M2 MacBook Air.
Here is the list of binaries that I had to install.
Since [Homebrew](https://brew.sh/) works on MacOS and Linux, I will use this to grab everything.
List of packages to install:
- [terraform](https://developer.hashicorp.com/terraform/install)
- [kubernetes-cli](https://formulae.brew.sh/formula/kubernetes-cli#default)
# Project Root
The root of the Terraform project contained my `main.tf`, `variables.tf`, and `terraform.tfvars`
## main.tf
```hcl
terraform {
  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = ">= 0.78.2"
    }
    talos = {
      source  = "siderolabs/talos"
      version = ">= 0.8.1"
    }
    null = {
      source  = "hashicorp/null"
      version = ">= 3.2.4"
    }
    helm = {
      source  = "hashicorp/helm"
      version = ">= 3.0.2"
    }
  }
  required_version = ">= 1.12.2"
}

locals {
  kubeconfig_path  = "${path.root}/exported_configs/kubeconfig"
  talosconfig_path = "${path.root}/exported_configs/talosconfig"
}

provider "proxmox" {
  endpoint  = var.endpoint
  api_token = var.api_token
  ssh {
    agent    = true
    username = var.username
    password = var.password
  }
}

provider "helm" {
  kubernetes = {
    config_path = local.kubeconfig_path
  }
}

module "proxmox" {
  source = "./modules/proxmox"

  node_name          = var.node_name
  image_datastore_id = var.image_datastore_id
  vm_datastore_id    = var.vm_datastore_id

  talos_version   = var.talos_version
  talos_image_url = var.talos_image_url

  network_bridge_device = var.network_bridge_device
  ipv4_gateway          = var.ipv4_gateway
  dns_servers           = var.dns_servers

  control_nodes             = var.control_nodes
  control_nodes_cores       = var.control_nodes_cores
  control_nodes_ram_size    = var.control_nodes_ram_size
  control_nodes_disk_size   = var.control_nodes_disk_size
  control_nodes_ipv4_prefix = var.control_nodes_ipv4_prefix

  worker_nodes             = var.worker_nodes
  worker_nodes_cores       = var.worker_nodes_cores
  worker_nodes_ram_size    = var.worker_nodes_ram_size
  worker_nodes_disk_size   = var.worker_nodes_disk_size
  worker_nodes_ipv4_prefix = var.worker_nodes_ipv4_prefix
}

module "talos" {
  depends_on = [module.proxmox]

  source = "./modules/talos"

  cluster_name              = var.cluster_name
  control_nodes             = var.control_nodes
  control_nodes_ipv4_prefix = var.control_nodes_ipv4_prefix
  worker_nodes              = var.worker_nodes
  worker_nodes_ipv4_prefix  = var.worker_nodes_ipv4_prefix
}

module "cilium" {
  depends_on = [module.talos]

  source = "./modules/cilium"
}

module "longhorn" {
  depends_on = [module.cilium]

  source = "./modules/longhorn"
}
```
This defined and configured providers for all the modules, along with passing the variables.
## variables.tf
```hcl
variable "endpoint" {
  description = "Endpoint for Proxmox host"
  type        = string
}

variable "api_token" {
  description = "API token for proxmox user"
  type        = string
  sensitive   = true
}

variable "username" {
  description = "PVE User"
  type        = string
}

variable "password" {
  description = "PVE User password"
  type        = string
  sensitive   = true
}

variable "node_name" {
  description = "Proxmox node name"
  type        = string
}

variable "image_datastore_id" {
  description = "Image download datastore location"
  type        = string
}

variable "vm_datastore_id" {
  description = "VM image datastore location"
  type        = string
}

variable "talos_version" {
  description = "Version of Talos Linux being deploy, for naming purposes"
  type        = string
}

variable "talos_image_url" {
  description = "Download link of the image file, not iso of Talos Linux"
  type        = string
}

variable "network_bridge_device" {
  description = "Network bridge device for nodes"
  type        = string
}

variable "ipv4_gateway" {
  description = "Default gateway for nodes"
  type        = string
}

variable "dns_servers" {
  description = "DNS server for nodes"
  type        = list(string)
}

variable "control_nodes" {
  description = "Nummber of control nodes to create"
  type        = number
  default     = 0
}

variable "control_nodes_cores" {
  description = "Number of cores for each control node"
  type        = number
  default     = 2
}

variable "control_nodes_ram_size" {
  description = "Size in MB for control node RAM"
  type        = number
  default     = 2048
}

variable "control_nodes_disk_size" {
  description = "VM disk size"
  type        = number
  default     = 10
}

variable "control_nodes_ipv4_prefix" {
  description = "IPv4 prefix for control nodes"
  type        = string
}

variable "worker_nodes" {
  description = "Nummber of worker nodes to create"
  type        = number
  default     = 0
}

variable "worker_nodes_cores" {
  description = "Number of cores for each worker node"
  type        = number
  default     = 1
}

variable "worker_nodes_ram_size" {
  description = "Size in MB for worker node RAM"
  type        = number
  default     = 1024
}

variable "worker_nodes_disk_size" {
  description = "VM disk size"
  type        = number
  default     = 10
}

variable "worker_nodes_ipv4_prefix" {
  description = "IPv4 prefix for worker nodes"
  type        = string
}

variable "cluster_name" {
  description = "Cluster name"
  type        = string
}
```
This contained all the variables that were needed for the modules.
## terraform.tfvars
```hcl
endpoint = "https://proxmox:8006/"
api_token = "user@pam!token=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"
username = "user"
password = "abcd123"

node_name = "pve"
image_datastore_id = "local-zfs"
vm_datastore_id = "local-zfs"

talos_version = "1.10.4"
talos_image_url = "https://factory.talos.dev/image/ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515/v1.10.4/nocloud-amd64.raw.gz"

network_bridge_device = "vmbr0"
ipv4_gateway = "192.168.0.1"
dns_servers = ["1.1.1.1"]

control_nodes = 1
control_nodes_cores = 2
control_nodes_ram_size = 2048
control_nodes_disk_size = 10
control_nodes_ipv4_prefix = "192.168.0.64/26"

worker_nodes = 1
worker_nodes_cores = 1
worker_nodes_ram_size = 1024
worker_nodes_disk_size = 10
worker_nodes_ipv4_prefix = "192.168.0.128/26"

cluster_name = "test-cluster"
```
This file contains all the variables values. I made sure that this file was not included when using my source control. I just used the [example `.gitignore` file](https://github.com/github/gitignore/blob/main/Terraform.gitignore) from GitHub.
# Proxmox
## Creating the Terraform user on Proxmox
Information taken from:
- [API Token Authentication](https://github.com/bpg/terraform-provider-proxmox/blob/main/docs/index.md#api-token-authentication)
- [SSH User](https://github.com/bpg/terraform-provider-proxmox/blob/main/docs/index.md#ssh-user)
Since some modules require the use of SSH to the Proxmox host, we are not able to rely solely on the API.

*To obtain a user for SSH, we need to actually run the commands in the shell. The PAM authentication is used for local nodes, while the PVE authentication is synced across all nodes in a cluster. See [here](- [User Management](https://pve.proxmox.com/wiki/User_Management#Administrator_Group) for extra info.*

In the Proxmox shell:
`adduser -m tfuser` to create the user along with the home directory
`visudo -f /etc/sudoers.d/tfuser` to create the sudoers file
Add the following to the file:
```
tfuser ALL=(root) NOPASSWD: /sbin/pvesm
tfuser ALL=(root) NOPASSWD: /sbin/qm
tfuser ALL=(root) NOPASSWD: /usr/bin/tee /var/lib/vz/*
```
If there is other datastores, add them with this:
```
terraform ALL=(root) NOPASSWD: /usr/bin/tee /mnt/pve/<other>
```

In the Proxmox GUI:
- Datacenter -> Permissions -> Users -> Add:
	- Create a new user `tfuser`
	- Select `tfuser` and change the password
	- *Note down the password for later*
- Datacenter -> Permissions -> Roles -> Create
	- Create a new role `tfroles`
	- Add these roles:
		- SDN.Use
		- Sys.Audit
		- Sys.Modify
		- Sys.AccessNetwork
		- VM.Allocate
		- VM.Audit
		- VM.Backup
		- VM.Clone
		- VM.Config.CDROM
		- VM.Config.CPU
		- VM.Config.Cloudinit
		- VM.Config.Disk
		- VM.Config.HWType
		- VM.Config.Memory
		- VM.Config.Network
		- VM.Config.Options
		- VM.Migrate
		- VM.Monitor
		- VM.PowerMgmt
		- VM.Snapshot
		- VM.Snapshot
		- Datastore.Allocate
		- Datastore.AllocateSpace
		- Datastore.AllocateTemplate
		- Datastore.Audit
	- *Proxmox docs on [privileges](https://pve.proxmox.com/wiki/User_Management)*
- Datacenter -> Permissions -> API Tokens -> Add
	- Select the `tfuser` user
	- Specify `tftoken` in the Token ID field
	- *Note down the Token ID and the Secret once it has been created*
- Datacenter -> Permissions -> Add -> User Permission
	- Choose the root path `/`
	- Choose the `tfuser` user
	- Choose the `tfroles` role
	- Check the propagate box
- Datacenter -> Permissions -> Add -> API Token Permission
	- Choose the root path `/`
	- Choose the `tfuser` user
	- Choose the `tfroles` role
	- Check the propagate box

## Configuration
### main.tf
```hcl
terraform {
  required_providers {
    proxmox = {
      source = "bpg/proxmox"
    }
  }
}

resource "proxmox_virtual_environment_download_file" "talos_linux_image" {
  node_name               = var.node_name
  content_type            = "iso"
  datastore_id            = var.image_datastore_id
  file_name               = "talos-${var.talos_version}-nocloud-amd64.iso"
  url                     = var.talos_image_url
  decompression_algorithm = "gz"
  overwrite               = false
}

resource "proxmox_virtual_environment_vm" "worker-nodes" {
  count       = var.worker_nodes
  name        = "worker-node-${format("%02d", count.index)}"
  description = "Managed by Terraform"
  tags        = ["terraform", "worker"]
  node_name   = var.node_name

  operating_system {
    type = "l26"
  }

  agent {
    enabled = true
  }

  cpu {
    cores = var.worker_nodes_cores
    type  = "x86-64-v2-AES"
  }

  memory {
    dedicated = var.worker_nodes_ram_size
  }

  disk {
    datastore_id = var.vm_datastore_id
    discard      = "on"
    iothread     = true
    file_format  = "raw"
    file_id      = proxmox_virtual_environment_download_file.talos_linux_image.id
    interface    = "virtio0"
    size         = var.worker_nodes_disk_size
  }

  network_device {
    bridge = var.network_bridge_device
  }

  initialization {
    datastore_id = var.vm_datastore_id
    dns {
      servers = var.dns_servers
    }
    ip_config {
      ipv4 {
        gateway = var.ipv4_gateway
        address = "${cidrhost(var.worker_nodes_ipv4_prefix, count.index)}/24"
      }
    }

  }
}

resource "proxmox_virtual_environment_vm" "control-nodes" {
  count       = var.control_nodes
  name        = "control-node-${format("%02d", count.index)}"
  description = "Managed by Terraform"
  tags        = ["terraform", "control"]
  node_name   = var.node_name

  operating_system {
    type = "l26"
  }

  agent {
    enabled = true
  }

  cpu {
    cores = var.control_nodes_cores
    type  = "x86-64-v2-AES"
  }

  memory {
    dedicated = var.control_nodes_ram_size
  }

  disk {
    datastore_id = var.vm_datastore_id
    discard      = "on"
    iothread     = true
    file_format  = "raw"
    file_id      = proxmox_virtual_environment_download_file.talos_linux_image.id
    interface    = "virtio0"
    size         = var.control_nodes_disk_size
  }

  network_device {
    bridge = var.network_bridge_device
  }

  initialization {
    datastore_id = var.vm_datastore_id
    dns {
      servers = var.dns_servers
    }
    ip_config {
      ipv4 {
        gateway = var.ipv4_gateway
        address = "${cidrhost(var.control_nodes_ipv4_prefix, count.index)}/24"
      }
    }

  }
}
```
This defined the virtual machine settings for the worker and control plane nodes. It also includes the image resource to automatically pull the correct image to use.
### variables.tf
```hcl
variable "node_name" {
  description = "Proxmox node name"
  type        = string
}

variable "image_datastore_id" {
  description = "Image download datastore location"
  type        = string
}

variable "vm_datastore_id" {
  description = "VM image datastore location"
  type        = string
}

variable "talos_version" {
  description = "Version of Talos Linux being deploy, for naming purposes"
  type        = string
}

variable "talos_image_url" {
  description = "Download link of the image file, not iso of Talos Linux"
  type        = string
}

variable "network_bridge_device" {
  description = "Network bridge device for nodes"
  type        = string
}

variable "ipv4_gateway" {
  description = "Default gateway for nodes"
  type        = string
}

variable "dns_servers" {
  description = "DNS server for nodes"
  type        = list(string)
}

variable "control_nodes" {
  description = "Nummber of control nodes to create"
  type        = number
  default     = 0
}

variable "control_nodes_cores" {
  description = "Number of cores for each control node"
  type        = number
  default     = 2
}

variable "control_nodes_ram_size" {
  description = "Size in MB for control node RAM"
  type        = number
  default     = 2048
}

variable "control_nodes_disk_size" {
  description = "VM disk size"
  type        = number
  default     = 10
}

variable "control_nodes_ipv4_prefix" {
  description = "IPv4 prefix for control nodes"
  type        = string
}

variable "worker_nodes" {
  description = "Nummber of worker nodes to create"
  type        = number
  default     = 0
}

variable "worker_nodes_cores" {
  description = "Number of cores for each worker node"
  type        = number
  default     = 1
}

variable "worker_nodes_ram_size" {
  description = "Size in MB for worker node RAM"
  type        = number
  default     = 1024
}

variable "worker_nodes_disk_size" {
  description = "VM disk size"
  type        = number
  default     = 10
}

variable "worker_nodes_ipv4_prefix" {
  description = "IPv4 prefix for worker nodes"
  type        = string
}
```
Defines all the variables that the resources are going to use.
## Reference
There are 3 main resources that are being created: the worker nodes, the control nodes, and the image itself. I configured the nodes with the [minimum requirements specified](https://www.talos.dev/v1.10/introduction/system-requirements/).
[Virtual machine docs](https://github.com/bpg/terraform-provider-proxmox/blob/main/docs/resources/virtual_environment_vm.md)
[Download file docs](https://github.com/bpg/terraform-provider-proxmox/blob/main/docs/resources/virtual_environment_download_file.md)

*Naming conventions for the VMs only go up to two digits*
*I use the count variable to "calculate" the IPv4 addresses, it currently is configured as `/24` subnet but because of the two digits, it can't fill up the subnet.*
### Talos Linux Image
To get the appropriate Talos image for Proxmox, use the [Talos Linux Image Factory](https://factory.talos.dev/)
Options selected:
- Cloud Server
- Latest Talos Linux version (*currently 1.10.4*)
- Nocloud
- amd64, SecureBoot off
- Extension: `siderolabs/qemu-guest-agent`
- Skip customization
**Get the link but change the extension from zx to gz since Proxmox doesn't support decompression with xz when downloading files**
# Talos Linux
## Configuration
### main.tf
```hcl
terraform {
  required_providers {
    talos = {
      source = "siderolabs/talos"
    }
    null = {
      source = "hashicorp/null"
    }
  }
}

locals {
  worker_node_ips  = [for i in range(var.worker_nodes) : "${cidrhost(var.worker_nodes_ipv4_prefix, i)}"]
  control_node_ips = [for i in range(var.control_nodes) : "${cidrhost(var.control_nodes_ipv4_prefix, i)}"]
  vip_ip           = cidrhost(var.worker_nodes_ipv4_prefix, var.control_nodes)
}

resource "talos_machine_secrets" "this" {}

data "talos_machine_configuration" "worker" {
  cluster_name     = var.cluster_name
  machine_type     = "worker"
  cluster_endpoint = "https://${cidrhost(var.worker_nodes_ipv4_prefix, var.control_nodes)}:6443"
  machine_secrets  = talos_machine_secrets.this.machine_secrets
}

data "talos_machine_configuration" "controlplane" {
  cluster_name     = var.cluster_name
  machine_type     = "controlplane"
  cluster_endpoint = "https://${cidrhost(var.worker_nodes_ipv4_prefix, var.control_nodes)}:6443"
  machine_secrets  = talos_machine_secrets.this.machine_secrets
}

data "talos_client_configuration" "this" {
  cluster_name         = var.cluster_name
  client_configuration = talos_machine_secrets.this.client_configuration
  nodes                = concat(local.control_node_ips, local.worker_node_ips)
}

resource "talos_machine_configuration_apply" "worker" {
  for_each = toset(local.worker_node_ips)

  client_configuration        = talos_machine_secrets.this.client_configuration
  machine_configuration_input = data.talos_machine_configuration.worker.machine_configuration
  node                        = each.value
  config_patches              = [file("${path.module}/configs/worker.yaml")]
}

resource "talos_machine_configuration_apply" "controlplane" {
  for_each = toset(local.control_node_ips)

  client_configuration        = talos_machine_secrets.this.client_configuration
  machine_configuration_input = data.talos_machine_configuration.controlplane.machine_configuration
  node                        = each.value
  config_patches              = [templatefile("${path.module}/configs/control.tpl", { vip_ip = local.vip_ip })]
}

resource "talos_machine_bootstrap" "this" {
  depends_on           = [talos_machine_configuration_apply.controlplane, talos_machine_configuration_apply.worker]
  node                 = cidrhost(var.control_nodes_ipv4_prefix, 0)
  client_configuration = talos_machine_secrets.this.client_configuration
}

resource "talos_cluster_kubeconfig" "this" {
  depends_on           = [talos_machine_bootstrap.this]
  node                 = cidrhost(var.control_nodes_ipv4_prefix, 0)
  client_configuration = talos_machine_secrets.this.client_configuration
}

resource "local_sensitive_file" "kubeconfig" {
  depends_on = [talos_cluster_kubeconfig.this]
  content  = resource.talos_cluster_kubeconfig.this.kubeconfig_raw
  filename = "${path.root}/exported_configs/kubeconfig"
}

resource "local_sensitive_file" "talosconfig" {
  depends_on = [data.talos_client_configuration.this]
  content  = data.talos_client_configuration.this.talos_config
  filename = "${path.root}/exported_configs/talosconfig"
}

resource "null_resource" "wait_for_kubernetes" {
  depends_on = [local_sensitive_file.kubeconfig]
  
  provisioner "local-exec" {
    working_dir = "${path.root}/exported_configs"
    command = <<EOT
      for i in {1..60}; do
        kubectl --kubeconfig="kubeconfig" get nodes && break
        echo "Waiting for Kubernetes API..."
        sleep 5
      done
    EOT
  }
}
```
`talos_machine_secrets` generates the one time secrets for the cluster.  Once the secrets are generated, I had a separate worker and control loop for the `talos_machine_configuration`, which applies the configurations for two types of nodes. After applying the configurations, I can bootstrap the cluster using the first control plane node's IP. I also write the kubeconfig for the Helm provider later.
### variables.tf
```hcl
variable "cluster_name" {
  description = "Cluster name"
  type        = string
}

variable "worker_nodes" {
  description = "Nummber of worker nodes to create"
  type        = number
  default     = 0
}

variable "worker_nodes_ipv4_prefix" {
  description = "IPv4 prefix for worker nodes"
  type        = string
}

variable "control_nodes" {
  description = "Nummber of control nodes to create"
  type        = number
  default     = 0
}

variable "control_nodes_ipv4_prefix" {
  description = "IPv4 prefix for control nodes"
  type        = string
}
```
This just takes in the variables from the root `variables.tf` file.
### outputs.tf
```hcl
output "talosconfig" {
  value     = data.talos_client_configuration.this.talos_config
  sensitive = true
}

output "kubeconfig" {
  value     = resource.talos_cluster_kubeconfig.this.kubeconfig_raw
  sensitive = true
}
```
This exports the Talos configuration file along with the kubeconfig.
To retrieve the outputs we can run:
```sh
terraform output -raw talosconfig > talosconfig
terraform output -raw kubeconfig > kubeconfig
```
### control.tpl
```yaml
machine:
  kubelet:
    extraMounts:
      - destination: /var/lib/longhorn
        type: bind
        source: /var/lib/longhorn
        options:
          - bind
          - rshared
          - rw
  install:
    disk: /dev/vda
  network:
    interfaces:
      - interface: eth0
        vip:
          ip: ${vip_ip}
cluster:
  network:
    cni:
      name: none
  proxy:
    disabled: true
  # inlineManifests:
  #   - name: namespace-longhorn-system
  #     contents: |-
  #       apiVersion: v1
  #       kind: Namespace
  #       metadata:
  #         name: longhorn-system
  #         labels:
  #           pod-security.kubernetes.io/enforce: privileged

```
This is the machine configuration for the control plane nodes. This uses the `.tpl` file extension to allow for [templating with Terraform](https://developer.hashicorp.com/terraform/language/functions/templatefile). 

For the modifications:
- Since the VM uses `virtio` for the disk, I needed to change the install disk to `/dev/vda`. If there was another type I could have checked by doing: `talosctl -n <node> get disks --insecure`.
- The same method was applied to find the correct network adapter: `talosctl -n <node> get addresses --insecure`, this was required to set up the [Layer 2 VIP Shared IP](https://www.talos.dev/v1.10/introduction/prodnotes/#layer-2-vip-shared-ip).
- The default CNI that ships with Talos is [Flannel](https://github.com/flannel-io/flannel). It also comes with `kube-proxy` enabled. Since I wanted to replace them with [Cilium](https://cilium.io/), I set the `cni` to `none` and the `proxy` to `false`.
- ~~I needed to add the mounts for [Longhorn](https://longhorn.io/) along with the [inlineManifest](https://www.talos.dev/v1.10/reference/configuration/v1alpha1/config/#Config.cluster.inlineManifests.) to create the privileged namespace required for it.~~
### worker.yaml
```yaml
machine:
  kubelet:
    extraMounts:
      - destination: /var/lib/longhorn
        type: bind
        source: /var/lib/longhorn
        options:
          - bind
          - rshared
          - rw
  install:
    disk: /dev/vda
cluster:
  network:
    cni:
      name: none
  proxy:
    disabled: true
```
This is the machine configuration for the worker nodes. This is the same as the control plane but just removing anything that is related to the control plane.
## Reference
For Talos configuration, I just needed to apply the machine configurations for both the control plane nodes and the worker nodes.
[Talos Provider Docs](https://github.com/siderolabs/terraform-provider-talos/tree/main/docs)
[Cilium CNI on Talos](https://www.talos.dev/v1.10/kubernetes-guides/network/deploying-cilium/)
[Longhorn CSI on Talos](https://longhorn.io/docs/1.9.0/advanced-resources/os-distro-specific/talos-linux-support/)
# ~~Cilium~~
## ~~Configuration~~
### ~~main.tf~~
```hcl
terraform {
  required_providers {
    helm = {
      source = "hashicorp/helm"
    }
  }
}

resource "helm_release" "cilium" {
  name       = "cilium-cni"
  repository = "https://helm.cilium.io/"
  chart      = "cilium"
  version    = "1.17.5"
  namespace  = "kube-system"

  values = [file("${path.module}/configs/cilium.yaml")]
}
```
~~This just installs the Cilium chart using Helm. I also specified a configuration file here.~~
### ~~cilium.yaml~~
```yaml
ipam:
  mode: kubernetes
kubeProxyReplacement: true
securityContext:
  capabilities:
    ciliumAgent:
      - CHOWN
      - KILL
      - NET_ADMIN
      - NET_RAW
      - IPC_LOCK
      - SYS_ADMIN
      - SYS_RESOURCE
      - DAC_OVERRIDE
      - FOWNER
      - SETGID
      - SETUID
    cleanCiliumState:
      - NET_ADMIN
      - SYS_ADMIN
      - SYS_RESOURCE
cgroup:
  autoMount:
    enabled: false
  hostRoot: /sys/fs/cgroup
k8sServiceHost: localhost
k8sServicePort: 7445
gatewayAPI:
  enabled: true
  enableAlpn: true
  enableAppProtocol: true
```
~~These are just the values I obtained from the [Talos Docs](https://www.talos.dev/v1.10/kubernetes-guides/network/deploying-cilium/#method-1-helm-install). This is for the Cilium deployment without `kube-proxy` that also has `GatewayAPI` support.~~
# ~~Longhorn~~
## ~~Configuration~~
### ~~main.tf~~
```hcl
terraform {
  required_providers {
    helm = {
      source = "hashicorp/helm"
    }
  }
}

resource "helm_release" "longhorn" {
  name             = "longhorn-csi"
  repository       = "https://charts.longhorn.io"
  chart            = "longhorn"
  version          = "1.8.2"
  namespace        = "longhorn-system"
  create_namespace = true
}
```
~~This just installs the Longhorn chart using Helm. ~~
# Execution
Run `terraform init` in the root directory to initialize Terraform
Run `terraform validate` to make sure there aren't any syntax or type issues
Run `terraform plan` to see the proposed state modifications
If everything looks good, run `terraform apply` to apply the changes

If everything runs smoothly, the cluster should now be reachable, I checked using
```sh
export KUBECONFIG=exported_configs/kubeconfig
kubectl get nodes
```

~~This should give me a cluster to play around with for the CI/CD pipeline.~~
I now need to setup the CNI and CSI for the cluster.