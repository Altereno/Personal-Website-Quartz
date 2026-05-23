# Background
Here is the progression of the game panels I've used:
- MineOS (Unable to find link D:)
- [PufferPanel](https://pufferpanel.com/)
- [Pterodactyl Panel](https://pterodactyl.io/) (Current)

I wanted to migrate over to [Pelican Panel](https://pelican.dev/) for fun.

# Installation
I am running Ubuntu 24.04 on a virtual machine.

## Getting Started
[Source](https://pelican.dev/docs/panel/getting-started)

### Dependencies
Recommended version for `PHP` is 8.5, that is what I will install:
```bash
sudo apt install PHP8.5-gd PHP8.5-mysql PHP8.5-mbstring PHP8.5-bcmath PHP8.5-xml PHP8.5-curl PHP8.5-zip PHP8.5-intl PHP8.5-sqlite3 PHP8.5-fpm
```

*This will install `Apache2`, but I am using Nginx, so I just removed it:*
```bash
sudo systemctl stop apache2
sudo apt remove apache2*
sudo apt-get autoremove
```
I believe there is a way to install only what is listed, I think I just needed this flag: `--no-install-recommends`

Make the directory and clone:
```sh
sudo mkdir -p /var/www/pelican
cd /var/www/pelican
curl -L https://github.com/pelican-dev/panel/releases/latest/download/panel.tar.gz | sudo tar -xzv
```

Install Composer and run:
```bash
curl -sS https://getcomposer.org/installer | sudo php -- --install-dir=/usr/local/bin --filename=composer
sudo COMPOSER_ALLOW_SUPERUSER=1 composer install --no-dev --optimize-autoloader
```

## Webserver Configuration
[Source](https://pelican.dev/docs/panel/webserver-config)

Before setting up the webserver, I needed to get certificates for them.

### Certificates
[Source](https://pelican.dev/docs/guides/ssl/)
Previously on Pterodactyl, I had certificates set up with my Nginx Reverse Proxy. That was a little annoying since I had to create a post renewal script with Let's Encrypt to update the Wings certificate.

For Pelican, the plan is to have it all self-contained.

To have certificates auto renew, I will use [Certbot](https://certbot.eff.org/). Since I am not going to expose this panel publicly, I have to use the DNS challenge option. My current DNS provider is Cloudflare.

To install Certbot and the [Cloudflare plugin](https://certbot-dns-cloudflare.readthedocs.io/en/stable/):
```bash
sudo snap install --classic certbot
sudo snap install certbot-dns-cloudflare
```

On the Cloudflare side, I needed to create an API token with the `Zone:DNS:Edit` permission. I saved the token in `/root/.tokens/cloudflare-token.ini` and set the permissions for it.

```ini
# Cloudflare API token used by Certbot
dns_cloudflare_api_token = 0123456789abcdef0123456789abcdef01234567
```
```bash
chmod 600 /root/.tokens/cloudflare-token.ini
```

I needed to create two certificates, one for the panel and one for the node.
```bash
certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.tokens/cloudflare-token.ini \
  -d pelican.stevenchen.one
certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.tokens/cloudflare-token.ini \
  -d node0.pelican.stevenchen.one
```

When Certbot renews the panel certificate, I also wanted it to reload my Nginx worker by adding this line into `/etc/letsencrypt/renewal/pelican.stevenchen.one.conf`:

```conf
[renewalparams]
.
.
Other Stuff
.
.
renew_hook = systemctl reload nginx
```

### Nginx
[Source](https://nginx.org/en/linux_packages.html#Ubuntu)

To install Nginx:
```bash
sudo apt install curl gnupg2 ca-certificates lsb-release ubuntu-keyring
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
    | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
https://nginx.org/packages/ubuntu `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list
echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
    | sudo tee /etc/apt/preferences.d/99nginx
sudo apt update
sudo apt install nginx
```

This installation of Nginx does not have `sites-available` and `sites-enabled`. It instead stores all its site configuration under `/etc/nginx/conf.d/`

Remove the default site and create the pelican site:
```bash
rm /etc/nginx/conf.d/default.conf
touch /etc/nginx/conf.d/pelican.conf
```

Configure this before copying to `pelican.conf`:
- Replace `<domain>` with the panel domain, in my case: `pelican.stevenchen.one`
- Check that `PHP8.5-fpm.sock` exists under `/run/PHP/`, otherwise change to match the socket 
```nginx
server_tokens off;

server {
    listen 80;
    server_name <domain>;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    http2 on;
    server_name <domain>;

    root /var/www/pelican/public;
    index index.PHP;

    access_log /var/log/nginx/pelican.app-access.log;
    error_log  /var/log/nginx/pelican.app-error.log error;

    # allow larger file uploads and longer script runtimes
    client_max_body_size 100m;
    client_body_timeout 120s;

    sendfile off;

    ssl_certificate /etc/letsencrypt/live/<domain>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<domain>/privkey.pem;
    ssl_session_cache shared:SSL:10m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384";
    ssl_prefer_server_ciphers on;

    # See https://hstspreload.org/ before uncommenting the line below.
    # add_header Strict-Transport-Security "max-age=15768000; preload;";
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Robots-Tag none;
    add_header Content-Security-Policy "frame-ancestors 'self'";
    add_header X-Frame-Options DENY;
    add_header Referrer-Policy same-origin;

    location / {
        try_files $uri $uri/ /index.PHP?$query_string;
    }

    location ~ \.PHP$ {
        fastcgi_split_path_info ^(.+\.PHP)(/.+)$;
        fastcgi_pass unix:/run/PHP/PHP8.5-fpm.sock;
        fastcgi_index index.PHP;
        include fastcgi_params;
        fastcgi_param PHP_VALUE "upload_max_filesize = 100M \n post_max_size=100M";
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param HTTP_PROXY "";
        fastcgi_intercept_errors off;
        fastcgi_buffer_size 16k;
        fastcgi_buffers 4 16k;
        fastcgi_connect_timeout 300;
        fastcgi_send_timeout 300;
        fastcgi_read_timeout 300;
        include /etc/nginx/fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

Another change needed is to change the user that Nginx runs as, inside `/etc/nginx/nginx.conf` change `user nginx;` to `user www-data;`.

Restart Nginx:
```bash
sudo systemctl restart nginx
```

## Panel Setup
[Source](https://pelican.dev/docs/panel/panel-setup)

Create the `.env`
```bash
cp .env.example .env
```

Update the URL of the panel inside the `.env` file:
```
APP_URL=https://pelican.stevenchen.one
```
Also save the APP_KEY somewhere.

Set up the environment: 
```bash
sudo php artisan p:environment:setup
```



Set permissions for the panel and webserver:
```bash
sudo chmod -R 755 storage/* bootstrap/cache/
sudo chown -R www-data:www-data /var/www/pelican
```

Create the SQLite DB and migrate:
```bash
sudo touch /var/www/pelican/database/database.sqlite
sudo chown www-data:www-data /var/www/pelican/database/database.sqlite
php artisan p:environment:database
sudo -u www-data php artisan migrate --seed --force
```

Create the queue service:
```bash
sudo php /var/www/pelican/artisan p:environment:queue-service --overwrite
```

Add the cron job for schedules using `crontab -u www-data -e`:
```
* * * * * PHP /var/www/pelican/artisan schedule:run >> /dev/null 2>&1
```

# Wings
[Source](https://pelican.dev/docs/wings/install)

Docker install script:
```bash
curl -sSL https://get.docker.com/ | CHANNEL=stable sudo sh
```

Creating the directory and downloading the wings binary:
```bash
sudo mkdir -p /etc/pelican /var/run/wings
sudo curl -L -o /usr/local/bin/wings "https://github.com/pelican-dev/wings/releases/latest/download/wings_linux_$([[ "$(uname -m)" == "x86_64" ]] && echo "amd64" || echo "arm64")"
sudo chmod u+x /usr/local/bin/wings
```

Create a node on the web UI and paste the auto deploy script into the terminal. This should work if everything is configured correctly.

Create the service in `/etc/systemd/system/wings.service`:
```
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
```

Start the daemon:
```bash
sudo systemctl enable --now wings
```

## Backups
Because I don't want to set up S3 at home, I have opted in to using a local backup for Wings. This will back up servers into a local directory.

On TrueNAS, I have:
- Created the dataset for backups
- Created the UID and GID for Pelican
- Applied ACL rules to allow R/W on the dataset
- Created a NFS share and added the Wings IP to the allow list
- Under the NFS share advanced settings:
	- Set the mapall user and group to the Pelican user (*Note below*)

*Note: The mapall just sets all the clients' user or group to the selected one, basically this just gives all of them the permissions of that selected user. I did this because I'm not too familiar with NFS and didn't want to sync the users and groups for the two machines. Also the IP whitelist should be plenty for internal usage.*

On the Pelican Node machine:
- Install `nfs-common`
- Create the backups directory with `mkdir -p /mnt/backups`
- Make the directory immutable with `chattr +i /mnt/backups` (*Note below*)
- Append `10.0.10.2:/mnt/DataPool/PelicanBackups    /mnt/backups    nfs    defaults,hard,nofail,_netdev    0    0` to `/etc/fstab` (*Note below*)
- Modify the `backup_directory` key in the `/etc/pelican/config.yml` to the new backups path
- Restart Wings with `systemctl restart wings`

*Note: Mounting creates an overlay over the directory its pointing to, and so if I make the local directory immutable, the backups should fail is the NFS volume isn't mounted*

*Note: `hard` causes the client to wait indefinitely when the NFS mount goes down; `nofail` lets the OS know that it is ok to continue booting when this is not mounted on boot; `_netdev` tells the OS that it should only try to mount this after a network connection has been established *

On the Pelican Web UI:
- Under `Admin -> Settings -> Backup`:
	- Set the `Backup Driver` to Wings
- Under `Admin -> Servers -> Environment Configuration`:
	- Under `Feature Limits`, set `Backups` above 0
- Under the game server:
	- Create a `Schedule` and save it
	- Edit the `Schedule` and add the `Create Backup` task to it