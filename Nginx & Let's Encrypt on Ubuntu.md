
# Secure Nginx with Let's Encrypt on Ubuntu 14.04

### A little bit about the leading actors 

[Let's Encrypt](https://letsencrypt.org/about/) is a new Certificate Authority (CA) that provides an easy way to obtain and install free TLS/SSL certificates, thereby enabling encrypted HTTPS on web servers. It simplifies the process by providing a software client, `letsencrypt`, that attempts to automate most (if not all) of the required steps. 

We will use Let's Encrypt to obtain a free SSL certificate and use it with Nginx on Ubuntu 14.04\. We will also make it automatically renew your SSL certificate. 

![Nginx with Let's Encrypt TLS/SSL Certificate and Auto-renewal](https://assets.digitalocean.com/articles/letsencrypt/nginx-letsencrypt.png)

## What we need to make this happen

Before get started, you'll need a few things.

1. ave an [Ubuntu 14.04 server with a non-root user](https://www.digitalocean.com/community/articles/initial-server-setup-with-ubuntu-14-04) who has `sudo` privileges. 

2. Point our domain or sub-domain to the server. If you haven't already, be sure to create an **A Record** that points your domain to the public IP address of your server. This is required because of how Let's Encrypt validates that you own the domain it is issuing a certificate for. For example, if you want to obtain a certificate for `example.com`, that domain must resolve to your server for the validation process to work. Our setup will use `example.com` and `www.example.com` as the domain names, so **both DNS records are required**.

Once you have all of the prerequisites out of the way, let's move on to installing the Let's Encrypt client software.

## Step 1 — Install Let's Encrypt Client

The first step to using Let's Encrypt(It had renamed to CertBot in May, 2016) to obtain an SSL certificate is to install the `letsencrypt` software on your server. Currently, the best way to install Let's Encrypt is to simply clone it from the official GitHub repository. In the future, it will likely be available via a package manager.

### Install Git and bc

Let's install Git and bc now, so we can clone the Let's Encrypt repository.

Update your server's package manager with this command:

```
sudo apt-get update

```

Then install the `git` and `bc` packages with apt-get:

```
sudo apt-get -y install git bc

```

With `git` and `bc` installed, we can easily download `letsencrypt` by cloning the repository from GitHub.

### Clone Let's Encrypt

We can now clone the Let’s Encrypt repository in `/opt` with this command:

```
sudo git clone https://github.com/certbot/certbot /opt/certbot

```

You should now have a copy of the `certbot` repository in the `/opt/certbot` directory.

## Step 2 — Obtain a Certificate

Let's Encrypt provides a variety of ways to obtain SSL certificates, through various plugins. Plugins that only obtain certificates, and don't install them, are referred to as "authenticators" because they are used to authenticate whether a server should be issued a certificate.

We'll use the **Webroot** plugin to obtain an SSL certificate.

### How To Use the Webroot Plugin

The Webroot plugin works by placing a special file in the `/.well-known` directory within your document root, which can be opened (through your web server) by the Let's Encrypt service for validation. Depending on your configuration, you may need to explicitly allow access to the `/.well-known` directory.

If you haven't installed Nginx yet, do so with this command:

```
sudo apt-get install nginx

```

To ensure that the directory is accessible to Let's Encrypt for validation, let's make a quick change to our Nginx configuration. By default, it's located at `/etc/nginx/sites-available/default`. We'll use `nano` to edit it:

```
sudo nano /etc/nginx/sites-available/default

```

Inside the server block, add this location block:

<div class="code-label " title="Add to SSL server block">Add to SSL server block</div>

```
        location ~ /.well-known {
                allow all;
        }

```

You will also want look up what your document root is set to by searching for the `root` directive, as the path is required to use the Webroot plugin. If you're using the default configuration file, the root will be `/usr/share/nginx/html`.

Save and exit.

Reload Nginx with this command:

```
sudo service nginx reload

```

Now that we know our `webroot-path`, we can use the Webroot plugin to request an SSL certificate with these commands. Here, we are also specifying our domain names with the `-d` option. If you want a single cert to work with multiple domain names (e.g. `example.com` and `www.example.com`), be sure to include all of them. Also, make sure that you replace the highlighted parts with the appropriate webroot path and domain name(s):

```
cd /opt/certbot
./certbot-auto certonly -a webroot --webroot-path=/usr/share/nginx/html -d example.com -d www.example.com

```

<span class="note">**Note:** The Let's Encrypt software requires superuser privileges, so you will be required to enter your password if you haven't used `sudo` recently.
</span>

After `letsencrypt` initializes, you will be prompted for some information. The exact prompts may vary depending on if you've used Let's Encrypt before, but we'll step you through the first time.

At the prompt, enter an email address that will be used for notices and lost key recovery:

![Email prompt](https://assets.digitalocean.com/articles/letsencrypt/le-email.png)

Then you must agree to the Let's Encrypt Subscribe Agreement. Select Agree:

![Let's Encrypt Subscriber's Agreement](https://assets.digitalocean.com/articles/letsencrypt/le-agreement.png)

If everything was successful, you should see an output message that looks something like this:

```
Output:IMPORTANT NOTES:
 - If you lose your account credentials, you can recover through
   e-mails sent to sammy@digitalocean.com
 - Congratulations! Your certificate and chain have been saved at
   /etc/letsencrypt/live/example.com/fullchain.pem. Your
   cert will expire on 2016-03-15. To obtain a new version of the
   certificate in the future, simply run Let's Encrypt again.
 - Your account credentials have been saved in your Let's Encrypt
   configuration directory at /etc/letsencrypt. You should make a
   secure backup of this folder now. This configuration directory will
   also contain certificates and private keys obtained by Let's
   Encrypt so making regular backups of this folder is ideal.
 - If like Let's Encrypt, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le

```

You will want to note the path and expiration date of your certificate, which was highlighted in the example output.

<span class="note">

**Firewall Note:** If you receive an error like `Failed to connect to host for DVSNI challenge`, your server's firewall may need to be configured to allow TCP traffic on port `80` and `443`.

**Note:** If your domain is routing through a DNS service like CloudFlare, you will need to temporarily disable it until you have obtained the certificate.

</span>

### Certificate Files

After obtaining the cert, you will have the following PEM-encoded files:

*   **cert.pem:** Your domain's certificate
*   **chain.pem:** The Let's Encrypt chain certificate
*   **fullchain.pem:** `cert.pem` and `chain.pem` combined
*   **privkey.pem:** Your certificate's private key

It's important that you are aware of the location of the certificate files that were just created, so you can use them in your web server configuration. The files themselves are placed in a subdirectory in `/etc/letsencrypt/archive`. However, Let's Encrypt creates symbolic links to the most recent certificate files in the `/etc/letsencrypt/live/<span class="highlight">your_domain_name</span>` directory. Because the links will always point to the most recent certificate files, this is the path that you should use to refer to your certificate files.

You can check that the files exist by running this command (substituting in your domain name):

```
sudo ls -l /etc/letsencrypt/live/your_domain_name

```

The output should be the four previously mentioned certificate files. In a moment, you will configure your web server to use `fullchain.pem` as the certificate file, and `privkey.pem` as the certificate key file.

### Generate Strong Diffie-Hellman Group

To further increase security, you should also generate a strong Diffie-Hellman group. To generate a 2048-bit group, use this command:

```
sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048

```

This may take a few minutes but when it's done you will have a strong DH group at `/etc/ssl/certs/dhparam.pem`.

## Step 3 — Configure TLS/SSL on Web Server (Nginx)

Now that you have an SSL certificate, you need to configure your Nginx web server to use it.

Edit the Nginx configuration that contains your server block. Again, it's at `/etc/nginx/sites-available/default` by default:

```
sudo nano /etc/nginx/sites-available/default

```

Find the `server` block. **Comment out** or **delete** the lines that configure this server block to listen on port 80\. In the default configuration, these two lines should be deleted:

<div class="code-label " title="Nginx configuration deletions">Nginx configuration deletions</div>

```
        listen 80 default_server;
        listen [::]:80 default_server ipv6only=on;

```

We are going to configure this server block to listen on port 443 with SSL enabled instead. Within your `server {` block, add the following lines but replace all of the instances of `example.com` with your own domain:

<div class="code-label " title="Nginx configuration additions — 1 of 3">Nginx configuration additions — 1 of 3</div>

```
        listen 443 ssl;

        server_name example.com www.example.com;

        ssl_certificate /etc/certbot/live/example.com/fullchain.pem;
        ssl_certificate_key /etc/certbot/live/example.com/privkey.pem;

```

This enables your server to use SSL, and tells it to use the Let's Encrypt SSL certificate that we obtained earlier.

To allow only the most secure SSL protocols and ciphers, and use the strong Diffie-Hellman group we generated, add the following lines to the same server block:

<div class="code-label " title="Nginx configuration additions — 2 of 3">Nginx configuration additions — 2 of 3</div>

```
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;
        ssl_dhparam /etc/ssl/certs/dhparam.pem;
        ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
        ssl_session_timeout 1d;
        ssl_session_cache shared:SSL:50m;
        ssl_stapling on;
        ssl_stapling_verify on;
        add_header Strict-Transport-Security max-age=15768000;

```

Lastly, outside of the original server block (that is listening on HTTPS, port 443), add this server block to redirect HTTP (port 80) to HTTPS. Be sure to replace the highlighted part with your own domain name:

<div class="code-label " title="Nginx configuration additions — 3 of 3">Nginx configuration additions — 3 of 3</div>

```
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}

```

Save and exit.

Now put the changes into effect by reloading Nginx:

```
sudo service nginx reload

```

The Let's Encrypt TLS/SSL certificate is now in place. At this point, you should test that the TLS/SSL certificate works by visiting your domain via HTTPS in a web browser.

You can use the Qualys SSL Labs Report to see how your server configuration scores:

```
In a web browser:https://www.ssllabs.com/ssltest/analyze.html?d=example.com

```

This SSL setup should report an **A+** rating.

## Step 4 — Set Up Auto Renewal

Let’s Encrypt certificates are valid for 90 days, but it’s recommended that you renew the certificates every 60 days to allow a margin of error. At the time of this writing, automatic renewal is still not available as a feature of the client itself, but you can manually renew your certificates by running the Let’s Encrypt client with the `renew` option.

To trigger the renewal process for all installed domains, run this command:

```
/opt/certbot/certbot-auto renew

```

Because we recently installed the certificate, the command will only check for the expiration date and print a message informing that the certificate is not due to renewal yet. The output should look similar to this:

```
Output:Checking for new version...
Requesting root privileges to run letsencrypt...
   /root/.local/share/letsencrypt/bin/letsencrypt renew
Processing /etc/certbot/renewal/example.com.conf

The following certs are not due for renewal yet:
  /etc/certbot/live/example.com/fullchain.pem (skipped)
No renewals were attempted.

```

Notice that if you created a bundled certificate with multiple domains, only the base domain name will be shown in the output, but the renewal should be valid for all domains included in this certificate.

A practical way to ensure your certificates won’t get outdated is to create a cron job that will periodically execute the automatic renewal command for you. Since the renewal first checks for the expiration date and only executes the renewal if the certificate is less than 30 days away from expiration, it is safe to create a cron job that runs every week or even every day, for instance.

Let's edit the crontab to create a new job that will run the renewal command every week. To edit the crontab for the root user, run:

```
sudo crontab -e

```

Add the following lines:

```
30 2 * * 1 /opt/certbot/certbot-auto renew >> /var/log/le-renew.log
35 2 * * 1 /etc/init.d/nginx reload

```

Save and exit. This will create a new cron job that will execute the `certbot-auto renew` command every Monday at 2:30 am, and reload Nginx at 2:35am (so the renewed certificate will be used). The output produced by the command will be piped to a log file located at `/var/log/le-renewal.log`.

<span class="note">For more information on how to create and schedule cron jobs, you can check our [How to Use Cron to Automate Tasks in a VPS](https://www.digitalocean.com/community/tutorials/how-to-use-cron-to-automate-tasks-on-a-vps) guide.
</span>

## Step 5 — Updating the Let’s Encrypt Client (optional)

Whenever new updates are available for the client, you can update your local copy by running a `git pull` from inside the Certbot directory:

```
cd /opt/certbot
sudo git pull

```

This will download all recent changes to the repository, updating your client.

## NODE set

That's it! Your web server is now using a free Let's Encrypt TLS/SSL certificate to securely serve HTTPS content.
