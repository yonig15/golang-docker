# Sample Golang app for CI
The example application in the repository is a simple HTTP server that listens on a port and returns a "Hello, World!" message when accessed. The repository includes a Dockerfile that can be used to build a Docker image of the application

## Build base image
```
make build-base
```

## Build test image
```
make build-test
```

## Run tests (make build-test first)
```
make test-units
```

## Run project
``` bash
make run
```
default port is 8080
you can change the port using `PORT` argument
> make run PORT=8081

<!-- ## Deploy to kubernetes
```
kubectl create -f webapp.yml
```

## Deploy to kubernetes service
```
kubectl create -f webapp-service.yml
``` -->

<!-- ## Describe service
```
kubectl describe svc golang-app-service
``` -->