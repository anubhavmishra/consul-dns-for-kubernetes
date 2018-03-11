# consul-dns-for-kubernetes
Consul DNS Interface for Kubernetes

## Overview

In this tutorial we will learn how to configure Kubernetes ([kube-dns](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)) to discover services registered in Consul using Consul's DNS interface.

## Prerequisites

This tutorial requires a Kubernetes cluster.

* kubernetes 1.8.7

### Kubernetes cluster using Google Cloud Platform

If you don't have a Kubernetes running, you can use [GKE](https://cloud.google.com/kubernetes-engine/) to start one up.

**Installation**

Install CLI tools for Google Cloud by downloading them [here](https://cloud.google.com/sdk/).

Configure Google Cloud region and zone

```bash
gcloud config set compute/region "us-west1"
gcloud config set compute/zone "us-west1-b"
```

Create Kubernetes cluster

```bash
gcloud beta container clusters create us-west1-b \
  --cluster-version 1.8.7-gke.1 \
  --machine-type n1-standard-2 \
  --scopes "cloud-platform,storage-ro,logging-write,monitoring-write,service-control,service-management,https://www.googleapis.com/auth/ndev.clouddns.readwrite"
```

*The scopes above can be tailored towards the permissions you might need.*

Save cluster credentials

```bash
gcloud container clusters get-credentials us-west1-b
```

### Clone Repository

The Consul DNS Interface for Kubernetes repository holds deployment files, scripts, and guides that are required to follow the tutorial. 

Use the following command to clone the repository

```bash
git clone https://github.com/anubhavmishra/consul-dns-for-kubernetes.git
```

The tutorial assumes that you are in `consul-dns-for-kubernetes` directory throughout the tutorial.

Use the following command to go to tutorial directory

```bash
cd consul-dns-for-kubernetes
```

## Bootstrap Consul Server

This tutorial assumes that the consul cluster is not running on Kubernetes. If you want to run a consul cluster on Kubernetes, follow the [consul-on-kubernetes](https://github.com/kelseyhightower/consul-on-kubernetes) by [Kelsey Hightower](https://github.com/kelseyhightower/).

In this tutorial, we will create a 3 node consul cluster.

Create Consul Servers

```bash
gcloud compute instances create consul-1 consul-2 consul-3 \
  --image-project ubuntu-os-cloud \
  --image-family ubuntu-1604-lts \
  --boot-disk-size 250GB \
  --machine-type n1-standard-1 \  
  --can-ip-forward \
  --scopes default,compute-ro \
  --tags "consul-server" \
  --metadata-from-file startup-script=scripts/bootstrap-consul-server.sh
```

Create consul deployment

The consul deployment starts two consul agents in client mode inside the Kubernetes cluster. In this tutorial, we will use Consul's [cloud auto join](https://www.consul.io/docs/agent/options.html#cloud-auto-joining) feature to discover Consul servers running outside the Kubernetes cluster. 

```bash
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: consul
....
        args:
          - "agent"
          - "-advertise=$(POD_IP)"
          - "-bind=0.0.0.0"
          - "-client=0.0.0.0"
          - "-data-dir=/var/consul"
          - "-datacenter=dc1"
          - "-retry-join=provider=gce tag_value=consul-server"
....
```

Notice we are using `-retry-join=provider=gce tag_value=consul-server` to use tags to discover the Consul servers.

```bash
kubectl apply -f deployments/consul.yaml
deployment "consul" created
```

This will create a deployment with two replicas of the consul agent running in client mode.

Check consul deployment, list pods for the deployment

```bash
kubectl get pods --namespace=kube-system | grep consul
```

```bash
consul-8f858fd45-7m5b8                                 1/1       Running   0          35s
consul-8f858fd45-sknrk                                 1/1       Running   0          36s
```

Create consul service

```bash
kubectl apply -f services/consul-dns.yaml
service "consul-dns" created
``` 

Check consul service

```bash
kubectl describe service consul-dns --namespace=kube-system
```

```bash
Name:              consul-dns
Namespace:         kube-system
Labels:            name=consul-dns
Annotations:       kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"name":"consul-dns"},"name":"consul-dns","namespace":"kube-system"},"spec":{...
Selector:          name=consul
Type:              ClusterIP
IP:                10.35.250.89
Port:              dns-tcp  53/TCP
TargetPort:        dns-tcp/TCP
Endpoints:         10.32.0.41:8600,10.32.2.41:8600
Port:              dns-udp  53/UDP
TargetPort:        dns-udp/UDP
Endpoints:         10.32.0.41:8600,10.32.2.41:8600
Session Affinity:  None
Events:            <none>
```

The `consul-dns` service exposes the Consul DNS interface to the pods running in the Kubernetes cluster.

Create kube-dns configmap

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    addonmanager.kubernetes.io/mode: EnsureExists
  name: kube-dns
  namespace: kube-system
data:
  stubDomains: |
    {"consul": ["$(kubectl get svc consul-dns --namespace=kube-system -o jsonpath='{.spec.clusterIP}')"]}
EOF
```

This deligates `.consul` domain to Consul DNS interface exposed by `consul-dns` service.

Test consul dns from inside kubernetes

```bash
kubectl apply -f job/dns.yaml
```

Get the pod name for dns 

```bash
kubectl get pods --show-all | grep dns
```

```bash
dns-hqglg                  0/1       Completed   0          20s
```

Get logs from the job

```bash
kubectl logs dns-hqglg 
```

Confirm `consul.service.consul` lookup is successful

```bash

; <<>> DiG 9.11.2-P1 <<>> consul.service.consul
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 43374
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;consul.service.consul.         IN      A

;; ANSWER SECTION:
consul.service.consul.  0       IN      A       10.138.0.5
consul.service.consul.  0       IN      A       10.138.0.7
consul.service.consul.  0       IN      A       10.138.0.6

;; Query time: 19 msec
;; SERVER: 10.35.240.10#53(10.35.240.10)
;; WHEN: Sat Mar 10 06:57:44 UTC 2018
;; MSG SIZE  rcvd: 98

```