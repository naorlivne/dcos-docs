---
post_title: Deployment Lab
nav_title: Deployments and Upgrades
menu_order: 1
---

This Deployment Lab aims at providing an introduction to [DC/OS](https://dcos.io/) service deployments.
It serves as a step-wise guide how to deploy new versions of a DC/OS service without causing downtimes.

The Deployment Lab includes the following sessions:

1. [Rolling Upgrades][1]
1. [Canary Deployments][2]
1. [Blue-Green Deployments][3]


## Preparation

Throughout this lab we will be using [simpleservice](https://github.com/dcos-labs/deployment-lab/tree/master/simpleservice), a
simple test service, allowing us to simulate certain behaviour such as reporting a certain version and health check delays.

If you want to follow along and try out the described steps yourself, here are the prerequisites:

- A running [DC/OS 1.8](https://dcos.io/releases/1.8.4/) cluster with at least one private agent, see also [installation](https://dcos.io/install/) if you don't have one yet.
- The [DC/OS CLI](https://dcos.io/docs/1.8/usage/cli/) installed and configured. 
- The [jq](https://stedolan.github.io/jq/) tool, command-line JSON processor, installed.

Finally, as a preparation you should have a (quick) look at the following docs:

- [health checks](https://mesosphere.github.io/marathon/docs/health-checks.html)
- [deployments](https://mesosphere.github.io/marathon/docs/deployments.html)
- [readiness checks](https://mesosphere.github.io/marathon/docs/readiness-checks.html) (OPTIONAL)
- [load balancing with HAProxy](https://serversforhackers.com/load-balancing-with-haproxy) (OPTIONAL)
- [Marathon Blue-Green deployments](https://mesosphere.github.io/marathon/docs/blue-green-deploy.html)
- [Marathon-LB ZDD](https://github.com/mesosphere/marathon-lb#zero-downtime-deployments)

*NOTE*: This lab is based on the work from [mhausenblas](https://github.com/mhausenblas/zdd-lab).

[1]: /docs/1.9/tutorials/deployments/rolling-upgrades/
[2]: /docs/1.9/tutorials/deployments/canary-deployments/
[3]: /docs/1.9/tutorials/deployments/blue-green-deployments/
