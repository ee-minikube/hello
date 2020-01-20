# hello
build &amp; deploy golang helloworld example on minikube

Go:

Get a docker hub account if you dont have one - you'll need it to install docker.

See here: https://hub.docker.com/signup

Install Docker on Mac

https://docs.docker.com/docker-for-mac/install/

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

Dockerfile

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
