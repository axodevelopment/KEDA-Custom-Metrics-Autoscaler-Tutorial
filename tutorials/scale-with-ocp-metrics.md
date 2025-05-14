# Scaling an app with OCP Metrics 

Tutorials around Streams For Apache Kafka running on OCP - How to manage day 2 tasks

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

