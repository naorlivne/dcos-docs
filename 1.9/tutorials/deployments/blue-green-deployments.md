---
post_title: Blue-Green Deployments
nav_title: Blue-Green Deployments
menu_order: 3
---

Another popular form of ZDD supported by DC/OS is the [Blue-Green deployment](http://martinfowler.com/bliki/BlueGreenDeployment.html). Here, the idea is basically to have two versions of your service (unsurprisingly called `blue` and `green`): let's say that `blue` is the live one, serving production traffic and `green` is the new version to be rolled out. Once all instances of `green` are healthy, a load balancer is reconfigured to cut over from `blue` to `green` and if necessary (to roll back) one can do the same in the reverse direction. 

Essentially, we want the following. We start out with `blue` being active: 

    +----------------+
    |                |
    |                |                +----------+
    |   blue (v0.9)  +------+         |          |
    |                |      |         |          |
    |                |      +---------+          |
    +----------------+                |          |
                                      |          |
                                      |          | <-------------+ clients
                                      |          |
    +----------------+                |          |
    |                |                |          |
    |                |                |          |
    |  green (v1.0)  |                |          |
    |                |                +----------+
    |                |
    +----------------+

And once `green` is healthy, we cut over to it by updating the routing:

    +----------------+
    |                |
    |                |                +----------+
    |   blue (v0.9)  |                |          |
    |                |                |          |
    |                |                |          |
    +----------------+                |          |
                                      |          |
                                      |          | <-------------+ clients
                                      |          |
    +----------------+                |          |
    |                |      +---------+          |
    |                |      |         |          |
    |  green (v1.0)  +------+         |          |
    |                |                +----------+
    |                |
    +----------------+

As a first step, we need a load balancer. For this we install [Marathon-LB](https://dcos.io/docs/1.8/usage/service-discovery/marathon-lb/) (MLB for short) from the Universe:

    $ dcos package install marathon-lb
    We recommend a minimum of 0.5 CPUs and 256 MB of RAM available for the Marathon-LB DCOS Service.
    Continue installing? [yes/no] yes
    Installing Marathon app for package [marathon-lb] version [1.4.1]
    Marathon-lb DC/OS Service has been successfully installed!
    See https://github.com/mesosphere/marathon-lb for documentation.

In its default configuration, just as we did with the `dcos package install` command above, MLB runs on a public agent, acting as an edge router and allows us to expose a DC/OS service to the outside world. The MLB default config looks like the following:

    {
      "marathon-lb": {
        "auto-assign-service-ports": false,
        "bind-http-https": true,
        "cpus": 2,
        "haproxy-group": "external",
        "haproxy-map": true,
        "instances": 1,
        "mem": 1024,
        "minimumHealthCapacity": 0.5,
        "maximumOverCapacity": 0.2,
        "name": "marathon-lb",
        "role": "slave_public",
        "sysctl-params": "net.ipv4.tcp_tw_reuse=1 net.ipv4.tcp_fin_timeout=30 net.ipv4.tcp_max_syn_backlog=10240 net.ipv4.tcp_max_tw_buckets=400000 net.ipv4.tcp_max_orphans=60000 net.core.somaxconn=10000",
        "marathon-uri": "http://master.mesos:8080"
      }
    }

MLB is using [HAProxy](http://www.haproxy.org/) under the hood and gets the information it needs to re-write the mappings from frontends to backends from the Marathon event bus. Once MLB is installed, you need to [locate the public agent](https://dcos.io/docs/1.8/administration/locate-public-agent/) it runs on, let's say `$PUBLIC_AGENT` is the resulting IP. Now, to see the HAProxy MLB has under management in action, visit the URL `http://$PUBLIC_AGENT:9090/haproxy?stats` and you should see something like the following:

![MLB HAProxy idle](img/haproxy-idle.png)

In the following we will walk through a manual sequence how to achieve the Blue-Green deployment, however in practice an automated approach is recommended (and pointed out at the end of this section).

So, let's dive into it. First we set up the `blue` version of `simpleservice` via MLB we're using [blue.json](blue-green/blue.json). In the following is the new section highlighted that has been added to `base-health.json` to make this happen:

    "labels": {
      "HAPROXY_GROUP": "external"
      "HAPROXY_0_PORT": "10080",
      "HAPROXY_0_VHOST": "http://ec2-52-25-126-14.us-west-2.compute.amazonaws.com"
    }

The semantics of the added labels from above is as follows:

- `HAPROXY_GROUP` is set to expose it on the (edge-routing) MLB we installed in the previous step.
- `HAPROXY_0_PORT` defines`10080` as the external, public port we want version `0.9` of `simpleservice` to be available.
- `HAPROXY_0_VHOST` is the virtual host to be used for the edge routing, in my case the FQDN of the public agent, see also the [MLB docs](https://dcos.io/docs/1.8/usage/service-discovery/marathon-lb/usage/).

Note that the labels you specify here actually define [service-level HAProxy configurations](https://github.com/mesosphere/marathon-lb/blob/master/Longhelp.md#templates) under the hood. 

Let's check what's going on in HAProxy now:

![MLB HAProxy blue](img/haproxy-blue.png)

In above HAProxy screen shot we can see the `blue` frontend `zdd_blue_10080` for our service, serving on `52.25.126.14:10099` (with `52.25.126.14` being the IP of my public agent) as well as the `blue`backend `zdd_blue_10080`, corresponding to the four instances DC/OS has launched as requested. To verify the ports we can use Mesos-DNS from within the cluster:

    core@ip-10-0-6-211 ~ $ dig _blue-zdd._tcp.marathon.mesos SRV

    ; <<>> DiG 9.10.2-P4 <<>> _blue-zdd._tcp.marathon.mesos SRV
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 31245
    ;; flags: qr aa rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 4

    ;; QUESTION SECTION:
    ;_blue-zdd._tcp.marathon.mesos. IN  SRV

    ;; ANSWER SECTION:
    _blue-zdd._tcp.marathon.mesos. 60 IN  SRV 0 0 19301 blue-zdd-rrf4y-s2.marathon.mesos.
    _blue-zdd._tcp.marathon.mesos. 60 IN  SRV 0 0 9383 blue-zdd-8sqqy-s2.marathon.mesos.
    _blue-zdd._tcp.marathon.mesos. 60 IN  SRV 0 0 3238 blue-zdd-4hzbx-s2.marathon.mesos.
    _blue-zdd._tcp.marathon.mesos. 60 IN  SRV 0 0 10164 blue-zdd-xu4a3-s2.marathon.mesos.

    ;; ADDITIONAL SECTION:
    blue-zdd-xu4a3-s2.marathon.mesos. 60 IN A 10.0.3.192
    blue-zdd-8sqqy-s2.marathon.mesos. 60 IN A 10.0.3.192
    blue-zdd-rrf4y-s2.marathon.mesos. 60 IN A 10.0.3.192
    blue-zdd-4hzbx-s2.marathon.mesos. 60 IN A 10.0.3.192

    ;; Query time: 1 msec
    ;; SERVER: 198.51.100.1#53(198.51.100.1)
    ;; WHEN: Sat Oct 15 09:15:28 UTC 2016
    ;; MSG SIZE  rcvd: 263

We're now in the position that we can access version `0.9` of `simpleservice` from outside the cluster:

    $ curl http://52.25.126.14:10080/endpoint0
    {"host": "52.25.126.14:10080", "version": "0.9", "result": "all is well"}

Next, we deploy version `1.0` of `simpleservice`, using [green.json](blue-green/green.json). Note that nothing has changed so far in HAProxy (check it out, you'll still see the `blue` frontend and backend), however, we have `green` now available within the cluster:

    core@ip-10-0-6-211 ~ $ dig _green-zdd._tcp.marathon.mesos SRV

    ; <<>> DiG 9.10.2-P4 <<>> _green-zdd._tcp.marathon.mesos SRV
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 32879
    ;; flags: qr aa rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 4

    ;; QUESTION SECTION:
    ;_green-zdd._tcp.marathon.mesos.  IN  SRV

    ;; ANSWER SECTION:
    _green-zdd._tcp.marathon.mesos. 60 IN SRV 0 0 30238 green-zdd-re77j-s2.marathon.mesos.
    _green-zdd._tcp.marathon.mesos. 60 IN SRV 0 0 7077 green-zdd-c8oxq-s2.marathon.mesos.
    _green-zdd._tcp.marathon.mesos. 60 IN SRV 0 0 3409 green-zdd-657om-s2.marathon.mesos.
    _green-zdd._tcp.marathon.mesos. 60 IN SRV 0 0 19658 green-zdd-w5mkc-s2.marathon.mesos.

    ;; ADDITIONAL SECTION:
    green-zdd-re77j-s2.marathon.mesos. 60 IN A  10.0.3.192
    green-zdd-657om-s2.marathon.mesos. 60 IN A  10.0.3.192
    green-zdd-c8oxq-s2.marathon.mesos. 60 IN A  10.0.3.192
    green-zdd-w5mkc-s2.marathon.mesos. 60 IN A  10.0.3.192

    ;; Query time: 1 msec
    ;; SERVER: 198.51.100.1#53(198.51.100.1)
    ;; WHEN: Sat Oct 15 09:19:49 UTC 2016
    ;; MSG SIZE  rcvd: 268

So we can test `green` cluster-internally, for example using the following command (executed from the Master, here): 
    
    core@ip-10-0-6-211 ~ $ curl green-zdd.marathon.mesos:7077/endpoint0
    {"host": "green-zdd.marathon.mesos:7077", "version": "1.0", "result": "all is well"}

Now let's say we're satisfied with `green`, all instances are healthy so we update it with below snippet, effectively exposing it via MLB, while simultaneously scaling back `blue` to `0` instances:

    "labels": {
      "HAPROXY_GROUP": "external",
      "HAPROXY_0_PORT": "10080",
      "HAPROXY_0_VHOST": "http://ec2-52-25-126-14.us-west-2.compute.amazonaws.com"
    }

As a result `green` should be available via MLB, so let's check what's going on in HAProxy now:

![MLB HAProxy green](img/haproxy-green.png)

Once we're done scaling down `blue` we want to verify if we can access version `1.0` of `simpleservice` from outside the cluster:

    $ curl http://52.25.126.14:10080/endpoint0
    {"host": "52.25.126.14:10080", "version": "1.0", "result": "all is well"}

And indeed we can. Since the exact mechanics of the deployment orchestration are rather complex, I recommend using [zdd.py](https://github.com/mesosphere/marathon-lb#zero-downtime-deployments) a script that makes respective API calls to the DC/OS System Marathon as well as takes care of gracefully terminating instances using the HAProxy stats endpoint.
