# docker-private-registry

A very light weight solution for hosting Docker images in private deployment and an on-premise data-center. It is a simple stack using the official Docker Registry application with file-back storage; Basic Auth and TLS with HAProxy, Let's Encrypt; simple UI for browsing the images.

### On Local Dev

```
git clone https://github.com/victorskl/docker-private-registry.git
cd docker-private-registry
mkdir -p data
touch .env
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

Docker Registry V2 API available at:

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

However, you can not push it through as it is in `ENV_MODE_BROWSE_ONLY` enforced; at the present.

```
docker tag alpine:latest localhost:8888/my-alpine
docker push localhost:8888/my-alpine
...
...
error parsing HTTP 403 response body: invalid character '<' looking for beginning of value: "<!DOCTYPE HTML PUBLIC \"-//IETF//DTD HTML 2.0//EN\">\n<html><head>\n<title>403 Forbidden</title>\n</head><body>\n<h1>Forbidden</h1>\n<p>You don't have permission to access /v2/project/my-alpine/blobs/uploads/\non this server.<br />\n</p>\n<hr>\n<address>Apache/2.4.10 (Debian) Server at reg.my-domain.com Port 80</address>\n</body></html>\n"
```

This is fine; we will be using HAProxy to front both services.

### Using HAProxy, Let's Encrypt, Basic Auth, SSL Termination

- Install HAProxy and adapt [haproxy.cfg](haproxy.cfg) to your environment; configure for 443 and cert location.

- Install [Certbot](https://letsencrypt.org); adjust haproxy config for TLS.

    ```
    mkdir -p /etc/haproxy/certs
    chmod 600 /etc/haproxy/certs
    
    certbot certonly --standalone --rsa-key-size 4096 --preferred-challenges http --http-01-port 10000 --noninteractive --agree-tos --email victorsankho.lin@email.address -d reg.my-domain.com
    certbot certificates
    cat /etc/letsencrypt/live/reg.my-domain.com/fullchain.pem /etc/letsencrypt/live/reg.my-domain.com/privkey.pem > /etc/haproxy/certs/reg.my-domain.com.pem
    ```

- Additional env config for UI

    ```
    rm .env
    cp env.sample .env
    vi .env [change reg.my-domain.com to your FQDN]
    ```


## Maintenance with Registry

- Note: if you are on TLS + Basic Auth, use the `-u` flag and https FQDN instead. The rest of the steps; left this out for the brevity.

```
curl -u victor:ChangeMe https://reg.my-doamin.com/v2/
```

### API

- Drilling down [the Registry API](https://docs.docker.com/registry/spec/api/#detail)

```
curl localhost:5000/v2/
curl localhost:5000/v2/_catalog
curl localhost:5000/v2/my-alpine/tags/list
curl localhost:5000/v2/my-alpine/manifests/latest
```

- Output of manifests query explained at: https://docs.docker.com/registry/spec/manifest-v2-2/

- Get the repo tag `:latest` manifests layers; _make sure you did this and save the output elsewhere; bec we will need the digest sha later in deleting blob layers section_.

```
curl -H "Accept: application/vnd.docker.distribution.manifest.v2+json" localhost:5000/v2/my-alpine/manifests/latest
    {
        "schemaVersion": 2,
        "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
        "config": {
            "mediaType": "application/vnd.docker.container.image.v1+json",
            "size": 1512,
            "digest": "sha256:3fd9065eaf02feaf94d68376da52541925650b81698c53c6824d92ff63f98353"
        },
        "layers": [
            {
                "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
                "size": 2065537,
                "digest": "sha256:ff3a5c916c92643ff77519ffa742d3ec61b7f591b6b7504599d95a4a41134e28"
            }
        ]
    }
```

### Deleting un-wanted images

- Let's [delete the image](https://docs.docker.com/registry/spec/api/#deleting-an-image) from the Registry

```
curl -I -H "Accept: application/vnd.docker.distribution.manifest.v2+json" localhost:5000/v2/my-alpine/manifests/latest
    HTTP/1.1 200 OK
    Content-Length: 528
    Content-Type: application/vnd.docker.distribution.manifest.v2+json
    Docker-Content-Digest: sha256:8c03bb07a531c53ad7d0f6e7041b64d81f99c6e493cb39abba56d956b40eacbc
    Docker-Distribution-Api-Version: registry/2.0
    Etag: "sha256:8c03bb07a531c53ad7d0f6e7041b64d81f99c6e493cb39abba56d956b40eacbc"
    X-Content-Type-Options: nosniff
    Date: Sun, 18 Mar 2018 01:59:41 GMT

curl -X DELETE localhost:5000/v2/my-alpine/manifests/sha256:8c03bb07a531c53ad7d0f6e7041b64d81f99c6e493cb39abba56d956b40eacbc
    {
        "errors": [
            {
                "code": "UNSUPPORTED",
                "message": "The operation is unsupported."
            }
        ]
    }
```

- This is because; we need to enable `Registry -> Storage -> Delete` setting to `true`. Default is `false`. The decision is environment specific; you might want to be cautious in Production environment and keep this disabled.

- [Overriding the default setting for Registry](https://docs.docker.com/registry/configuration/) can be done in the `.env` file or the docker-compose `environment` or the `volume` mount [registry/config.yml](registry/config.yml). Let turn it on:

```
echo "REGISTRY_STORAGE_DELETE_ENABLED=true" >> .env
docker-compose -p dev up -d
docker exec -it dev_registry_1 env
```

- We can delete the image `:latest` this time.

```
curl -vvv -X DELETE localhost:5000/v2/my-alpine/manifests/sha256:8c03bb07a531c53ad7d0f6e7041b64d81f99c6e493cb39abba56d956b40eacbc
```

### Deleting un-wanted blob layer

- For [the Blob](https://docs.docker.com/registry/spec/api/#blob) and to delete [the blobs layer](https://docs.docker.com/registry/spec/api/#deleting-a-layer), get the digests from previous step; _see, I told ya!_.

```
curl localhost:5000/v2/my-alpine/blobs/sha256:3fd9065eaf02feaf94d68376da52541925650b81698c53c6824d92ff63f98353
{
    "architecture": "amd64",
    "config": {
        "Hostname": "",
        "Domainname": "",
        "User": "",
        "AttachStdin": false,
        "AttachStdout": false,
        "AttachStderr": false,
        "Tty": false,
        "OpenStdin": false,
        "StdinOnce": false,
        "Env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
        ],
        "Cmd": [
            "/bin/sh"
        ],
        "ArgsEscaped": true,
        "Image": "sha256:fbef17698ac8605733924d5662f0cbfc0b27a51e83ab7d7a4b8d8a9a9fe0d1c2",
        "Volumes": null,
        "WorkingDir": "",
        "Entrypoint": null,
        "OnBuild": null,
        "Labels": null
    },
    "container": "30e1a2427aa2325727a092488d304505780501585a6ccf5a6a53c4d83a826101",
    "container_config": {
        "Hostname": "30e1a2427aa2",
        "Domainname": "",
        "User": "",
        "AttachStdin": false,
        "AttachStdout": false,
        "AttachStderr": false,
        "Tty": false,
        "OpenStdin": false,
        "StdinOnce": false,
        "Env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
        ],
        "Cmd": [
            "/bin/sh",
            "-c",
            "#(nop) ",
            "CMD [\"/bin/sh\"]"
        ],
        "ArgsEscaped": true,
        "Image": "sha256:fbef17698ac8605733924d5662f0cbfc0b27a51e83ab7d7a4b8d8a9a9fe0d1c2",
        "Volumes": null,
        "WorkingDir": "",
        "Entrypoint": null,
        "OnBuild": null,
        "Labels": {}
    },
    "created": "2018-01-09T21:10:58.579708634Z",
    "docker_version": "17.06.2-ce",
    "history": [
        {
            "created": "2018-01-09T21:10:58.365737589Z",
            "created_by": "/bin/sh -c #(nop) ADD file:093f0723fa46f6cdbd6f7bd146448bb70ecce54254c35701feeceb956414622f in / "
        },
        {
            "created": "2018-01-09T21:10:58.579708634Z",
            "created_by": "/bin/sh -c #(nop)  CMD [\"/bin/sh\"]",
            "empty_layer": true
        }
    ],
    "os": "linux",
    "rootfs": {
        "type": "layers",
        "diff_ids": [
            "sha256:cd7100a72410606589a54b932cabd804a17f9ae5b42a1882bd56d263e02b6215"
        ]
    }
}

curl -X DELETE -vvv localhost:5000/v2/my-alpine/blobs/sha256:3fd9065eaf02feaf94d68376da52541925650b81698c53c6824d92ff63f98353
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 5000 (#0)
> DELETE /v2/my-alpine/blobs/sha256:3fd9065eaf02feaf94d68376da52541925650b81698c53c6824d92ff63f98353 HTTP/1.1
> Host: localhost:5000
> User-Agent: curl/7.54.0
> Accept: */*
>
< HTTP/1.1 202 Accepted
< Content-Length: 0
< Docker-Distribution-Api-Version: registry/2.0
< X-Content-Type-Options: nosniff
< Date: Sun, 18 Mar 2018 03:00:44 GMT
< Content-Type: text/plain; charset=utf-8
<
* Connection #0 to host localhost left intact


curl -X DELETE -vvvv localhost:5000/v2/my-alpine/blobs/sha256:ff3a5c916c92643ff77519ffa742d3ec61b7f591b6b7504599d95a4a41134e28
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 5000 (#0)
> DELETE /v2/my-alpine/blobs/sha256:ff3a5c916c92643ff77519ffa742d3ec61b7f591b6b7504599d95a4a41134e28 HTTP/1.1
> Host: localhost:5000
> User-Agent: curl/7.54.0
> Accept: */*
>
< HTTP/1.1 202 Accepted
< Content-Length: 0
< Docker-Distribution-Api-Version: registry/2.0
< X-Content-Type-Options: nosniff
< Date: Sun, 18 Mar 2018 03:24:21 GMT
< Content-Type: text/plain; charset=utf-8
<
* Connection #0 to host localhost left intact
```

### Garbage Collection

- Normally, it may not need to manually delete the blob layer; but just delete the un-wanted image tag or version. Then, when no manifest is referencing to the the specify blob layer, we can use [garbage-collector](https://docs.docker.com/registry/garbage-collection/) facility to automatically and routinely purge the unused layers; therefore regain the storage space. Let have a dry-run.

```
docker exec -it dev_registry_1 bin/registry garbage-collect /etc/docker/registry/config.yml --dry-run
my-alpine

0 blobs marked, 4 blobs eligible for deletion
blob eligible for deletion: sha256:3fd9065eaf02feaf94d68376da52541925650b81698c53c6824d92ff63f98353
blob eligible for deletion: sha256:8c03bb07a531c53ad7d0f6e7041b64d81f99c6e493cb39abba56d956b40eacbc
blob eligible for deletion: sha256:a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4
blob eligible for deletion: sha256:ff3a5c916c92643ff77519ffa742d3ec61b7f591b6b7504599d95a4a41134e28
```

- And let's perform the gc:

```
docker exec -it dev_registry_1 bin/registry garbage-collect /etc/docker/registry/config.yml
my-alpine

0 blobs marked, 4 blobs eligible for deletion
blob eligible for deletion: sha256:8c03bb07a531c53ad7d0f6e7041b64d81f99c6e493cb39abba56d956b40eacbc
INFO[0000] Deleting blob: /docker/registry/v2/blobs/sha256/8c/8c03bb07a531c53ad7d0f6e7041b64d81f99c6e493cb39abba56d956b40eacbc  go.version=go1.7.6 instance.id=bdcbcd4c-051a-4c55-b0bd-15056859a392
blob eligible for deletion: sha256:a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4
INFO[0000] Deleting blob: /docker/registry/v2/blobs/sha256/a3/a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4  go.version=go1.7.6 instance.id=bdcbcd4c-051a-4c55-b0bd-15056859a392
blob eligible for deletion: sha256:ff3a5c916c92643ff77519ffa742d3ec61b7f591b6b7504599d95a4a41134e28
INFO[0000] Deleting blob: /docker/registry/v2/blobs/sha256/ff/ff3a5c916c92643ff77519ffa742d3ec61b7f591b6b7504599d95a4a41134e28  go.version=go1.7.6 instance.id=bdcbcd4c-051a-4c55-b0bd-15056859a392
blob eligible for deletion: sha256:3fd9065eaf02feaf94d68376da52541925650b81698c53c6824d92ff63f98353
INFO[0000] Deleting blob: /docker/registry/v2/blobs/sha256/3f/3fd9065eaf02feaf94d68376da52541925650b81698c53c6824d92ff63f98353  go.version=go1.7.6 instance.id=bdcbcd4c-051a-4c55-b0bd-15056859a392
```

- https://gbougeard.github.io/blog.english/2017/05/20/How-to-clean-a-docker-registry-v2.html


### Deleting un-wanted repository

___NOTE: DO THESE STEPS WITH EXTREMELY CAUTIOUS___

- Perform deleting all images/manifests from the repo `my-alpine`.

- Perform garbage-collect; especially tail on the target repo `my-alpine`.

- When none exist in the target repo (_can check through UI_), bring the Registry stack down and rm the target repo `my-alpine`.

```
docker-compose -p dev down

tree -L 1 data/docker/registry/v2/repositories                                
data/docker/registry/v2/repositories
└── my-alpine

1 directory, 0 files

cd data/docker/registry/v2/repositories/
rm -rf my-alpine

docker-compose -p dev up -d
``` 

---

REF

- https://docs.docker.com/registry/deploying/
- https://docs.docker.com/registry/spec/api/
- https://github.com/kwk/docker-registry-frontend