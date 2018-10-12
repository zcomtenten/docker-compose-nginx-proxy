# Nginx-Proxy

This docker-compose.yml users the **official nginx** and the **official certbot** container. It has optimized nginx configuration to be used as a https proxy together with certbot. Following my instructions you should get an <span style="color:green; font-weight:bold;">A+ rating</span> at [ssllabs.com](https://www.ssllabs.com/ssltest).



The container will use the network **www-network** as a proxy-tier. Add every container to this network that servers as a upstream http host.

## Directory structure

```
.
├── conf.d                       # Site-specific configuration
│   ├── example.com.conf
│   ├── ...
├── protect                      # HTTP Password Protection
│   ├── .htpasswd
├── snippets                     # Config we want to reuse at conf.d files
│   ├── certbot-webroot.conf     # Serves Let's encrypt .well-known files
│   ├── certbot-standalone.conf  # as alternative method
│   ├── proxy.conf           
│   ├── ssl.conf             

```

## How To Use

### 1. Clone this repo

```bash
mkdir -p /docker/00-nginx-proxy
cd /docker/00-nginx-proxy
git clone https://github.com/sebastian13/docker-compose-nginx-proxy.git .
```

### 2. Prepare for SSL
```bash
mkdir -p ./ssl/test
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
	-keyout ./ssl/test/selfsigned.key \
	-out ./ssl/test/selfsigned.crt
openssl dhparam -out ssl/dhparams2048.pem 2048
```

### 3. Create your site's config
```bash
cp conf.d/{example.com,yoursite.com.conf}
```
Replace *example.com* with your domain, and set your *$upstream container*.

### 4. Ready to go
```bash
docker-compose up -d
```

## Nginx

I'm using the [official nginx container](https://hub.docker.com/_/nginx/) here. All volumes will be mounted read-only.

### Manually reload the configuration
As you change site-specific configuration in *conf.d*, you should consider reloading the configuration instead of restarting the container. This is because your container will not start if the configuration contains errors.

```
docker exec nginx-proxy nginx -s reload
```

## Let's Encrypt SSL Certificates

### Reqeust a new Certificate

```
docker-compose run --rm certbot certonly \
 --agree-tos --no-eff-email --hsts --webroot -w /var/www \
 --cert-name=example.com -m mail@example.com -d example.com
```

Then, link the certificate in your nginx site.conf + reload the nginx-proxy.


### List existing Certificates
```shell
docker-compose run --rm certbot certificates
```

### Delete existing Certificates
```shell
docker-compose run --rm certbot delete --cert-name example.com
```

### Renew Certificates

Define a Cronjob like this, to renew the certificates periodically.

```
0 0 * * SAT docker-compose -f /docker/00-nginx-proxy/docker-compose.yml up certbot && docker exec nginx-proxy nginx -s reload
```

To manually check your certificates for renewal run `docker-compose up certbot`.


## Get A+ SSL Rating

1. Generate your own Diffie-Hellman parameters. Put it inside the directory **ssl**.

 `openssl dhparam -out ssl/dhparams2048.pem 2048`

2. Include the **ssl.conf snippet** at your site specific configuration. Also, include the **ssl\_trusted\_certificate**.

 ```
 server {
 		...
 		ssl_trusted_certificate /etc/nginx/ssl/live/example.com/chain.pem;
  		include /etc/nginx/snippets/ssl.conf;
  		...
 }
 ```

## Password protection
Protect your site with http authentication.

```
sudo apt-get update
sudo apt-get install apache2-utils
```

Run this command. You will be asked to supply and confirm a password for the user.

```bash
sudo htpasswd -c /etc/nginx/protect/.htpasswd first_user
sudo htpasswd /etc/nginx/protect/.htpasswd another_user
```

```
server {
  ...
  location / {
    auth_basic "Restricted Content";
    auth_basic_user_file /etc/nginx/protect/.htpasswd;
  } 
} 
```

You can find detailed instructions at [digitalocean](https://www.digitalocean.com/community/tutorials/how-to-set-up-password-authentication-with-nginx-on-ubuntu-14-04)