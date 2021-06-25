# Ubuntu 18.04 (LEMP)

👏 Author: Adapted from public docs of very well known cloud providers.

This guide is for **Ubuntu 18.04.3 (LTS) x64** and it will install Chevereto in a LEMP stack (Linux, NGINX, MySQL and PHP).

> Replace `example.com` with your own domain

## Prepare the system

Before begin, make sure that the base system is updated.

```sh
sudo apt update && sudo apt upgrade
```

Go with the recommended option if asked about updating or keep packages.

## Install NGINX

```sh
sudo apt install nginx
```

## Install MySQL database

```sh
sudo apt install mysql-server
```

Enter to the MySQL console.

```sh
sudo mysql -u root
```

Once in the MySQL console, you will see a `mysql>` prompt. Run the following statements:

```sh
CREATE DATABASE chevereto;
CREATE USER 'chevereto' IDENTIFIED BY 'enter_a_password_here';
GRANT ALL ON chevereto.* TO 'chevereto' IDENTIFIED BY 'enter_a_password_here';
quit
```

**Note:** You must take note of the database name, user and password (where it reads `enter_a_password_here`) as these details will be required later on.

Once the database, user and permissions are set, run the following command to secure your MySQL installation.

```sh
sudo mysql_secure_installation
```

Answer all the questions as follows.

```sh
Set root password? [Y/n] n
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
```

## Install PHP

```sh
sudo apt install php-fpm php-zip php-curl php-mbstring php-gd php-mysql
```

Create the `chevereto.ini` file using the nano editor.

```sh
sudo nano /etc/php/7.2/fpm/conf.d/chevereto.ini
```

Paste `Ctrl+Shift+V` this:

```sh
upload_max_filesize = 50M;
post_max_size = 50M;
max_execution_time = 30;
memory_limit = 512M;
```

Write close `Ctrl+o Ctrl+x`.

## Setup the website

Create the path for the website files and assign the `www-data` owner and group.

```sh
sudo mkdir -p /var/www/html/example.com/public_html
sudo chown www-data:www-data /var/www/html/example.com/public_html
```

Remove the default NGINX website.

```sh
sudo rm -f /etc/nginx/sites-enabled/default
```

Create the website configuration file using the nano editor.

```sh
sudo nano /etc/nginx/sites-available/example.com.conf
```

Paste `Ctrl+Shift+V` this:

```nginx
server {
    listen         80 default_server;
    listen         [::]:80 default_server;
    server_name    example.com;
    root           /var/www/html/example.com/public_html;
    index          index.html;

    # Context limits
    client_max_body_size 50M;

    # Disable access to sensitive files
    location ~* (app|content|lib)/.*\.(po|php|lock|sql)$ {
        deny all;
    }

    # Image not found replacement
    location ~ \.(jpe?g|png|gif|webp)$ {
        log_not_found off;
        error_page 404 /content/images/system/default/404.gif;
    }

    # CORS header (avoids font rendering issues)
    location ~ \.(ttf|ttc|otf|eot|woff|woff2|font.css|css|js)$ {
        add_header Access-Control-Allow-Origin "*";
    }

    # Pretty URLs
    location / {
        index index.php;
        try_files $uri $uri/ /index.php$is_args$query_string;
    }

    location ~* \.php$ {
        fastcgi_pass unix:/run/php/php7.2-fpm.sock;
        include         fastcgi_params;
        fastcgi_param   SCRIPT_FILENAME    $document_root$fastcgi_script_name;
        fastcgi_param   SCRIPT_NAME        $fastcgi_script_name;
    }
}
```

Write close `Ctrl+o Ctrl+x`.

Create the website symbolic link (site available).

```sh
sudo ln -s /etc/nginx/sites-available/example.com.conf /etc/nginx/sites-enabled/
```

Restart PHP and NGINX.

```sh
sudo systemctl restart php7.2-fpm
sudo systemctl restart nginx
```

## Setup HTTPS

Install the Certbot and web server-specific packages, then run Certbot.

```sh
sudo apt install python-certbot-nginx
sudo certbot --nginx
```

Answer yes when it ask if you will like to automatically redirect HTTP traffic to HTTPS traffic.

When the tool completes, Certbot will update your web server configuration so that it uses the new certificate.

## Install Chevereto

Download the installer to your website directory, masked as www-data.

```sh
sudo -u www-data wget -O /var/www/html/example.com/public_html/installer.php https://chevereto.com/download/file/installer
```

Open your website at `/installer.php`, follow the process (ignore the warning about Nginx rules). Once done, go to the dashboad and make sure that the connecting IP match yours.

## Real connecting IP (CloudFlare, optional)

If you run CloudFlare you will need to configure the `ngx_http_realip_module`. Install the CloudFlare IP ranges sync script.

```sh
wget -P /opt/scripts https://raw.githubusercontent.com/ergin/nginx-cloudflare-real-ip/master/cloudflare-sync-ips.sh
chmod +x /opt/scripts/cloudflare-sync-ips.sh
/opt/scripts/cloudflare-sync-ips.sh
```

The file `/etc/nginx/cloudflare` should include now the CloudFlare IP ranges, which now must be included into your NGINX configuration at `/etc/nginx/nginx.conf`. Include the following line inside the `http{}` block:

```sh
include /etc/nginx/cloudflare;
```

Add a crontab to update CloudFlare IP ranges everyday.

```sh
sudo crontab -e
```

If asked about the editor, choose `nano`.

Paste `Ctrl+Shift+V` this:

```sh
# Auto sync Cloudflare IP ranges and reload NGINX
0 */12 * * * /opt/scripts/cloudflare-sync-ips.sh >/dev/null 2>&1
```

Write close `Ctrl+o Ctrl+x`.
