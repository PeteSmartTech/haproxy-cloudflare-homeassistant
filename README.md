# HAProxy + Cloudflare + HomeAssistant
This guide is a proposed solution for the issue https://github.com/home-assistant/core/issues/40421

I struggled myself to resolve that and expose my HA instance safely so I came up to a resolution.

# Adjust the `haproxy.cfg`
We need to edit the `haproxy.cfg`, put Your IP's of the HomeAssistant instance.

Let's start by setting Your Public IP for the ACL in this line
```
acl from_public_ip req.hdr(CF-Connecting-IP) -i <your-public-IP>
```

Then we set the hostname of Your HomeAssistant instance
```
acl hst_homeassistant hdr(host) -i home.yourdomain.tld 
```

Then we set the backend for the HomeAssistant
```
backend bk_homeassistant
  mode http
  server homeassistant <homeassistantIP>:8123 check verify none
```

The issue `400:Bad request` is caused by HomeAssistant receiving many headers from proxy and it doesn't know what to do with them. We can fix it by removing the headers from the request, and adding only ones that are necessary.
```
http-request del-header X-Forwarded-For
http-request del-header X-Forwarded-Proto

option forwardfor except 127.0.0.1

http-request add-header X-Forwarded-Proto https
http-request add-header X-Forwarded-Port 443
```

### Overall the 'haproxy.cfg' from this repo is ready to be used, but feel free to adjust it to Your own needs.


# ACL that allows access only from our Public IP
We can add an ACL that will allow access only from our Public IP. This is useful if we want to access our HomeAssistant instance only from our home network, preventing any unauthorized access.

This ACL is added at the beginning of the `frontend` section, before the `use_backend` rule.
```
  acl from_public_ip req.hdr(CF-Connecting-IP) -i <your-public-IP>
  http-request set-header X-From-Public-IP true if from_public_ip
```
And in the backend section, we add a rule that will deny access if the `X-From-Public-IP` header is not present.
```
  http-request allow if { req.hdr(X-From-Public-IP) -m str true } 
  http-request deny
```


# Cloudflare's Origin CA (recommended)
If we want to use the Cloudflare's Origin CA, we must obtain it at our Cloudflare Dashboard at `ourdomain.tld -> SSL/TLS -> Origin Server -> Create Certificate`

We can include the hostname's we need, select the lifetime of the certificate and then generate it.

This certificate will be served by HAProxy and it will be used to secure the connection between the Cloudflare and our HAProxy.

This is the recommended way to secure the connection between the Cloudflare and our HAProxy.

# Self-signed certificate
We can also use the self-signed certificate for the HAProxy, but it's not recommended. The browser will not give warnings about certificate because the traffic is proxied by Cloudflare, however, it's not the best practice.

# Let's Encrypt certificate
We can use the Let's Encrypt certificate for our HAProxy. We just have to remember to renew it every 90 days.
If we proxy the traffic through Cloudflare's network it won't matter much since we will only see the Cloudflare's certificate in the browser.

# Combine the certificate into single .pem
HAProxy requires that both Privatekey and Certificate will be combined into a single .pem file. We can do it by using cat

Remember that order matters, so first goes the private key followed by the certificate.  
```
cat privatekey.pem certificate.pem > combined.pem
```

When we have our combined certificate ready, we can put it in `/opt/haproxy/certs`

# Run the HAProxy
We can run the HAProxy by using the command
```
docker compose up -d
```
Now we can access our HomeAssistant instance by using the `home.yourdomain.tld` and it will be secured by the Cloudflare's Origin CA.
