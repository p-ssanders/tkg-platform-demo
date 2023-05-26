#   Making Kubernetes Useful from Scratch

https://miro.com/app/board/uXjVO7K5t8c=/

##  [MIRO] Introduction

I'm Sam Sanders, an Engineer at Tanzu Labs App Services Public Sector.

I've done some version of this talk for our engineering community here at VMware Tanzu Labs a few times, and today I'm going to deliver it here to you.

*(read the slide)*

How many people here have gotten hands-on with kubernetes?

##  [MIRO] Platform Quote

For example, some platforms don’t do much for the developer.

They’re a place where software runs, and can be accessed by users, 
BUT getting software running there, and getting developers access to it is slow and complex, requiring multiple tickets and requests, and lots of waiting.

Other platforms do a lot for the developer. Platforms based on Cloud Foundry or TAP automate a lot of operational tasks for developers, and help get software in front of users through developer self-service.

Today we’re going to see what happens when a fake/pretend enterprise adopts kubernetes and begins to make it useful for developers.

We’ll be using the freely-available Tanzu Kubernetes Grid (TKG) to manage our Kubernetes. We're not going to use upstream (open-source) kubernetes, because although we're here for pain, we don't want that much pain.

And we'll automate some of the work necessary for getting our app teams' software in front of users, and we’ll drive out that automation incrementally, as I, the developer, run into problems trying to use kubernetes.

Luckily my platform team (also me) is willing to take an incremental approach and work with me (the developer) to enhance the platform.

## [MIRO] TKG

What is TKG? [*read the slide*]

So we have a new platform team, and they're ready to provide kubernetes to the fake organization using TKG on AWS, which means they have a “management cluster” running on AWS.

I think of the management cluster as an application running on a VM that exposes an API for CRUDing other kubernetes clusters where business or mission applications run.

TKG comes with a CLI you can use to interact with that API for CRUDing clusters.

The platform team just kicked off this new kubernetes effort, and they’re accepting requests for environments using our enterprise ticketing system.

##  [MIRO] User Story

So I’m a developer, and I’m executing upon this story in my backlog.

*[Read the story]*

My product manager is asking for an acceptance environment, so I need to request one from my platform team. Then I can deploy our application there, and I should be done.

It does look like I’ll need a valid TLS certificate, and somehow a DNS name.

Based on the platform team’s website, it looks like environments are just TKG-managed clusters at the moment.

I begin by requesting an acceptance environment for my team’s application using the enterprise ticketing system.

##  [MIRO - as platform engineer] Step 1a: Create the cluster

I got this request from a team to create an acceptance environment.

I'll create a cluster to represent this acceptance environment, where the team can run their workloads.

I have already prepared a cluster configuration file, with really basic settings.

Mostly just copied from the “AWS Workload Cluster Template” https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.6/vmware-tanzu-kubernetes-grid-16/GUID-tanzu-k8s-clusters-aws.html

This is going to provision AWS machines, with Kubernetes running on them.

I’ve already done this using the TKG CLI because it takes about 5 minutes.

##  [TERMINAL - as platform engineer]: Step 1b: Give Kubeconfig

Now I’ll give the kubeconfig to the developer

    $ tanzu cluster kubeconfig get demo-acceptance-environment --export-file kubeconfig.yaml --admin

We didn't implement RBAC yet, so the developer will just get admin access to their cluster I guess.

##  [TERMINAL - as developer] Step 2: Access the cluster

Someone from platform gave me a kubeconfig.

I've used kubernetes once or twice, so I kind of know that a kubeconfig is how I configure the kubectl cli to access a cluster.

I copied it to my local directory and I'm going to export an environment variable:

    $ export KUBECONFIG=/Users/ssanders/workspace/tkg-platform-demo/kubeconfig.yaml

I guess I can login to the cluster now.

    $ k get ns

##  [MIRO] What happened?

I guess my get namespace command worked because I’m an admin, since I don’t think the platform team implemented RBAC or multi-tenancy yet. They just provision clusters for each team's environment to avoid those issues of multi-tenancy and RBAC.

##  [TERMINAL - as developer] Step 3: Deploy my demo application

So I guess I have to use my kubernetes knowledge to deploy my team’s demo app now?

I’ve already published an image containing my app to our image repository.

I had to submit a ticket to get those pipelines created for my new app.

And for what it’s worth I had to submit another ticket to get a Harbor project created...

But anyway, my image is in in the repo, and I should be able to deploy it.

I guess I’ll start by creating a namespace for this app, called demo. Maybe I'll have other apps that I'll put into this cluster and namespacing might be a good idea.

    $ cd app/demo
    $ k create ns demo
    $ k apply -f deployment.yaml
    $ k get -w deployment demo -n demo
    $ k apply -f service.yaml
    $ k port-forward service/demo 8081:8080 -n demo

##  [MIRO] Problem

The app is running, and we can even access it using port-forwarding, but that isn’t useful for acceptance, especially not by my product manager.

Let me bring this up with the platform team.

##  [TERMINAL - as platform engineer] Step 4a: Ingress

I'm going to enhance the platform to automate ingress routing from the Internet, so developers don’t have to do it, or submit tickets.

Port-forwarding isn't viable, and as a platform engineer I don't want developers solving for ingress on their own (e.g.: by managing load balancers and public IPs).

Luckily, we can use the TKG CLI to CRUD “packages” as well.

These are similar to Helm charts – or software packaged to run on kubernetes.
In our case since we're using TKG, it comes with a repository full of useful packages.

I'm using the built-in `tanzu-standard` package repository distributed with Tanzu Kubernetes Grid.

    $ tanzu package available list

So let’s install an Ingress Controller, Contour, using the Tanzu package, that will manage ingress for the entire cluster with a single load balancer and public IP.

    $ tanzu package available list contour.tanzu.vmware.com
    $ k create ns tanzu-system-ingress
    $ tanzu package install contour --package contour.tanzu.vmware.com --version 1.23.5+vmware.1-tkg.1 --values-file platform/contour/values.yaml -n tanzu-system-ingress

##  [MIRO] What happened?

It’s a bunch of software, specifically:
- Envoy, the proxy server application (like ngninx) https://www.envoyproxy.io/
- Contour, the “controller” application that interacts with the Kubernetes API to learn about changes, and react accordingly.

*DETOUR*
Controller pattern is critical to understanding automation on Kubernetes.

https://kubernetes.io/docs/concepts/architecture/controller/

Contour is a controller, and reacts to Ingress resources by configuring Envoy.
Think literally updating “envoy.conf” with new routes.
But there’s other kinds of controllers that react to other kinds of resources, and we’ll see more of that later.

##  [TERMINAL - as platform engineer] Step 4a: Ingress

    $ k get ns
    $ k get all -n tanzu-system-ingress
    $ curl -m 2 LOAD_BALANCER_URL

##  [TERMINAL - as developer] Step 4b: Ingress

Let’s configure ingress, by simply creating ingress resource that will be detected by the ingress controller (contour).

Tail the logs of the contour pods:

    $ k get pods -l=app=contour -n tanzu-system-ingress
    $ k logs -f $POD_NAME -n tanzu-system-ingress

    $ export KUBECONFIG=/Users/ssanders/workspace/tkg-platform-demo/kubeconfig.yaml
    $ cat app/demo/ingress-v1.yaml
    $ k apply -f app/demo/ingress-v1.yaml

Now we can add a host header to our requests to the load balancer, and get a response.

    $ k get all -n tanzu-system-ingress
    $ curl -H 'Host: demo.tkg.samsanders.dev' LOAD_BALANCER_URL

##  [MIRO] Problem

So we’ve automated routing from the public internet, and that’s better than expecting our PM to get access to our cluster and setup port forwarding on their machines. But PMs use browsers to access web applications, they don’t craft custom requests with Host headers, so we need a DNS name for our app.

Let me ask my platform team about DNS.

##  [Terminal - as platform engineer] Step 5a: DNS

I'm going to enhance kubernetes to automate the synchronization of DNS entries, so developers don’t have to do it, or submit tickets. I want developers to self-service DNS so I don't have to manage tickets or give them access to the DNS server.

Let’s add another package called external-dns to automate the creation of DNS entries.

    $ tanzu package available list
    $ k create ns tanzu-system-service-discovery
    $ k apply -f platform/external-dns/aws-credentials-secret.yaml
    $ cat platform/external-dns/values.yaml
    $ tanzu package install external-dns --package external-dns.tanzu.vmware.com --version 0.12.2+vmware.5-tkg.1 --values-file platform/external-dns/values.yaml -n tanzu-system-service-discovery

##  [MIRO] What happened?

##  [TERMINAL] Step 5b: DNS

    $ k get all -n tanzu-system-service-discovery
    $ k logs POD_NAME -n tanzu-system-service-discovery

We can see the Ingress was processed!

##  [AWS Route 53]

##  [TERMINAL]

    $ curl demo.tkg.samsanders.dev

##  [MIRO] Problem

Just by creating an Ingress, the platform now creates DNS entries for us and routes traffic to our application from the internet. 

But it doesn't work in the browser, because of the .dev TLD, which requires TLS in Chrome because .dev TLDs being on the HSTS pre-load list. https://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security

So we need to connect using TLS, but we didn’t specify anything about TLS in our Ingress, and we don’t have a certificate. So it won’t work.

##  [TERMINAL - as platform engineer] Step 6a: Certificates

Let's enhance the platform to automate the obtaining and installation of TLS certificates using Let’s Encrypt, so developers don’t have to do it, or submit tickets that I have to manage.

We actually already have cert-manager installed, but we need to make to configure it so it can obtain Let's Encrypt certs and manage them automatically.

    $ tanzu package installed list -A
    $ k get ns

The first part of that is giving cert-manager access to AWS so cert-manager can do the LE dance to prove ownership of the domain.

    $ k apply -f platform/cert-manager/aws-credentials-secret.yaml

The second part is creating configuration, or what cert-manager calls a ClusterIssuer, to tell cert-manager how to issue certificates i.e.:

- where to obtain?
- how to prove ownership of the domain?

    $ k apply -f platform/cert-manager/cluster-issuer-staging.yaml
    $ k get clusterissuer lets-encrypt-staging

I'm using LE Staging, because if I make a mistake I don't want to rate limit myself :D

##  [MIRO] What happened?

##  [TERMINAL - as developer] Step 6b: Certificates

Now we can update the Ingress definition so that cert-manager detects it, and reacts.

We can watch the controller logs while we do it

    $ k logs -f pod/cert-manager-POD -n cert-manager

    $ cat app/demo/ingress-v2.yaml
    $ diff app/demo/ingress-v1.yaml app/demo/ingress-v2.yaml
    $ k apply -f app/demo/ingress-v2.yaml
    $ k get certificaterequests -n demo

    $ k get certificate demo-cert -n demo
    $ k get secret -n demo

##  [Chrome]

https://demo.tkg.samsanders.dev

##  [MIRO] What happened?

So we validated the functionality using Let’s Encrypt Staging server. Now let’s update to prod to get a real certificate, or can just stop here.

    $ cat platform/cert-manager/cluster-issuer-prod.yaml
    $ k apply -f platform/cert-manager/cluster-issuer-prod.yaml
    $ diff app/demo/ingress-v2.yaml app/demo/ingress-v3.yaml
    $ k apply -f app/demo/ingress-v3.yaml