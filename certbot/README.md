# Let's Encrypt Certbot with Cloudflare DNS Plugin
In case You'd like to generate Let's Encrypt certificates for Your setup with Cloudflare, you can do it by utilizing these commands.  
It simply spawns a certbot, tells that we are using a dns-cloudflare plugin (which we should have installed at this point)  
If not here is how to intall it:
`https://certbot.eff.org/`

Install `snap` if using other distro than Ubuntu.

In this guide we are using Ubuntu

### First let's remove any certbot packages currently installed in our system (if any exist)  
`sudo apt-get remove certbot`

### Then let's install the newest certbot with snap
`sudo snap install --classic certbot`

### Then we will install the certbot's Cloudflare plugin
`sudo snap install certbot-dns-cloudflare`

### We will setup a symlink to ensure that `certbot` command can be run
`sudo ln -s /snap/bin/certbot /usr/bin/certbot`

### We need to generate an API key to access Cloudflare's DNS Zone's by the Certbot
The Token needed by Certbot requires `Zone:DNS:Edit` permissions for only the zones you need certificates for.
### Here's the link to obtain an API key:
`https://dash.cloudflare.com/?to=/:account/profile/api-tokens`

### Example credentials file using restricted API Token (recommended):

### cloudflare.ini  
```
Cloudflare API token used by Certbot
dns_cloudflare_api_token = 0123456789abcdef0123456789abcdef01234567
```

After we created the cloudflare.ini let's restrict access to it by using `chmod 600 cloudflare.ini`

Then we agree Terms of Service, enter our email, we give the certbot a path to our Cloudflare API key to allow Certbot to verify ownership of our domain and we set the propagation time to 60 seconds.

Then we can pass the domains. We can set the root domain and generate a wildcard certificate `*.domain.tld`

However if You want to issue a higher level wildcard like `*.something.domain.tld`, You need to specify that aswell because the `*.domain.tld` wildcard will not cover it  

We assume that the `secret` for certbot is stored in the same path as HAProxy, so `/opt/haproxy/certbot/`

### Adjust paths to Your own needs if You don't like the way I am doing it.

```
certbot certonly \
  --dns-cloudflare \
  --non-interactive --agree-tos --email your@email.tld \
  --dns-cloudflare-credentials /opt/haproxy/certbot/secrets/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 60 \
  -d *.yourdomain.tld \
  -d yourdomain.tld
  ```

Certbot will save certificates in the `/etc/letsencrypt/live` path

We want to combine both certificate and privatekey into a single .pem file (because HAProxy requires that format)

We can do it by using the cat and tee, and pass the single .pem into the HAProxy certificates directory

For the sake of this guide we assume that the path we keep HAProxy data in `/opt/haproxy/`

Accordingly the certificates will be stored in `/opt/haproxy/certs/`

```
sudo cat /etc/letsencrypt/live/yourdomain.tld/fullchain.pem \
    /etc/letsencrypt/live/yourdomain.tld/privkey.pem \
    | sudo tee /opt/haproxy/certs/yourdomain.tld.pem
```


### Read more about Certbot's Cloudflare DNS Plugin:
`https://certbot-dns-cloudflare.readthedocs.io/en/stable/`
