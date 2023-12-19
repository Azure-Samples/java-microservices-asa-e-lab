---
title: 'Lab 3: Enable monitoring'
layout: default
nav_order: 5
has_children: true
---

# Lab 03: Enable monitoring and end-to-end tracing

# Student manual

## Lab scenario

You have created your first Spring Apps service, installed your microservices as apps and exposed them through the `api-gateway`. Now that everything is up and running, it would be helpful to be able to monitor the availability of your apps and detect any errors or exceptions that might occur during their usage. In this lab, you will implement their end-to-end monitoring.

## Objectives

After you complete this lab, you will be able to:

- Live stream the logs from your apps
- Configure Application Insights to receive monitoring information from your apps
- Analyze app-specific monitoring data
- Configure diagnostics settings
- Analyze logs

The below image illustrates the end state you will be building in this lab.

![Lab 3 architecture](../images/asa-openlab-3.png)

## Lab Duration

- **Estimated Time**: 60 minutes

## Instructions

In this lab, you will:

- Live stream the logs from your apps
- Configure Application Insights to receive monitoring information from your apps
- Analyze app-specific monitoring data
- Configure diagnostics settings
- Analyze logs

   {: .note }
   > The instructions provided in this exercise assume that you successfully completed the previous exercise and are using the same lab environment, including your Git Bash session with the relevant environment variables already set.
