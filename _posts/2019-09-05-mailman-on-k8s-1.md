---
title: "mailman on k8s -- idea "
date: 2019-09-05
type: tech
---

## Mailman brief introduction

Mailman is open source software providing discussion channel in email way. from the running system view, it consists of four key components, mailman-core, DB(postgres), mailman-web, deliver-server(exim4), for each component:

- mailman-core：The list deliver launcher.
- mailman-web: Providing the web management.
- deliver-server(exim4): Receiving and sending email
- DB(postgres): Data persistence system.

Still now, mailman can deploy on server and [docker-compose](https://github.com/maxking/docker-mailman), but the scalability is a big challenge with the basic infrastructure isn't easy to scale. so running on k8s is a case that should consider.

## Mailman on k8s

If we are going to running the whole mailman system on k8s, it would be a good start dividing each component into following aspects.

- Container image
- Runtime configuration
- The service for out site user
- Data persistent

For container image, there is nothing much special than docker image. just keep them as the same as [docker-compose](https://github.com/maxking/docker-mailman).

Configuration is much different from docker compose, as you may see, it uses the environment variable to set the value in docker compose. but k8s is should be managed by other way, like ```configmap```, ```secret```.

For the service endpoint part, the generate way is to use ```service``` instead of host port. other more, we should consider whether is it necessary to add load-balancer to add the scalability of each component. e.g. mailman-web, exim4. this depends on the workloads.

 Regarding to the data store, major data will store in DB, deploying a postgres in pod is enough, SSD volume would be plus. one point should think it over, the mailman-core and deliver server should share the configuration space, there are two ways to resolve this, one is run the two components in one pod, so that they can share space easily, another idea is to mount a share volume. different component have its own isolated resource would be much better than mix together. so, here we consider to provision a share ```pv``` and mount into two component container.

The finial component diagram would be:

![]({{ site.url }}/images/2019-09-05-mailman-on-k8s/mailman_k8s_arc.png)

## Other more 

As the mailman configuration contain privacy data, such as the admin name and email, the api access key and secret key. so we have to consider split these information into ```secret```, these secret configuration should be create in k8s by manual and keep them unreachable by others.


Next post I will take a deeper look into the yamls for each components.

