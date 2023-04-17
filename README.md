# Kubernetes Cloud Provider for KIND

KIND has demonstrated to be a very versatile, efficient, cheap and very useful tool for Kubernetes testing. However, KIND doesn't offer capabilities for testing all the features that depend on cloud-providers, specially the Load Balancers, causing a gap on testing and a bad user experience, since is not easy to connect to the applications running on the cluster.

`cloud-provider-kind` aim to fill this gap and provide an agnostic and cheap solution for all the Kubernetes features that depend on a cloud-provider using KIND.

- [Slack channel](https://kubernetes.slack.com/messages/kind)
- [Mailing list](https://groups.google.com/forum/#!forum/kubernetes-sig-testing)

## How to use it

Run a KIND cluster:

```sh
$ kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.26.0) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂

```

**Note**

Control-plane nodes need to remove the special label `node.kubernetes.io/exclude-from-external-load-balancers` to be able to access the workloads running on those nodes using a LoadBalancer Service.

```sh
$ kubectl label node kind-control-plane node.kubernetes.io/exclude-from-external-load-balancers-
node/kind-control-plane unlabeled
```

Once the cluster is running, we need to run the `cloud-provider-kind` in a terminal and keep it running. The `cloud-provider-kind` will monitor all your KIND clusters and `Services` with Type `LoadBalancer` and create the correponding LoadBalancer containers that will expose those Services.

```sh
bin/cloud-provider-kind
I0416 19:58:18.391222 2526219 controller.go:98] Creating new cloud provider for cluster kind
I0416 19:58:18.398569 2526219 controller.go:105] Starting service controller for cluster kind
I0416 19:58:18.399421 2526219 controller.go:227] Starting service controller
I0416 19:58:18.399582 2526219 shared_informer.go:273] Waiting for caches to sync for service
I0416 19:58:18.500460 2526219 shared_informer.go:280] Caches are synced for service
...
```

### Creating a Service and exposing it via a LoadBalancer

Let's create an application that listens on port 8080 and expose it in the port 80 using a LoadBalancer.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: policy-local
  labels:
    app: MyLocalApp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: MyLocalApp
  template:
    metadata:
      labels:
        app: MyLocalApp
    spec:
      containers:
      - name: agnhost
        image: registry.k8s.io/e2e-test-images/agnhost:2.40
        args:
          - netexec
          - --http-port=8080
          - --udp-port=8080
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: lb-service-local
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  selector:
    app: MyLocalApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

```sh
$ kubectl apply -f examples/loadbalancer_etp_local.yaml
deployment.apps/policy-local created
service/lb-service-local created
$ kubectl get service/lb-service-local
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
lb-service-local   LoadBalancer   10.96.207.137   192.168.8.7   80:31215/TCP   57s
```

We can see how the `EXTERNAL-IP` field contains an IP, and we can use it to connect to our
application.

```
$ curl  192.168.8.7:80/hostname
policy-local-59854877c9-xwtfk

$  kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
policy-local-59854877c9-xwtfk   1/1     Running   0          2m38s
```


**Note**

Only tested with `docker` and `Linux`, but `podman` and `Windows` and `Mac`users feedback will be helpful to support those platforms.

The project is still in very alpha state, bugs are expected, please report them back opening a Github issue.

### Code of conduct

Participation in the Kubernetes community is governed by the [Kubernetes Code of Conduct](code-of-conduct.md).

[owners]: https://git.k8s.io/community/contributors/guide/owners.md
[Creative Commons 4.0]: https://git.k8s.io/website/LICENSE