# docker-private-registry

```
git clone https://github.com/victorskl/docker-private-registry.git
cd docker-private-registry
mkdir -p data
docker-compose build
docker-compose -p dev up -d
```

Make a quick push:

```
docker pull alpine:latest
docker tag alpine:latest localhost:5000/my-alpine
docker push localhost:5000/my-alpine
docker rmi alpine:latest localhost:5000/my-alpine
docker pull localhost:5000/my-alpine
```

Docker Registry V2 available at:

```
curl localhost:5000/v2/
curl localhost:5000/v2/my-alpine/tags/list
```

UI available at:  [http://localhost:8888](http://localhost:8888)

REF

- https://docs.docker.com/registry/deploying/
- https://docs.docker.com/registry/spec/api/
- https://github.com/kwk/docker-registry-frontend