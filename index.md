## 2017-04-02 - Installing seafile on a Raspberry Pi with raspbian

```
$ uname -a
Linux raspberrypi 4.4.50-v7+ #970 SMP Mon Feb 20 19:18:29 GMT 2017 armv7l GNU/Linux
```

1. Create a working place for yourself. `mkdir -p /srv/seafile/installed`
1. Download stable release for Raspberry Pi. `/srv/seafile/installed# wget https://github.com/haiwen/seafile-rpi/releases/download/v6.0.8/seafile-server_6.0.8_stable_pi.tar.gz`
1. Create user for seafile. `adduser --system --group --disabled-login --no-create-home --home /srv/seafile --shell /bin/false seafile`
1. Install prerequisites. `apt-get install python2.7 libpython2.7 python-setuptools python-imaging python-ldap python-urllib3 sqlite3`
1. Extract the package. `/srv/seafile# tar -xzf /home/pi/seafile/installed/seafile-server_6.0.8_stable_pi.tar.gz`
1. Make seafile user own everything. `chown -R seafile:seafile /srv/seafile`
1. Execute initialization script. `/srv/seafile/seafile-server-6.0.8# sudo -u seafile ./setup-seafile.sh`
1. Install nginx. `apt-get install nginx`
1. Add file `/etc/nginx/sites-available/seafile.conf` with content:

```
server {
    listen 80;
    server_name seafile.example.com;

    root /var/www/html;
    location /.well-known {  # do not redirect requests for .well-known location
        allow all;
    }

    location / {  # the default location redirects to https
        return 301 https://$host$request_uri;
    }
}
```

10. Add file `/etc/nginx/sites-available/seafile.ssl.conf` with content:

```
server {
    listen 443;
    server_name seafile.example.com;

    ssl on;
    ssl_certificate /etc/letsencrypt/live/seafile.example.com/cert.pem;        # path to your cacert.pem
    ssl_certificate_key /etc/letsencrypt/live/seafile.example.com/privkey.pem;    # path to your privkey.pem

# For further ssl options look into: https://mozilla.github.io/server-side-tls/ssl-config-generator/?server=nginx-1.6.2&openssl=1.0.1t&hsts=yes&profile=intermediate

    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;

    # Diffie-Hellman parameter for DHE ciphersuites, recommended 2048 bits
#    ssl_dhparam /path/to/dhparam.pem;

    # intermediate configuration. tweak to your needs.
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers 'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS';
    ssl_prefer_server_ciphers on;

    # HSTS (ngx_http_headers_module is required) (15768000 seconds = 6 months)
    add_header Strict-Transport-Security max-age=15768000;

    # OCSP Stapling ---
    # fetch OCSP records from URL in ssl_certificate and cache them
    ssl_stapling on;
    ssl_stapling_verify on;

    ## verify chain of trust of OCSP response using Root CA and Intermediate certs
    ssl_trusted_certificate /etc/letsencrypt/live/seafile.example.com/fullchain.pem;

#    resolver <IP DNS resolver>;
# SSL end

    server_tokens off;

    proxy_set_header X-Forwarded-For $remote_addr;

		location / {
        fastcgi_pass    127.0.0.1:8000;
        fastcgi_param   SCRIPT_FILENAME     $document_root$fastcgi_script_name;
        fastcgi_param   PATH_INFO           $fastcgi_script_name;

        fastcgi_param   SERVER_PROTOCOL     $server_protocol;
        fastcgi_param   QUERY_STRING        $query_string;
        fastcgi_param   REQUEST_METHOD      $request_method;
        fastcgi_param   CONTENT_TYPE        $content_type;
        fastcgi_param   CONTENT_LENGTH      $content_length;
        fastcgi_param   SERVER_ADDR         $server_addr;
        fastcgi_param   SERVER_PORT         $server_port;
        fastcgi_param   SERVER_NAME         $server_name;
        fastcgi_param   REMOTE_ADDR         $remote_addr;
        fastcgi_param   HTTPS               on;
        fastcgi_param   HTTP_SCHEME         https;

        access_log      /var/log/nginx/seahub.access.log;
        error_log       /var/log/nginx/seahub.error.log;
        fastcgi_read_timeout 36000;
        client_max_body_size 0;
    }

		location /seafhttp {
        rewrite ^/seafhttp(.*)$ $1 break;
        proxy_pass http://127.0.0.1:8082;
        client_max_body_size   0;
        proxy_connect_timeout  36000s;
        proxy_read_timeout     36000s;
        proxy_send_timeout     36000s;
        send_timeout           36000s;
    }

    location /media {
        root /srv/seafile/seafile-server-latest/seahub;
    }
}
```

11. Install certbot. `wget https://dl.eff.org/certbot-auto && chmod a+x certbot-auto && ./certbot-auto`
1. Prepare nginx if your domain name is long. Add `server_names_hash_bucket_size 64;` to your `/etc/nginx/nginx.conf`.
1. Activate your http config. `ln -s /etc/nginx/sites-available/seafile.conf /etc/nginx/sites-enabled/seafile.conf && nginx -t && systemctl reload nginx`
1. Request certificate. `./certbot-auto certonly --webroot -w /var/www/html/ -d seafile.example.com`
1. Activate your https config. `ln -s /etc/nginx/sites-available/seafile.ssl.conf /etc/nginx/sites-enabled/seafile.ssl.conf && nginx -t && systemctl reload nginx`
1. Set `FILE_SERVER_ROOT = 'https://seafile.example.com/seafhttp'` in `/srv/seafile/conf/seahub_settings.py`.
1. Set `SERVICE_URL = https://seafile.example.net` in `/srv/seafile/conf/ccnet.conf`.
1. Start seafile. `/srv/seafile/seafile-server-latest$ sudo -u seafile ./seafile.sh start && sudo -u seafile ./seahub.sh start-fastcgi`

What remains to be done:

* Let certbot run regularily to refresh certificates.
* Let certbot run as `letsencrypt` user only.
* Wrap seahub.sh and seafile.sh in services so they start automatically after reboot.
* Add DH params file.

## 2017-02-19 - Changing owncloud from .deb to manual install

On a server I maintain, Owncloud is installed from the Debian repository.
However, the way the .deb package behaves does not fit properly with the rest of the services running on the server.
For example, the Apache config file has to be adjusted almost every update.
Also, you need to maintain the repositories separately in apt if new releases come in, and, on Debian, updates are comparably late.
Because of this, I want to change to a manual install that is not maintained by the package manager.

However, after some discussion with friends, I came to the conclusion that not only an upgrade and the change to a manual installation is in order, but also a change to Nextcloud.
The reasons for the change are:

* Faster development
* More dedicated to openness
* Compatible to the Owncloud client
* Maintainers who appear to take upgrade security more serious
* Features like ActiveSync and Outlook integration

I am aware that not all of these are hard facts.
Still, AS is one of the most important arguments to me because it enables my users to sync with the application without needing to resort to \*DAV adapters.

[The process to change to Nextcloud appears to be straight-forward.](https://nextcloud.com/migration/)
First I need to upgrade to ownCloud 8.2, then switch over to Nextcloud 9.0, then upgrade Nextcloud itself.
Because changing ownCloud to a manual install before switching over to Nextcloud, I will upgrade it using the Debian repositories, and install Nextcloud manually.
Before every step, I take a backup, and after every step, I ensure everything is working before proceeding.
This way, I have the possibility to salvage everything I did up to the point where it fails, and the day will not be wasted.

### 1. Make a backup

To be on the safe side, I backup the ownCloud folder, the database, and, in addition, my apache2 configuration files for ownCloud.

### 2. Upgrade to the latest ownCloud 8.1

Using apt, I intend to upgrade to ownCloud 8.1.9-12.1, which is the current version in the currently enabled Debian repository.
The involved packages are `owncloud`, `owncloud-config-apache`, and `owncloud-server`.
I was pondering on no longer using `owncloud-config-apache` and configure it myself, but that would be a waste of time at this point.
A problem I have is that I am still having the old opensuse repositories configured.
They need to go first, as I am getting warnings all over that the keys I had imported are no longer relevant or trusted.

#### 2.1 Adjust apt repository to that of ownCloud

I get the latest key and repository information from [ownCloud's download page](https://download.owncloud.org/download/repositories/stable/owncloud/) and configure my system accordingly.
I use this opportunity to move the ownCloud repository information out of my current main sources.list file into a separate one.
I also finally disable the Debian Wheezy sources I somehow forgot to remove.
Sloppy me.

Afterwards, I am really unhappy with the result, as apt now offers only the latest release of ownCloud, which is 9.1.4.
To get there, it is recommended to upgrade to 8.2 and 9.0 first, which I now cannot do using the apt package manager.
So I revert to the repository on opensuse.org using [opensuse's ownCloud 8.1 release channel page](http://software.opensuse.org/download/package?project=isv:ownCloud:community:8.1&package=owncloud).
While it seems pointless to download the GPG key for the repository via http, I still do it because it makes a man-in-the-middle at least more difficult.
Now apt shows me the correct packages to update and I can continue.

#### 2.2 Installing the upgrade to 8.1.12

After changing repository, I am offered version 8.1.12, which is more recent than I was offered before.
Yay, the effort of changing repository shows first advantages aside from the disappeared messages regarding untrusted repositories.

#### 2.3 Upgrade aftermath

The upgrade leaves me with a message to run `occ` manually and informs me that ownCloud is in maintenance mode for now.
Checking the page, sure enough, it is.
Checking in the ownCloud directory using `occ status`, I am informed some apps require an upgrade.
I run `occ upgrade`, check again with `occ status`, and then check my apps with `occ app:list`.
I enable calendar, contacts and documents again using the `occ` tool and disable the maintenance mode with `occ maintenance:mode --off`.
After checking ownClouds functionality, I am satisfied.
All the data is there, all the functionality is there, it appears to have gone well.

### 3. Upgrade to the latest ownCloud 8.2

Same as before: Add the proper repository, backup, upgrade, check functionality and data.
I add the [ownCloud 8.2 repository](https://download.owncloud.org/download/repositories/8.2/owncloud/) and its key and update apt's repositories.
The ownCloud package shows now 8.2.10 as most recent version and offers the upgrade.
I backup all files like I did the last time, and start the upgrade via apt.
Afterwards, while ownCloud is in maintenance mode, I check the status with the command line tool and perform `occ upgrade`.
After enabling the apps again and disabling maintenance mode, I check all functionality and make another backup, just in case.


To be continued.
