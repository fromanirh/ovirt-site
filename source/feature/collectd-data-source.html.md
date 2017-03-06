---
title: Collectd data souce
category: feature
authors: fromani
feature_name: Collectd as data souce
feature_modules: engine,vdsm
feature_status: Design
---

# Collectd data source

## Overview

We want to move the host and VM monitoring from Vdsm to [collectd](http://www.collectd.org).
By offloading this task to collectd, we can greatly simplify Vdsm, getting rid of one complex
and resource-demanding Vdsm component. Furthermore, collectd is a widespread, mature and resource-light
solution well established in the field. However, we add dependency to another third party service,
increasing the overall complexity. To offset the added complexity, Vdsm will move all the monitoring
and reporting to collectd, but will still acquire data directly from the system/libvirt to
support its flows, like live merge or automatic disk resize.


## Owner

Name: [Francesco Romani](http://github.com/fromanirh)
Email: <fromani@redhat.com>


## Detailed description


### How it works

Vdsm will stop most of current monitoring. Vdsm will keep monitoring only the bare minimum of data
it needs internally to support its flows (e.g. libvirt block jobs, disk high water mark).
We will add collectd as dependency of Vdsm. Vdsm will ship the recommended configuration settings for collectd.
Collectd will do the host and vm monitoring.
Newer Engines, starting from 4.2, will consume the data provided by collectd bypassing Vdsm.
Engine will consume the data either accessing collectd directly or via one intermediate data storage (e.g. fluentd).


### Engine Data Consumption

This section is provisional to enable the discussion and to produce the best strategy; once that is achieved, it
will be removed, the content merged in the rest of the document.
Here we discuss all the possible ways Engine could use to consume the data provided by collectd.

#### Vdsm passthrough

Vdsm just polls collectd instead of the host/libvirt, and relays the data to Engine.

Pros:

* No changes needed on Engine

Cons:

* Most inefficient solution performance-wise
* Minimal gains with respect the current approach

Gaps:

* Vdsm: [patches posted](https://gerrit.ovirt.org/#/q/topic:collectd+status:open), testing needed
* Engine: No change needed


#### Engine direct access

Engine joins the [collectd multicast group](https://collectd.org/wiki/index.php/Networking_introduction) to receive
push notifications from collectd

Pros:

* Performance
* Push notifications: Engine will just need to wait for data to come

Cons:

* Configuration: could multicast configuration make life harder for HE/HA?
* Push notifications: Engine currently use polling, which is pull model (TBD: check this is true), potentially invasive
  changes in the config (TBD: check this is true)
* Collectd send rates, not absolute values. We need a protocol to be reliable in presence of restarts/loss of connectivty

Gaps:

* Vdsm: patch to reduce/disable the monitoring: TBD
* Engine: client library; review of the monitoring flow to consume the data (push vs pull model)


#### Intermediate data store

We make use of one intermediate data store, like fluentd, to circumnvent the drawback that collectd send rates.
The metrics team wants to have fluentd on each host anyway, so we can make use of this. Engine will query
the intermediate data store, which will provide durable storage and richer query capabiliyies

Pros:

* Richer query capabilities
* Robust in presence of data loss - Intermediate store will handle gaps.

Cons:

* Needs to decide which intermediate store to use (fluentd?)
* More moving parts (decreased reliability)
* More data transfers involved, albeit on the same host

Gaps:

* Client libraries for Engine - we need to decide what we want to use first.
* Same considerations as per Engine direct access, we could need to review the current Engine flows (push vs pull)
  depending on the intermediate store API.


### Open issues

1. add vdsm-tool configurator support for collectd?

2. Do we want to use one intermediate durable data store?

3. Related to former: collectd send rates. We can mitigate this with a per-host data store, using a reliable link.
