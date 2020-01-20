# Build a simple Hello World Golang WebServer in a Docker Container &amp; Deploy the Golang Hello World docker image  on Minikube. Load Balance the Hello World Service.

Note these steps are for a MacOS computer

Steps: Build the Golang application and deploy to a docker container

Get a docker hub account if you dont have one - you'll need it to install docker.

See here: https://hub.docker.com/signup

Install Docker on Mac

https://docs.docker.com/docker-for-mac/install/

Make sure you have brew installed on your Mac

Install Minikube using brew and add extensions

brew install minikube

minikube start
minikube addons enable ingress

Install git using brew

brew install git

Clone this repo

git clone 

 
Heres the golang:

```bash
package main

import (
	"fmt"
	"net/http"
	"os"
	"log"
)

func sayHello(w http.ResponseWriter, r *http.Request) {

	hostname, error := os.Hostname()
	 if error != nil {
	  panic(error)
	 }
	 
	fmt.Fprint(w, "Hello World: Server  : ")
	fmt.Fprint(w, hostname)
	log.Println("said hello")
}

func main() {
	http.HandleFunc("/", sayHello)

	// get port env var
	port := "8080"
	portEnv := os.Getenv("PORT")
	if len(portEnv) > 0 {
		port = portEnv
	}

	log.Printf("Listening on port %s...", port)
	log.Fatal(http.ListenAndServe(fmt.Sprintf(":%s", port), nil))

}
```
Here's the dockerfile:

Dockerfile

```bash
FROM golang:1.13.6-stretch as builder
# install dep
RUN go get github.com/golang/dep/cmd/dep
# create a working directory
WORKDIR /go/src/app
# add Gopkg.toml and Gopkg.lock
ADD Gopkg.toml Gopkg.toml
ADD Gopkg.lock Gopkg.lock
# install packages
RUN dep ensure --vendor-only
# add source code
ADD src src
# build the source
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main src/main.go

# use a minimal alpine image
FROM alpine:3.11.3
# add ca-certificates in case you need them
RUN apk add --no-cache ca-certificates
# set working directory
WORKDIR /root
# copy the binary from builder
COPY --from=builder /go/src/app/main .
```

Build the Golang helloworld app in debian stretch docker and deploy to latest alpine - with a google gcr tag push  latest to gcr.

Open a terminal in Mac and run the following:

```bash
docker build  -t eu.gcr.io/guestbook-171610/helloworld .
docker push eu.gcr.io/guestbook-171610/helloworld:latest
docker images |grep helloworld
```

Deploy helloworld to minikube

In terminal run:

minikube dashboard

which launches http://127.0.0.1:65041/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/#/overview?namespace=default in a browser leave the terminal opened and open another one for use later.

From the browser window that was opened click on the + in the top right.

This will open a new resource window  - choose the create from form tab
App Name helloworld
Container Image: eu.gcr.io/guestbook-171610/helloworld:latest
Number of Pods: 3
Service: External
Port: 8080 External Port: 8080 Protocol: TCP
click on show Advanced Options

Description: helloworld app

Then click on Deploy

This will deploy and then open the helloworld namespace overview display which should look as this:



Namespace: Choose create a new namespace.
Will launch a dialog give the namespace name helloworld

minikube dashboard

