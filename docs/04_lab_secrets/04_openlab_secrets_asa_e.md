---
title: 'Lab 4: Secure secrets using Key Vault'
layout: default
nav_order: 6
has_children: true
---

# Lab 04: Secure application secrets using Key Vault

# Student manual

## Lab scenario

Your team is now running a first version of the `spring-petclinic` microservice application in Azure. However you are concerned that your application secrets are stored directly in configuration code. As a matter of fact, GitHub has been generating notifications informing you about this vulnerability. You want to remediate this issue and implement a secure method of storing application secrets that are part of the database connection string. In this unit, you will step through implementing such method.

## Objectives

After you complete this lab, you will be able to:

- Create an Azure Key Vault instance
- Store your connection string elements as Azure Key Vault secrets
- Create a managed identity for your microservices
- Grant the managed identity permissions to access the Azure Key Vault secrets
- Update application config
- Update, rebuild, and redeploy each app

The below image illustrates the end state you will be building in this lab.

![Lab 4 architecture](../images/asa-openlab-4.png)

## Lab Duration

- **Estimated Time**: 60 minutes

## Instructions

During this lab, you will:

- Create an Azure Key Vault instance
- Store your connection string elements as Azure Key Vault secrets
- Create a managed identity for your microservices
- Grant the managed identity permissions to access the Azure Key Vault secrets
- Update application config
- Update, rebuild, and redeploy each app

   {: .note }
   > The instructions provided in this exercise assume that you successfully completed the previous exercise and are using the same lab environment, including your Git Bash session with the relevant environment variables already set.
