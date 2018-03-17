# docker-private-registry

```
git clone https://github.com/victorskl/docker-private-registry.git
cd docker-private-registry
mkdir -p data
docker-compose build
docker-compose -p dev up -d
```

### Working with Registry

Make a quick push:

```
docker pull alpine:latest
docker tag alpine:latest localhost:5000/my-alpine
docker push localhost:5000/my-alpine
docker rmi localhost:5000/my-alpine
docker pull localhost:5000/my-alpine
```

Docker Registry V2 available at:

```
curl localhost:5000/v2/
curl localhost:5000/v2/my-alpine/tags/list
```

Clean up:

```
docker rmi localhost:5000/my-alpine
```

### Working through UI

UI available at:  [http://localhost:8888](http://localhost:8888)

Apparently, the UI web app also expose the [`/v2/` Registry endpoint](https://github.com/kwk/docker-registry-frontend/issues/75). So, you can also do HTTP GET the image through it from the Registry; similarly:

```
curl localhost:8888/v2/
curl localhost:8888/v2/my-alpine/tags/list
docker pull localhost:8888/my-alpine
```

But you can not push it through as it is in `ENV_MODE_BROWSE_ONLY` enforced; at the present.

```
docker tag alpine:latest localhost:8888/my-alpine
docker push localhost:8888/my-alpine

...
...
error parsing HTTP 403 response body: invalid character '<' looking for beginning of value: "<!DOCTYPE HTML PUBLIC \"-//IETF//DTD HTML 2.0//EN\">\n<html><head>\n<title>403 Forbidden</title>\n</head><body>\n<h1>Forbidden</h1>\n<p>You don't have permission to access /v2/project/my-alpine/blobs/uploads/\non this server.<br />\n</p>\n<hr>\n<address>Apache/2.4.10 (Debian) Server at reg.my-domain.com Port 80</address>\n</body></html>\n"
```

### Basic Auth, SSL Termination with HAProxy

- Install HAProxy and adapt [haproxy.cfg](haproxy.cfg) to your environment; 80 or 443

- Install Certbot if using TLS; adjust haproxy config

    ```
    mkdir -p /etc/haproxy/certs
    chmod 600 /etc/haproxy/certs
    
    certbot certonly --standalone --rsa-key-size 4096 --preferred-challenges http --http-01-port 10000 --noninteractive --agree-tos --email victorsankho.lin@email.address -d reg.my-domain.com
    certbot certificates
    cat /etc/letsencrypt/live/reg.my-domain.com/fullchain.pem /etc/letsencrypt/live/reg.my-domain.com/privkey.pem > /etc/haproxy/certs/reg.my-domain.com.pem
    ```

- Prepare additional env config for UI

    ```
    cp env.sample .env
    vi .env [change reg.my-domain.com to your FQDN]
    ```


---

REF

- https://docs.docker.com/registry/deploying/
- https://docs.docker.com/registry/spec/api/
- https://github.com/kwk/docker-registry-frontend