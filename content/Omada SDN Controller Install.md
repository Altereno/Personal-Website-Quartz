# Background
My previous installation of the [Omada](https://www.omadanetworks.com/) SDN controller was on an LXC based off of Ubuntu 20.04. That version has reached EOL since May 31, 2025.

Omada documentation sucks so bad (as of April 11, 2026), I'm not really sure how I got it to run in the first place. There is a way to run this in Docker, but I'm using LXC on Proxmox to assign a physical interface that is connected to the management network. I'm writing this to document what I did in case I need this later.

I will be using an LXC image based off Ubuntu 24.04.

# Prerequisites
Make sure to export the configuration before nuking anything.

The Omada website states that I will need `openjdk`, `jsvc`, and `mongodb`. However, they failed to mention that `openjdk` versions 8 and 11 are not supported with the current release of the controller (6.2.0.17). They also linked an older version of `jsvc` and `mongodb`.

Here is a quick script to install all the required packages:
```bash
# Install requirements for mongodb, along with Apache Commons (jsvc included) and openjdk-17
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y gnupg curl libcommons-daemon-java openjdk-17-jre-headless

# Add mongodb repository
curl -fsSL https://pgp.mongodb.com/server-8.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg \
   --dearmor   
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list

# Install mongodb
sudo apt-get update
sudo apt-get install -y mongodb-org
```

# Resources
- [MongoDB](https://www.mongodb.com/docs/v8.0/tutorial/install-mongodb-on-ubuntu/)
- [Omada SDN Controller Download](https://support.omadanetworks.com/us/product/omada-software-controller/?resourceType=download) (This link might be dead later...)

# Note
When `dpkg` failed, I had to run this command to unlock `apt`: 

`dpkg --remove --force-remove-reinstreq omadac`