---
title: Kickstarting Red Hat Enterprise Linux (RHEL) systems using a highly customized Kickstart with Red Hat Satellite 6
---
### Introduction
Following my [Red Hat Satellite 6 concept](https://blog.scheib.me/2023/05/30/redhat-satellite-concept.html), I thought it made sense to share my highly customized Kickstart to use with Red Hat Satellite 6.

In a later blog post, I'll further share how I make use of [Template Sync](https://access.redhat.com/documentation/de-de/red_hat_satellite/6.12/html/managing_hosts/synchronizing_templates_repositories_managing-hosts) a functionality, that allows importing the templates and also automatically assigning them to defined Operating Systems.

### Overview

A Kickstart file in Satellite is rendered *before* transmitting it to the host that requested it. But what Satellite makes actually awesome for using with Kickstart is the high degree of flexibility you have when using it. The reason being: metadata. If you followed my earlier blog post, you know we defined *a lot* of different objects, which provide small pieces of information about a system.

We can use these information pieces - or like I call them, metadata - to stich together a highly customized Kickstart that *perfectly* fits for the system we are about to deploy.

Before we dig deep into Kickstart, we need to understand how Satellite makes use of Templates and how we make use of it.


Satellite provides the following Template types:

| Template Type             | Description                                                                                                    |
| :------------------------ | :------------------------------------------------------------------------------------------------------------- |
| Partition Tables          | This template type to provides the possibility to surprise, surprise, create partition templates for our hosts |
| Job Templates             | We can use Job Templates to run Jobs within Satellite                                                          |
| Provisioning Templates    | These are the templates we can use for kickstarting systems                                                    |



