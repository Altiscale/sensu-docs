---
title: ""
linkTitle: "Using Check Aggregates"
description: ""
weight: 350
version: "5.16"
product: "Sensu Go"
platformContent: false
menu:
  sensu-go-5.16:
    parent: guides
---

Sensu allows you to monitor groups of checks or entities via aggregates. In Sensu Go, you construct aggregates using labels. This tutorial explains how to configure an entity aggregate and a check aggregate.

## Entity aggregate

To start, configure an entity.
Add a label to your agent:

```yaml
---
# Sensu agent configuration

##
# agent overview
##

labels:
  server_type: "webservers"

```

This example adds the label `server_type` with the value `"webservers"`. It's useful to know when a number of agents on your webservers stop reporting in rather than just a single node.

 Next, add a similar label to a check.

## Check aggregate

This example check aggregate uses an http metric check for a fictitious app site: 

```yaml
---
type: CheckConfig
api_version: core/v2
metadata:
  name: cpu-check
  namespace: default
  labels:
    app_group: "identity_servers"
spec:
  command: "check-http.rb -u https://identity.{{ .entity.name }}.example.com"
  interval: 10
  publish: true
  handlers:
  - slack
  runtime_assets:
  - sensu-plugins/sensu-plugins-http
  - sensu/sensu-ruby-runtime
  subscriptions:
  - linux

```

While Sensu is capable of monitoring singular entities, it is also capable of monitoring groups of entities (and checks) through the use of aggregates. _______________ . Let's get started.

===

## Entity aggregates

To include a check or entity in an aggregate, you must assign a label or set of labels to it. For this example, imagine you have 20 webservers serving a number of applications. In this scenario, you might not care if a single webserver stops responding, but you would care if 15 of the 20 webservers all stop responding. 

Add a label in your `/etc/sensu/agent.yml`:

```yaml
---
# Sensu agent configuration

##
# agent overview
##
name: "webserver01.example.com"
namespace: "default"
subscriptions:
  - webservers
labels:
  server_role: "webserver"
```

After you add the label, restart your agent to pick up the change in configuration:

`systemctl restart sensu-agent`

Next, configure a check that will identify events with the label you assigned to the entity and ensure that these events are in an OK status. Here's an example:

```yaml
---
api_version: core/v2
type: CheckConfig
metadata:
  namespace: default
  name: webservers-aggregate-check
spec:
  runtime_assets:
  - sensu/sensu-aggregate-check
  command: sensu-aggregate-check --api-user=foo --api-pass=bar --entity-labels='server_role:webserver' --warn-percent=75 --crit-percent=50
  subscriptions:
  - backend
  publish: true
  interval: 30
  handlers:
  - slack
  - pagerduty
  - email
```

The check command uses a username/password combination to access the API and matches events with the label "server_role: webserver". The check will create an event if 75% of the aggregate events are in a `warning` state and if 50% of the aggregate events are in a critical state.

## Check Aggregates

Checks can also comprise aggregates. To continue the scenario, suppose that your webservers are serving various applications on different ports: 80, 8080, and 9000. A standard check grouping might look like this:

```yaml
---
type: CheckConfig
metadata:
  name: check-webapp-80
  namespace: default
spec:
  command: "check-http.rb -u http://webserver01.example.com"
  handlers: 
  - slack
  high_flap_threshold: 0
  interval: 10
  low_flap_threshold: 0
  publish: true
  runtime_assets:
  - sensu-plugins/sensu-plugins-http
  - sensu/sensu-ruby-runtime
  subscriptions:
  - linux
```

```yaml
---
type: CheckConfig
metadata:
  name: check-webapp-8080
  namespace: default
spec:
  command: "check-http.rb -u --port 8080 http://webserver01.example.com"
  handlers: 
  - slack
  high_flap_threshold: 0
  interval: 10
  low_flap_threshold: 0
  publish: true
  runtime_assets:
  - sensu-plugins/sensu-plugins-http
  - sensu/sensu-ruby-runtime
  subscriptions:
  - linux
```

```yaml
---
type: CheckConfig
metadata:
  name: check-webapp-9000
  namespace: default
spec:
  command: "check-http.rb -u --port 9000 http://webserver01.example.com"
  handlers: 
  - slack
  high_flap_threshold: 0
  interval: 10
  low_flap_threshold: 0
  publish: true
  runtime_assets:
  - sensu-plugins/sensu-plugins-http
  - sensu/sensu-ruby-runtime
  subscriptions:
  - linux
```

Three separate checks are monitoring your web application. However, if you want view your webapp's health, these three checks don't do the best job of providing that insight. These checks are isolated from each other, and each check alerts individually. 

Instead, it makes more sense to configure this group of checks as an aggregate because you might not care if a check on an individual host fails, but you will certainly care if a large percentage of the checks are in a warning or critical state across a number of hosts.

To turn these checks into an aggregate, add a label to each of them:

```yaml
---
type: CheckConfig
metadata:
  name: check-webapp-80
  namespace: default
  labels:
    service_type: webapp
spec:
  command: "check-http.rb -u http://webserver01.example.com"
  high_flap_threshold: 0
  interval: 10
  low_flap_threshold: 0
  publish: true
  runtime_assets:
  - sensu-plugins/sensu-plugins-http
  - sensu/sensu-ruby-runtime
  subscriptions:
  - linux
```

```yaml
---
type: CheckConfig
metadata:
  name: check-webapp-8080
  namespace: default
  labels:
    service_type: webapp
spec:
  command: "check-http.rb -u --port 8080 http://webserver01.example.com"
  high_flap_threshold: 0
  interval: 10
  low_flap_threshold: 0
  publish: true
  runtime_assets:
  - sensu-plugins/sensu-plugins-http
  - sensu/sensu-ruby-runtime
  subscriptions:
  - linux
```

```yaml
---
type: CheckConfig
metadata:
  name: check-webapp-9000
  namespace: default
  labels:
    service_type: webapp
spec:
  command: "check-http.rb -u --port 9000 http://webserver01.example.com"
  high_flap_threshold: 0
  interval: 10
  low_flap_threshold: 0
  publish: true
  runtime_assets:
  - sensu-plugins/sensu-plugins-http
  - sensu/sensu-ruby-runtime
  subscriptions:
  - linux
```

You can use the labels as part of an aggregate that gives you more visibility into the health of our webapp. You also removed handlers from the check. If you want to alert on an aggregate, it's better to handle the aggregate instead of handling each individual check.

Now, to check these services as part of a combined aggregate, use a check like this:

```yaml
---
api_version: core/v2
type: CheckConfig
metadata:
  namespace: default
  name: webapp-aggregate-check
spec:
  runtime_assets:
  - sensu/sensu-aggregate-check
  command: sensu-aggregate-check --api-user=foo --api-pass=bar --entity-labels='service_type:webapp' --warn-percent=75 --crit-percent=50
  subscriptions:
  - backend
  publish: true
  interval: 30
  handlers:
  - slack
  - pagerduty
  - email
```

Congratulations! Your aggregate is in place. Here's how it might look in the Sensu web UI:

![TO DO INSERT DASHBOARD PIC HERE][]

<!--LINKS-->
[1]:
[2]:
[3]:
[4]:
[5]:
