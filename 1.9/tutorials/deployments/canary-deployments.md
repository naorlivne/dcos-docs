---
post_title: Canary Deployments
nav_title: Canary Deployments
menu_order: 2
---

The deployments discussed so far all allowed us to do rolling upgrades of a service without causing any downtimes. That is, at any point in time, clients of the `simpleservice` would be served with some version of the service. However, there is one drawback with the deployments so far: clients of the service will potentially see different versions during the deployment in an uncontrolled manner until the point in time all new instances of the service would turn healthy.

In a more realistic setup one would use a load balancer in front of the service instances: on the one hand, this would more evenly distribute the load amongst the service instances and on the other hand it allows us to carry out more advanced ZDD such as the one we're discussing in the following: a [canary deployment](http://martinfowler.com/bliki/CanaryRelease.html). The basic idea behind it is to expose a small fraction of the clients to a new version of the service. Once you're confident it works as expected you roll out the new version to all users. If you take this a step further, for example, by having multiple versions of the service you can do also A/B testing with it.

We now have a look at a canary deployment with DC/OS: we will have 3 instances serving version `0.9` of `simpleservice` and 1 instance serving version `1.0` and want 80% of the traffic to be served by the former and 20% by the latter, the canary. In addition and in contrast to the previous cases want to expose the service to the outside world. That is, `simpleservice` should not only be available to clients within the DC/OS cluster but publicly available, from the wider Internet. So we aim to end up with the following situation:

    +----------+
    |          |
    |   v0.9   +----+
    |          |    |
    +----------+    |
                    |                 +----------+
    +----------+    |                 |          |
    |          |    |             80% |          |
    |   v0.9   +----------------------+          |
    |          |    |                 |          |
    +----------+    |                 |          |
                    |                 |          | <-------------+ clients
    +----------+    |                 |          |
    |          |    |             20% |          |
    |   v0.9   +----+        +--------+          |
    |          |             |        |          |
    +----------+             |        |          |
                             |        +----------+
    +----------+             |
    |          |             |
    |   v1.0   +-------------+
    |          |
    +----------+

Enter [VAMP](http://vamp.io/). VAMP is a platform for managing containerized microservices, supporting canary releases, route updates, metrics collection and service discovery. Note that while VAMP is conveniently available as a [package in the DC/OS Universe](https://github.com/mesosphere/universe/tree/version-3.x/repo/packages/V/vamp/) we will install a more recent version manually in the following to address a dependencies such as Elasticsearch and Logstash better and have a finer-grained control over how we want to use VAMP.

You can either set up VAMP in an automated fashion, using a [DC/OS Jobs-based installer](https://gist.github.com/mhausenblas/bb967625088902874d631eaa502573cb) or manually, carrying out the following steps:

1. Deploy [vamp-es.json](canary/vamp-es.json)
1. Deploy [vamp.json](canary/vamp.json)
1. Deploy [vamp-gateway.json](canary/vamp-gateway.json) 

Deploy above either via the `dcos marathon app add` command or using the DC/OS UI and note that in `vamp-gateway.json` you need to change the `instances` to the number of agents you have in your cluster (find that out via `dcos node`):

    ...
    "instances": 3,
    ...

Now, head over to `http://$PUBLIC_AGENT:8080`, in my case `http://52.25.126.14:8080/` and you should see:

![VAMP idle](img/vamp-idle.png)

Now you can define a VAMP blueprint (also available via [simpleservice-blueprint.yaml](canary/simpleservice-blueprint.yaml)) by pasting it in the VAMP UI under the `Blueprints` tab and hit `Create` or use the VAMP [HTTP API](http://vamp.io/documentation/api-reference/) to submit it:

    ---
    name: simpleservice
    gateways:
      10099: simpleservice/port
    clusters:
      simpleservice:
       gateways:
          routes:
            simpleservice:0.9:
              weight: 80%
            simpleservice:1.0:
              weight: 20%
       services:
          -
            breed:
              name: simpleservice:0.9
              deployable: mesosphere/simpleservice:1.0
              ports:
                port: 0/http
              env:
                SIMPLE_SERVICE_VERSION: "0.9"
            scale:
              cpu: 0.1
              memory: 32MB
              instances: 3
          -
            breed:
              name: simpleservice:1.0
              deployable: mesosphere/simpleservice:1.0
              ports:
                port: 0/http
              env:
                SIMPLE_SERVICE_VERSION: "1.0"
            scale:
              cpu: 0.1
              memory: 32MB
              instances: 1

To use the above blueprint, hit the `Deploy as` button and you should see the following in the `Deployments` tab:

![VAMP simpleservice deployments](img/vamp-deployments.png)

As well as the following under the `Gateways` tab:

![VAMP simpleservice gateways](img/vamp-gateways.png)

We can now check which version clients of `simpleservice` see, using the [canary-check.sh](canary/canary-check.sh) test script as shown in the following (with the public agent, that is, `http://$PUBLIC_AGENT` as the first argument and the number of clients as the optional second argument, `10` in this case):

    $ ./canary-check.sh http://52.25.126.14 10
    Invoking simpleservice: 0
    Invoking simpleservice: 1
    Invoking simpleservice: 2
    Invoking simpleservice: 3
    Invoking simpleservice: 4
    Invoking simpleservice: 5
    Invoking simpleservice: 6
    Invoking simpleservice: 7
    Invoking simpleservice: 8
    Invoking simpleservice: 9
    Out of 10 clients of simpleservice 8 saw version 0.9 and 2 saw version 1.0

As expected, now 80% of the clients see version `0.9` and 20% are served by version `1.0`.

Tip: If you want to simulate more clients here, pass in the number of clients as the second argument, as in `./canary-check.sh http://52.25.126.14 100` to simulate 100 clients, for example.

With this we conclude the canary deployment section and if you want to learn more, you might also want to check out the [VAMP tutorial on this topic](http://vamp.io/documentation/guides/getting-started-tutorial/2-canary-release/).
