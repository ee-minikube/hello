# Build a simple helloworld Golang WebServer in a Docker Container &amp; Deploy the Golang Hello World docker image  on Minikube. Load Balance the Hello World Service.

Note these steps are for a MacOS computer

## Steps: Build the Golang application and deploy to a docker container

### Get a docker hub account if you dont have one - you'll need it to install docker.

See here: https://hub.docker.com/signup

### Install Docker on Mac

https://docs.docker.com/docker-for-mac/install/

### Make sure you have brew installed on your Mac

https://brew.sh

### Install Minikube using brew and add extensions

Open a terminal on the Mac.

```bash
$ brew install minikube
$ minikube start
$ minikube addons enable ingress
```
### Install git using brew

```bash
$ brew install git
```

### Clone this repo

```bash
$ git clone https://github.com/ee-minikube/hello.git
$ cd hello
```

### Install local docker registry

```bash
$ docker run -d -p 5000:5000 --restart=always --name registry registry:2
```


### Golang 

View the main.go file with your favourite editor its here in the repo src/main. It basically uses resposeWriter as the Web Server serving Hello World and the hostname of the container:

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

### Dockerfile

View the Dockerfile with your favourite editor Dockerfile. It builds the Golang helloworld app in debian stretch docker container and deploys to latest alpine - then pushes to eu.gcr.io/guestbook-171610/helloworld:latest - note you will need to set up your own registry see below .

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

## Steps: Build the Golang helloworld app in debian stretch docker and deploy to latest alpine - with a google gcr tag push  latest to gcr.

### Open a terminal in Mac and run the following from the hello directory:

As above gcr has scanning on it and will show any vuneralbilities. 

Choose one or the other I've used Google gcr but Docker Hub is also shown below

### Either Push to Google gcr

Here's a good blog on using GCP GCR https://www.techbeatly.com/2019/10/adding-container-images-to-google-container-registry-gcr.html/.

```bash
$ docker build  -t eu.gcr.io/guestbook-171610/helloworld:latest .
$ docker push      eu.gcr.io/guestbook-171610/helloworld:latest
$ docker images |grep helloworld
```

### OR Push to Docker Repo

Set up do

```bash
$ docker build  -t bjrooney/helloworld .
$ docker push      bjrooney/helloworld
$ docker images |grep helloworld
```

## Steps: Deploy helloworld to minikube

In terminal run:

```bash
$  minikube dashboard
````

which launches something like  http://127.0.0.1:65041/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/#/overview?namespace=default in a browser leave the terminal opened and open another one for use later.


## Deploying the docker image to kube

### From the browser window that was opened click on the + in the top right.

This will open a new resource window  - choose the create from form tab:

![Create](/images/create.png)

App Name: helloworld

Container Image: eu.gcr.io/guestbook-171610/helloworld:latest (Note if you use docker hub eg: docker.io/bjrooney/helloworld)

Number of Pods: 3

Service: External

Port: 8080 External Port: 8080 Protocol: TCP

click on show Advanced Options

Namespace: Choose create a new namespace.
Will launch a dialog give the namespace name helloworld and hit enter

Description: helloworld app

Then click on Deploy

This will deploy and then open the helloworld namespace overview display which should look as this:

![Overview](/images/overview.png)

All should be green

This creates a deployment json which looks like this:
```json

kind: Deployment
apiVersion: apps/v1
metadata:
  name: helloworld
  namespace: helloworld
  selfLink: /apis/apps/v1/namespaces/helloworld/deployments/helloworld
  uid: 115cd0b0-bdfd-4a51-a56e-7ff331e31ebb
  resourceVersion: '83821'
  generation: 1
  creationTimestamp: '2020-01-20T14:37:15Z'
  labels:
    k8s-app: helloworld
  annotations:
    deployment.kubernetes.io/revision: '1'
    description: helloworld app
spec:
  replicas: 3
  selector:
    matchLabels:
      k8s-app: helloworld
  template:
    metadata:
      name: helloworld
      creationTimestamp: null
      labels:
        k8s-app: helloworld
      annotations:
        description: helloworld app
    spec:
      containers:
        - name: helloworld
          image: 'eu.gcr.io/guestbook-171610/helloworld:latest'
          resources: {}
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          imagePullPolicy: Always
          securityContext:
            privileged: false
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      dnsPolicy: ClusterFirst
      securityContext: {}
      schedulerName: default-scheduler
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600
status:
  observedGeneration: 1
  replicas: 3
  updatedReplicas: 3
  readyReplicas: 3
  availableReplicas: 3
  conditions:
    - type: Available
      status: 'True'
      lastUpdateTime: '2020-01-20T14:37:28Z'
      lastTransitionTime: '2020-01-20T14:37:28Z'
      reason: MinimumReplicasAvailable
      message: Deployment has minimum availability.
    - type: Progressing
      status: 'True'
      lastUpdateTime: '2020-01-20T14:37:28Z'
      lastTransitionTime: '2020-01-20T14:37:15Z'
      reason: NewReplicaSetAvailable
      message: ReplicaSet "helloworld-c77d9899b" has successfully progressed.
```

## Deploying ingress controller

You then need to deploy the ingress controller which will give you an nginx instance  which is done by choosing the correct namespace ie helloworld on the left band menu then click on ingresses followed by the + top right



```bash
$ kubectl config set-context minikube --namespace helloworld
$ kubectl apply -f ingress.yml
```

Where ingress.yml is in the attached repo


```yaml
apiVersion: networking.k8s.io/v1beta1 # for versions before 1.14 use extensions/v1beta1
kind: Ingress
metadata:
  name: helloworld-ingress
  namespace: helloworld
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
  - host: hello-world.info
    http:
      paths:
      - path: /
        backend:
          serviceName: helloworld
          servicePort: 8080
```

![Ingres](/images/ingres.png)

## Finally edit hosts adding the endpoint displayed on the helloworld namespace ingress page

```bash
$ sudo vi /etc/hosts
```

Which should look something like this after editing
```bash
##
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
127.0.0.1       localhost
255.255.255.255 broadcast
192.168.64.3    hello-world.info
::1             localhost
# Added by Docker Desktop
# To allow the same kube context to work on the host and the container:
127.0.0.1 kubernetes.docker.internal
# End of section
```
Where hello-world.info is the url

## In the browser visit: http://hello-world.info/ 

Hitting refresh will cycle through the 3 servers

![Web](/images/Web.png)
