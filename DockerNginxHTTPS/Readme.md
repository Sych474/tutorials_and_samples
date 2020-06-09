# Docker with Nginx + HTTPS + self-signed SSL

Task: Create docker with Nginx to host app with HTTPS. Use the self-signed certificate, and make the certificate trusted on your PC. 

I use todoMVC app written by Vue js as an example app to host it using Nginx (see link on the end).

## 1. Creating self-signed certificate 

I use openssl for generating self-signed certificates. 

### 1.1 Generate root certificate
Using this commands - create root certificate.
```
openssl req -x509 -nodes -new -sha256 -days 365 -newkey rsa:2048 -keyout RootCA.key -out RootCA.pem -subj "/C=RU/CN=RootCA"
openssl x509 -outform pem -in RootCA.pem -out RootCA.crt
```

You will have 4 files:
- RootCA.crt
- RootCA.key
- RootCA.pem
- RootCA.srl

### 1.2 Add config file
Create file `domains.ext` and insert inside:

```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = localhost
DNS.2 = nginx-ssl.local
```

You can add additional DNS to this file, or change URL to your oun. 

### 1.3 Generate app certificate
Using this commands generate certificate for your app. 

```
openssl req -new -nodes -newkey rsa:2048 -keyout app.key -out app.csr -subj "/C=RU/ST=Moscow/L=Moscow/O=Test-Certificates/CN=nginx-ssl.local"
openssl x509 -req -sha256 -days 365 -in app.csr -CA RootCA.pem -CAkey RootCA.key -CAcreateserial -extfile domains.ext -out app.crt
```
You will have 3 files:
- app.crt
- app.key
- app.pem

You can customize in -subj your own
Country (C)
State (ST)
City/Location (L)
Organization (O)
URL of your app (CN)

**Now you have root sertificate and sertificate for your app**

## 2. Configuring Nginx

For enabling HTTPS in Nginx we need to add these lines to config:
```
    listen 443 ssl http2 default_server;
    listen [::]:443 ssl http2 default_server;
    
    ssl_certificate /etc/ssl/certs/app.crt;
    ssl_certificate_key /etc/ssl/private/app.key;
```

I take default nginx config and update it. This is my result [config](./default.conf).

## 3. Build docker 

Now we are ready for building our docker container. 

I use nginx:1.17.1-alpine image as base, and all we need is to copy:
- updated nginx configuration file - `default.conf` to `/etc/nginx/conf.d/`;
- certificate file - `app.crt` to `/etc/ssl/certs/`;
- certificate key - `app.key` to `/etc/ssl/private/`;
- app - `src` folder to `/usr/share/nginx/html` 

And finaly - restart nginx in docker after config update. 

This is full [Dockerfile](./Dockerfile).

to build docker use command:
```
docker build -t nginx-ssl .
```

## 4 Run docker 

Now, when we have a docker image - let's run it. We need to assign 443 ports of docker container and host.

You can use [docker-compose.yml](./docker-compose.yml) file from this repository by command:

```
docker-compose up -d
```

Or you can use another way to run the docker container.

## 5. Configuring a client to work with a self-signed certificate.

### 5.1. Windows 10

#### 5.1.1 update hosts
Add the line to the end of `c:\Windows\System32\drivers\etc\hosts` file:
```
127.0.0.1 nginx-ssl.local
```
Instead of 127.0.0.1, you can write the IP address of the machine running the app.
Note: you need to do it as administrator. 

#### 5.1.2 add the root certificate 
Go to Google Chrome, `Settings -> Manage certificates -> Trusted Certificate Authorities -> Import`. 
Select the file `RootCA.crt` from nginx_ssl folder, put all the checkmarks and click ok.

#### 5.1.3 reboot your PC

### 5.2. Linux(Ubuntu)

#### 5.2.1 update hosts
Add the line to the `/etc/hosts` file:
```
127.0.0.1 nginx-ssl.local
```
Instead of 127.0.0.1, you can write the IP address of the machine running the app.

#### 5.2.2 add the root certificate 
Go to Google Chrome, `Settings -> Manage certificates -> Authorities -> Import`. 
Select the file `RootCA.crt` from nginx_ssl folder, put all the checkmarks and click ok.

## 6. Congratulations! 
Now, go to https://nginx-ssl.local and enjoy.

## References  

- https://vuejs.org/v2/examples/todomvc.html - app, used as example
- https://gist.github.com/cecilemuller/9492b848eb8fe46d462abeb26656c4f8 - sertificates generation
