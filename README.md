# docker-private-registry

```
git clone https://github.com/victorskl/docker-private-registry.git
cd docker-private-registry
mkdir -p data
docker-compose build
docker-compose -p dev up -d
```

### Working direct with Registry

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

Apparently, the UI web app also expose the [`/v2/` Registry endpoint](https://github.com/kwk/docker-registry-frontend/issues/75). So, you can also post the image through it to the Registry; similarly:

```
docker tag alpine:latest localhost:8888/my-alpine
docker push localhost:8888/my-alpine
curl localhost:8888/v2/
curl localhost:8888/v2/my-alpine/tags/list
```


REF

- https://docs.docker.com/registry/deploying/
- https://docs.docker.com/registry/spec/api/
- https://github.com/kwk/docker-registry-frontend