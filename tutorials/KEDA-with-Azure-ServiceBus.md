# Scaling an app with KEDA (NOT OCP CMA) AND ServiceBus

Tutorials around KEDA and Azure ServiceBus

## Overview of tutorial

This is not supported by Red Hat but KEDA (upstream) does  have support for Azure Products including Azure Service Bus.

## Table of Contents

Tutorial listing

1. [Prereqs](#pre-requisites)
2. [Tutorial Breakouts](#tutorial-steps)
3. [Reference Docs](#reference-documents)

---

## Pre requisites

- Azure subscription: Sign up if you don’t have one.
- OpenShift CLI (oc) installed.
- KEDA operator installed on OpenShift.
- Azure CLI installed to interact with Azure resources.
- An OpenShift cluster running and accessible

This is heavily based upon https://learn.microsoft.com/en-us/azure/aks/keda-workload-identity

KEDA Scaler: https://keda.sh/docs/2.17/scalers/azure-service-bus/


(Webdev)
Configured with Grafana
https://github.com/webdevops/azure-metrics-exporter



## Tutorial Steps

First we need to install KEDA

```bash
oc apply -f https://github.com/kedacore/keda/releases/download/v2.17.0/keda-2.17.0.yaml
```

Confirm KEDA is deployed and running

```bash
oc get pods -n keda
```

Create a namespace in Service Bus

```bash
SB_NAME=<service-bus-name>
SB_HOSTNAME="${SB_NAME}.servicebus.windows.net"
az servicebus namespace create \
    --name $SB_NAME \
    --resource-group <resource-group-name> \
    --disable-local-auth
```

Create a queue in Service Bus namespace creatd above

```bash
SB_QUEUE_NAME=<service-bus-queue-name>
az servicebus queue create \
    --name $SB_QUEUE_NAME \
    --namespace $SB_NAME \
    --resource-group <resource-group-name>
```

Create Managed Identity and Workload Identity

```bash
MI_NAME=<managed-identity-name>
MI_CLIENT_ID=$(az identity create \
    --name $MI_NAME \
    --resource-group <resource-group-name> \
    --query "clientId" \
    --output tsv)
```


Push data to Azure Service Bus

```bash
oc apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: myproducer
spec:
  template:
    metadata:
      labels:
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: $MI_NAME
      containers:
      - image: ghcr.io/azure-samples/aks-app-samples/servicebusdemo:latest
        name: myproducer
        env:
        - name: OPERATION_MODE
          value: "producer"
        - name: MESSAGE_COUNT
          value: "100"
        - name: AZURE_SERVICEBUS_QUEUE_NAME
          value: $SB_QUEUE_NAME
        - name: AZURE_SERVICEBUS_HOSTNAME
          value: $SB_HOSTNAME
      restartPolicy: Never
EOF
```

```bash
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: myconsumer-scaledjob
spec:
  jobTargetRef:
    template:
      metadata:
        labels:
          azure.workload.identity/use: "true"
      spec:
        serviceAccountName: $MI_NAME
        containers:
        - image: ghcr.io/azure-samples/aks-app-samples/servicebusdemo:latest
          name: myconsumer
          env:
          - name: OPERATION_MODE
            value: "consumer"
          - name: MESSAGE_COUNT
            value: "10"
          - name: AZURE_SERVICEBUS_QUEUE_NAME
            value: $SB_QUEUE_NAME
          - name: AZURE_SERVICEBUS_HOSTNAME
            value: $SB_HOSTNAME
        restartPolicy: Never
  triggers:
  - type: azure-servicebus          #<-- Note
    metadata:
      queueName: $SB_QUEUE_NAME #<-- queue name
      namespace: $SB_NAME       #<-- ns
      messageCount: "10"        #<-- messageCount
    authenticationRef:
      name: azure-servicebus-auth
```









https://learn.microsoft.com/en-us/azure/aks/keda-workload-identity