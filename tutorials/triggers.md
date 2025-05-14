# Scaling an app with OCP Metrics 

Tutorials around KEDA Triggers

## Overview of tutorial

There are many different triggers with Custom Metrics Autoscaler, we will briefly explore some of these.

## Table of Contents

Tutorial listing

1. [Prereqs](#pre-requisites)
2. [Tutorial Breakouts](#tutorial-steps)
3. [Reference Docs](#reference-documents)

---

## Pre requisites

Please review `pre-req.md` if you wish to follow the steps with a setup cluster.

## Tutorial Steps

| Type               | Ref                    | Description           |
|--------------------|--------------------------------|------------------|
| CPU Trigger | cpu utilization | Scales based on the CPU utilization of the target workload. |
| Memory Trigger | memory utilization | Scales based on the memory utilization of the target workload. |
| Kafka Trigger | kafka | Scales based on Kafka metrics such as consumer lag or queue depth. |
| Cron Trigger | cron | Scales based on a schedule, similar to cron jobs in Kubernetes. This allows periodic scaling at fixed times or intervals.|
| Prometheus Trigger | prometheus | Scales based on a query from a Prometheus server, using Prometheus metrics for autoscaling. |


CPU Trigger

```bash
  triggers:
  - type: cpu
    metricType: Utilization | AverageValue
    metadata:
      value: '60'
  minReplicaCount: 1
  ```


Memory Trigger

```bash
  triggers:
  - type: memory
    metricType: Utilization | AverageValue
    metadata:
      value: '60'
      containerName: api
  ```

KafkaTrigger

```bash
  triggers:
  - type: kafka
    metadata:
      topic: my-topic
      bootstrapServers: my-cluster-kafka-bootstrap.openshift-operators.svc:9092
      consumerGroup: my-group
      lagThreshold: '10'
      activationLagThreshold: '5'
      offsetResetPolicy: latest
      allowIdleConsumers: true
      scaleToZeroOnInvalidOffset: false
      excludePersistentLag: false
      version: '1.0.0'
      partitionLimitation: '1,2,10-20,31'
      tls: enable
  ```

  Cron Trigger

  ```bash
  triggers:
  - type: cron
    metadata:
      timezone: Asia/Kolkata
      start: "0 6 * * *"
      end: "30 18 * * *"
      desiredReplicas: "100"
  ```

  Prometheus Trigger

  ```bash
   triggers:
    - authenticationRef:
        name: keda-trigger-auth-prometheus
      metadata:
        authModes: bearer
        metricName: prometheus-example-app-keda
        namespace: ns1
        query: sum(rate(container_memory_usage_bytes{namespace="ns1"}[3m]))
        serverAddress: https://thanos-querier.openshift-monitoring.svc.cluster.local:9092
        threshold: "2"
      type: prometheus
```