# Scaling an app with OCP Metrics 

Tutorials around KEDA and OCP Metrics

## Overview of tutorial

Sometimes you will want to have KEDA scale by some custom metrics available in OCP.  This tutorial will cover how to do that effectively with CMA

## Table of Contents

Tutorial listing

1. [Prereqs](#pre-requisites)
2. [Tutorial Breakouts](#tutorial-steps)
3. [Reference Docs](#reference-documents)

---

## Pre requisites

Please review `pre-req.md` if you wish to follow the steps with a setup cluster.

`ServiceMonitor` - https://github.com/axodevelopment/ServiceMonitor/blob/main/README.md

## Tutorial Steps

Before we start after you have reviewed the prereqs please double check that

`oc get pods -n openshift-keda`

returns pods and that they are all running in a healthy state.

First lets deploy an app that produces prometheus metrics that we want to have KEDA consume to scale off of.

`ref/prom-example-app-v1.yaml` <-- located here

If you are not familiar with how ServiceMonitors work and how to enable them please review in ## Pre Requisites the `ServiceMontitor` section.

Long story sort a ServiceMonitor will scrape a prometheus endpoint periodically.  With this in place KEDA will scrape prometheus endpoint add metrics into OCP metrics and then KEDA can consume those and scale up as needed.

--- Review ---
By now you should have 

- ns1 deployed
- Deployment prometheus-example-app deployed
- Service prometheus-example-app deployed
- ServiceMonitor deployed
- kedacontroller deployed watching namespace ns1


If you have the above completed now we need to create a `ServiceAccount`, `Secrets`, `TriggerAuthentication`, `Role` and `RoleBinding` so we can give access to `ScaledObject` to consume metrics from `OCP`

Since we will be calling via the thanos-querier we will do the following steps:

Create the `ServiceAccount`

```bash
oc create serviceaccount thanos -n ns1
```

You can either apply the secret from `ref/prom-example-thanos-token-secret.yaml` or apply the below

```bash
apiVersion: v1
kind: Secret
metadata:
  name: thanos-token
  annotations:
    kubernetes.io/service-account.name: thanos
type: kubernetes.io/service-account-token
```

Now we need a way for KEDA to authenticate against OCP Metrics we will use `TriggerAuthentication` for this.
For review this contains the secret and crt that were created previously.

```bash
oc apply -f prom-example-trigger-auth-prom.yaml
```

Now we need to create a `Role` and `RoleBinding`

```bash
oc apply -f prom-example-role-v1.yaml
```

```bash
oc apply -f prom-example-rolebinding-v1.yaml
```

--- Review ---
By now you should have 

- ns1 deployed
- Deployment prometheus-example-app deployed
- Service prometheus-example-app deployed
- ServiceMonitor deployed
- kedacontroller deployed watching namespace ns1
- Created the thanos SA
- Created a secret for thanos called thanos-token
- Created a TriggerAuthentication
- Created a Role
- Binded the thanos SA to the Role


Once you have done that we are now able to deploy the ScaledObject and see our pods grow.

---

To have KEDA actually scale an object you need to deploy a `ScaledObject` resource here is the example I am using for CMA:

```bash
  apiVersion: keda.sh/v1alpha1
  kind: ScaledObject
  metadata:
    labels:
      scaledobject.keda.sh/name: keda-scaled
    name: keda-scaled
    namespace: ns1
  spec:
    maxReplicaCount: 10
    minReplicaCount: 1
    pollingInterval: 30
    scaleTargetRef:
      kind: Deployment
      name: prometheus-example-app
    triggers:
    - type: prometheus
      metadata:
        authModes: bearer
        metricName: prometheus-example-app-keda
        namespace: ns1
        query: sum(rate(container_memory_usage_bytes{namespace="ns1"}[3m]))
        serverAddress: https://thanos-querier.openshift-monitoring.svc.cluster.local:9092
        threshold: "2"
      authenticationRef:
        name: keda-trigger-auth-prometheus
```

In here we have

```bash
    maxReplicaCount: 10
    minReplicaCount: 1
    pollingInterval: 30
    scaleTargetRef:
      kind: Deployment
      name: prometheus-example-app
```

This describes that should be scaled, and the min and max replicas.

The scaling factor and is based upon the value given and threshold following loosely *** around the following formula

```bash
desiredReplicas = round(minReplicaCount + scaleFactor * (maxReplicaCount - minReplicaCount))
```

For multiple triggers you would structure it like so:

```bash
...
triggers:
    - type: prometheus
      metadata:
        authModes: bearer
        metricName: prometheus-example-app-keda
        namespace: ns1
        query: sum(rate(container_memory_usage_bytes{namespace="ns1"}[3m]))
        serverAddress: https://thanos-querier.openshift-monitoring.svc.cluster.local:9092
        threshold: "2"
      authenticationRef:
        name: keda-trigger-auth-prometheus

    - type: prometheus
      metadata:
        authModes: bearer
        metricName: kafka-consumer-lag
        namespace: ns1
        query: sum(rate(kafka_consumergroup_lag{namespace="ns1", topic="mytopic", consumergroup="my-group"}[5m]))
        serverAddress: https://thanos-querier.openshift-monitoring.svc.cluster.local:9092
        threshold: "10"
      authenticationRef:
        name: keda-trigger-auth-prometheus
```