---
layout: post
authors: [pieter_vincken]
title: 'Istio Service Mesh: service to service communication'
image: /img/2019-04-14-istio-service-mesh-s2s/istio.jpeg
tags: [Istio, Service Mesh, Kubernetes, Cloud, Microservices]
category: Cloud
comments: true
---

> This post will describe how to use the Istio service mesh to provide service to service authentication and authorization in a Kubernetes cluster.
> It will show how ServiceRoles, ServiceRoleBindings and Identities in Istio can be used to achieve this. 

# Table of content 

* [What is Istio?](#what-is-istio)
* [Istio concepts](#istio-concepts)
* [Show me the code](#show-me-the-code)
* [Conclusion](#conclusion)

## What is Istio?

Istio is a service mesh created by Google, Lyft and IBM. 
It aims to simplify some security and management aspects of a microservices software architecture.
More information on Istio and its features can be found in its [docs](https://istio.io/docs/){:target="_blank" rel="noopener noreferrer"}.
In this blogpost we will highlight one of the key security features of Istio: service to service authentication and authorization.
For the sake of simplicity, this post will focus on an Istio setup in Kubernetes.

In a microservices architecture, managing access to services can be a challenging operation. 
For end-user facing services, JWTs are used to add authorization information to a request.
They are used by the service to determine which end-user is making the request.
These tokens can be generated based on information that the end-user provides to an identity provider.
In most cases this information is a username and password, with some additional 2FA if possible.
This setup can be achieved by using OpenID Connect as a protocol with the authorization code grant flow and an identity provider like Keycloak.

When services communicate with each other, they also need to provide an identity to each other.
A common option to do this is by using client credentials grant flow of OpenID Connect.
In this flow a service provides its client credentials to authenticate against the identity provider, and to be able to generate an access token once authenticated.
This token will be used to communicate to a service.

These are types of authorization flows on application level.
They allow services to determine what resources an end-user or service can access.
Istio's service to service role based acccess control (RBAC) is not on application level but on communication level.
It specifies which services can connect and communicate with each other. 
In order to achieve this, Istio connects an identity to each service in the mesh and allows it to authenticate itself.
The requested service can use this identity to determine if the service is allowed to connect or not.  
Istio makes use of proxies to handle all traffic (into and out of services) and using mutual trusted certificates to secure the connection and provide an identity to these proxies.
When using the automatic proxy injection, enabling Istio's service to service RBAC mechanism is almost as easy as flipping a switch.

There are five main components responsible for making this possible in Istio: Citadel, Pilot, Galley, Mixer and Envoy. 

*Citadel* is Istio's fortress of trust.
It manages all certificates and acts as a Root CA in the Istio setup.

*Galley* is the main configuration manager.
It is responsible for gathering all required information from the underlying platform.

*Pilot* manages all routing information and manages all the information for the proxies.
It will initialise the proxies during start-up with their configuration and the certificates from Citadel.

*Mixer* is responsible for all monitoring, logging and authorization information.
Whenever a proxy performs an action, Mixer knows about it. 
This allows it to both monitor and log connections, but also provide authorization information to the proxies. 

The final piece to the puzzle is *Envoy*. 
Envoy is the sidecar proxy responsible for handling the actual traffic between services in the service mesh.
It will setup and manage the required mTLS connections and perform all required check with regards to the routing. 
Envoy is managed as a separate project and in theory an other proxy could be used, but Envoy is most common.

<img class="image fit" style="margin:0px auto; max-width: 600px;" src="{{ '/img/2019-04-14-istio-service-mesh-s2s/arch.svg' | prepend: site.baseurl }}" alt="Istio architecture drawing" />

A final, optional component is the sidecar injector.
This component is not mandatory for the service mesh to work, but makes using it a lot easier.
The injector is set up as a mutating webhook admission controller. 
In a nutshell, this allows the injector to inspect and update some specific objects in the Kubernetes API.
It will automatically inject the Envoy sidecar proxy into every pod which needs it.

## Istio concepts

Istio stores all its configuration directly in the Kubernetes API through the use of Custom Resource Definitions (CRDs).
Next, a small description of the ones relevant for our blog are explained.

### Policies

[Policies](https://istio.io/docs/concepts/security/#authentication-policies){:target="_blank" rel="noopener noreferrer"} are at the heart of the mTLS setup in Istio.
They define when mTLS should be used and how.
Policies can be scoped in two levels: mesh wide (Mesh Policies) and namespace wide.

### Destination Rules

[Destination rules](https://istio.io/docs/concepts/traffic-management/#destination-rules){:target="_blank" rel="noopener noreferrer"} are a set of rules that are evaluated when a service is called.
They define multiple different routing options.
For the scope of this blogpost, they will only be used to define which services require to be accessed using mTLS.

### Service Roles

[Service Roles](https://istio.io/docs/concepts/security/#servicerole){:target="_blank" rel="noopener noreferrer"} are used in Istio to describe which access a role provides. 
It specifies which endpoints of a specific service can be used.
Currently this is described by specifying the full internal DNS name of the service, the methods and the paths that the role can access.

### Service Role Bindings

[Service Role Bindings](https://istio.io/docs/concepts/security/#servicerole){:target="_blank" rel="noopener noreferrer"} are used to connect identities (service accounts) or identity properties (namespaces) to actual roles. 
When a binding is created, the identities connected to it are allowed the access specified in the referenced service role.

## Show me the code

The Istio service to service authentication and authorization will now be explained by using an example setup.
Note that the code snippets have been shortened in this blogpost.
This is denoted with three dots `...`.
The full examples can be found in the accompanying repository on [Github](https://github.com/pietervincken/istio-service-to-service-demo){:target="_blank" rel="noopener noreferrer"}

### Prerequisites

This demo assumes that Istio is already installed in the cluster with the demo profile enabled. 
See [Install Istio](https://istio.io/docs/setup/kubernetes/install/helm/#option-1-install-with-helm-via-helm-template){:target="_blank" rel="noopener noreferrer"} for more information on the installation of Istio.
In the demo repository, a small [script](https://github.com/pietervincken/istio-service-to-service-demo/blob/master/1-install-istio.sh){:target="_blank" rel="noopener noreferrer"} can be found that can assist in setting up the demo environment

### Setup

The setup of our application is a very simple service with a database backend. 
Our service exposes one HTTP GET endpoint which will be accessed by the outside world. 
Our database is an Apache CouchDB instance.
Both the database and the service run inside Kubernetes.
The setup is shown in the image below.

<img class="image fit" style="margin:0px auto; max-width: 600px;" src="{{ '/img/2019-04-14-istio-service-mesh-s2s/setup.png' | prepend: site.baseurl }}" alt="demo setup" />

### Create namespace

First, a new namespace is created.
The service and database will both be added to this namespace.

```bash
kubectl create namespace with-istio
kubectl label namespace with-istio istio-injection=enabled
```

### Install CouchDB

Next, the database is installed.
This looks like a normal stateful set for a CouchDB database.
There are some important changes.

First, a specific service account is created for CouchDB. 
This is needed as Istio will use the service accounts in Kubernetes as its identities.
The service account is linked to the podspec in the stateful set definition. 
This way, it can be used by the Istio proxy later on.

Secondly, the probes have been adapted to work in Istio.
Since Istio intercepts all traffic in the pod, it will also intercept requests from the Kube API to the service.
Since the demo setup requires mTLS to be used, the probes would fail because the Kube API doesn't use mTLS.
Instead of manually changing the probes, Istio now has the option to rewrite the probes during the automatic proxy injection. 
More information on the probes can be found in the [Istio Docs](https://istio.io/help/ops/setup/app-health-check/#probe-rewrite){:target="_blank" rel="noopener noreferrer"}.

Note that no Istio specific configuration is required in the service manifests.
This is possible because the demo profile automatically enables the sidecar injector and we enabled the injection on the `with-istio` namespace using the `istio-injection=enabled` label.
The automatic sidecar injector will inject the Envoy sidecar into all pods.

```yaml
---
# Source: couchdb/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: couchdb
---
# Source: couchdb/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: couchdb
  labels:
   ...
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app.kubernetes.io/name: couchdb
    app.kubernetes.io/instance: couchdb
---
# Source: couchdb/templates/statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: couchdb
  labels:
    ...
spec:
  replicas: 1
  serviceName: couchdb
  selector:
    matchLabels:
      app.kubernetes.io/name: couchdb
      app.kubernetes.io/instance: couchdb
  template:
    metadata:
      labels:
        app.kubernetes.io/name: couchdb
        app.kubernetes.io/instance: couchdb
    spec:
      serviceAccountName: couchdb
      containers:
        - name: couchdb
          image: "couchdb:2.3.0"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 5984
              protocol: TCP
          livenessProbe:
            exec:
              command:
              - curl
              - http://localhost:5984/_up
          readinessProbe:
            exec:
              command:
              - curl
              - http://localhost:5984/_up
          resources:
            ...
```

### Install the service: test-app

A small NodeJS application was created for this demo.
It exposes an HTTP GET endpoint which connects to the CouchDB database.
The manifests are very similar to the CouchDB versions.
As with CouchDB, note that no Istio specific configuration is required on the manifests.
A service account is created and linked to provide the service with a unique identity in Kubernetes and Istio.

```yaml
---
# Source: test-app-chart/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: test-app-test-app-chart
---
# Source: test-app-chart/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: test-app-test-app-chart
  ...
---
# Source: test-app-chart/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app-test-app-chart
  labels:
   ...
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: test-app-chart
      app.kubernetes.io/instance: test-app
  template:
    metadata:
      labels:
        app.kubernetes.io/name: test-app-chart
        app.kubernetes.io/instance: test-app
    spec:
      serviceAccountName: test-app-test-app-chart
      containers:
        - name: test-app-chart
          image: "test-app:latest"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          livenessProbe:
            ...
          resources:
            ...
```

The current setup is displayed in the following drawing.

<img class="image fit" style="margin:0px auto; max-width: 446px;" src="{{ '/img/2019-04-14-istio-service-mesh-s2s/istio-basic-setup.png' | prepend: site.baseurl }}" alt="Basic setup in Istio" />

### Enabling mutual TLS (mTLS)

Currently the service can connect to the backend just fine. 
TLS is currently not used to communicate between the service. 

The following manifest defines a policy which changes this. 
It is a namespace scoped policy telling Istio that all services in the `with-istio` namespace should **ONLY** accept mTLS connections.
This configuration will be picked up by Pilot and distributed to all Envoy proxies in the `with-istio` namespace.
When this policy is applied, Envoy will drop any requests it gets that don't use mTLS.

```yaml
apiVersion: "authentication.istio.io/v1alpha1"
kind: "Policy"
metadata:
  name: "default"
  namespace: with-istio
spec:
  peers:
  - mtls: 
      mode: STRICT
```

Note, this policy only affects the incoming connections on the Envoy proxy.
When a request would be sent to the test-app service now, it would be rejected with an HTTP 503 error code.
This is shown in the following drawing.

<img class="image fit" style="margin:0px auto; max-width: 408px;" src="{{ '/img/2019-04-14-istio-service-mesh-s2s/istio-mtls-broken.png' | prepend: site.baseurl }}" alt="Broken mTLS drawing" />

Next, the outgoing (client) connections needs to be configured to use mTLS.
This can be done by specifying a destination rule for the services.
A destination rule defines a set of rules that are evaluated for every outgoing request from a proxy.
This rules defines that every proxy in `with-istio` namespace needs to use mutual TLS for every service that ends with `.local`.
By applying this rule, the requests will succeed again on the test-app service.

```yaml
apiVersion: "networking.istio.io/v1alpha3"
kind: "DestinationRule"
metadata:
  name: "default"
  namespace: with-istio
spec:
  host: "*.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

### Enabling Role Based Access Control (RBAC) on the services

Services can now communicate securely over mTLS.
To increase the security even further, RBAC can be added to the services.
RBAC allows for roles to be defined that specify access to specific services in the cluster. 
By attaching these roles to service accounts (which are connected to services) services can be permitted to access specific other services. 
This limits the reach a single service has in the cluster and therefor adheres to the least privileges principle.

The following manifest defines a cluster RBAC configuration.
Such configuration can only exist once in the entire service mesh and it needs to have the name `default`.
The mode `ON_WITH_INCLUSION` specifies that all subjects that are listed in the inclusion section need to have RBAC enabled. 
These subjects can be namespaces and/or specific services.
Specifying the namespace `with-istio` in the inclusion section, enables RBAC for all services in that namespace.
By default the RBAC configuration will reject all requests which don't have the proper access defined with an HTTP 403 error code. 

```yaml
apiVersion: "rbac.istio.io/v1alpha1"
kind: ClusterRbacConfig
metadata:
  name: default
spec:
  mode: 'ON_WITH_INCLUSION'
  inclusion:
    namespaces: ['with-istio']
```

After this RBAC config is applied, requests to the test-app instance will start failing again. 
The test-app currently doesn't have a role attached to its service account that allows it to access the CouchDB database. 
Therefor all requests to the service will be rejected with an HTTP error code of 403.
This is shown in the following drawing.

<img class="image fit" style="margin:0px auto; max-width: 417px;" src="{{ '/img/2019-04-14-istio-service-mesh-s2s/istio-rbac-refused.png' | prepend: site.baseurl }}" alt="RBAC refuses connection" />

The following manifest creates a role that allows access to the CouchDB service for all GET requests on any given path.
Note that the full service name is used in the services specification, this is currently required by Istio.
This is only needed for Istio to identify the traffic, short names can still be used to access the service.
By applying this service role, nothing will change to the requests to the test-app since the role is not yet connected to the service account of the test-app service. 

```yaml
apiVersion: "rbac.istio.io/v1alpha1"
kind: ServiceRole
metadata:
  name: couchdb-role
  namespace: with-istio
spec:
  rules:
  - services: ["couchdb.with-istio.svc.cluster.local"]
    methods: ["GET"]
    paths: ['*']
```

So next, we link the new role to the service account of the test-app service.
This is done through a service role binding.
There are two sections to this binding: the role and the subjects.
The role is the one that was created using the previous manifest. 
The subjects can be any identity known to Istio. 
In the demo scenario, only the service accounts are known.
Istio defines a service account as a user identity. 
As with the service names, the service account reference needs to be the full reference scoped towards the cluster.
This allows services from outside of the namespaces to be specified as well.

```yaml
apiVersion: "rbac.istio.io/v1alpha1"
kind: ServiceRoleBinding
metadata:
  name: bind-test-app-service-couchdb-role
  namespace: with-istio
spec:
  subjects:
  - user: "cluster.local/ns/with-istio/sa/test-app-test-app-chart"
  roleRef:
    kind: ServiceRole
    name: "couchdb-role"
```

After applying the last manifest, requests should again be authorized and allowed to connect to the CouchDB instance.

## Conclusion

This demo showed how Istio can be used to secure communication between services using mTLS.
Moreover it showed how the service mesh level authentication can be used to grant or deny access to services in the mesh.
A role can be connected to a service account to allow access. 
Important to note is that the service mesh only allowes or denies traffic.
It doesn't influence the application level access.

In a nutshell, Istio allows cluster admins to enable secure communication, and strong authentication and authorization mechanisms on their Kubernetes cluster without having to manage all kinds of certificates, usernames and passwords. 
The application developers don't need to adopt their application in order to communicate securely in the cluster, nor do they have to change their deployment configuration to enable the service mesh.

This blogpost only highlighted a portion of the features of Istio. 
Security is only a part of the feature set. 
Istio also allows advanced traffic management, monitoring and logging.
Maybe something for a future blogpost.