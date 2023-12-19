---
title: 'Lab Tips and troubleshooting'
layout: default
nav_order: 11
---

# A couple of tips when you run this lab

An overview of the tips in this section:

- [Use Codespaces](#use-codespaces)
- [.azcli files will save your day](#azcli-files-will-save-your-day)
- [On error perform these steps](#on-error-perform-these-steps)
- [In case you made a coding error](#in-case-you-made-a-coding-error)
- [Config error](#config-error)
- [Not all steps are running smoothly in the codespace (unfortunately)](#not-all-steps-are-running-smoothly-in-the-codespace-unfortunately)
- [Don't commit your GitHub PAT token](#dont-commit-your-github-pat-token)
- [In case the GitHub PAT really doesn't work for you](#in-case-the-github-pat-really-doesnt-work-for-you)
- [Persisting environment variables in a GitHub Codespace](#persisting-environment-variables-in-a-github-codespace)

## Use Codespaces

The best and easiest way to run this lab is definitely through the use of a codespace. It has all the tools pre-installed for you. All the steps as well have been tested through the codespace that is included in the repo. The second best alternative is using Visual Studio Code locally with the remote containers option.

The least best option is with a local install of all the tooling. You can get unexpected errors when using this option. Try to avoid it if you can. We still provide it as an alternative for people who really can't use the codespace or remote containers.

## .azcli files will save your day

In case you are using Visual Studio Code, you can record your statements in a file with the _.azcli_ extension. This extension in combination with the [Azure CLI Tools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.azurecli) gives you extra capabilities like IntelliSense and directly running a statement from the script file in the terminal window. 

When using this extension you can keep a record of all the steps you executed in an _.azcli_ file and quickly execute these statements through the `Ctrl+'` shortcut. Check out the extension, it will save you time in the lab!

## On error perform these steps

There are a couple of places in the lab where the steps you need to execute include easy to miss steps. In case of any error the default way to recover from the error is:

1. re-check whether you executed each step.

1. Check all YAML indentation.

1. Check whether you saved all the files that have changes.

1. Check the logs of the specific failing microservice.

   ```bash
   az spring app logs --name <service-name> --follow
   ```

### In case you made a coding error

In case you see you made a coding error, you will need to rebuild and redeploy the specific failing microservice.

To rebuild and redeploy a failing microservice:

1. Navigate to the root of the application and rebuild the specific microservice.

   ```bash
   cd ~/workspaces/java-microservices-asa-standard-lab/src
   mvn clean package -DskipTests -rf :spring-petclinic-<microservice-name>
   ```

1.  Redeploy the microservice.

   ```bash
   az spring app deploy --name <service-name> \
       --config-file-patterns <service-name> \
       --artifact-path <service-jar> 
   ```

### Config error

In case you made an error in the config repo. Fixing that error in the config repo and restarting services should be enough to recover:

1. Fix the error in the config repo, save the file, commit and push the changes:

   ```bash
   git add .
   git commit -m 'your commit message'
   git push
   ```

1. Restart any services that depend on the new config.

   ```bash
   az spring app restart <service-name>
   ```

      {: .note }
   > You can also double check the output logs or the log tables for additional error messages related to bad config.

## Not all steps are running smoothly in the codespace (unfortunately)

It might be that some steps do not run smoothly in a codespace on some more locked down environments.

In case you use a subscription that has additional policies that lock down what you are allowed to do in the subscription, this might make some of the steps fail. The currently known failures include:

- Not Authorized on some operations: specifically operations on managed identities and Key Vault might suffer from policy settings on the subscription when you run them from a codespace.

How to recover: re-execute the step in a cloud shell window.

## Don't commit your GitHub PAT token

In Lab 2 you will make use of a GitHub PAT token. In case you are using azcli files to keep track of all your steps, it might be that setting the environment variable for the PAT token is part of your azcli file. Remove the GitHub PAT token value before committing your code to any repository. 

Once you accidentally push the GitHub PAT token, it will immediately get invalidated by GitHub, and hence it will become unusable. This will make your application configuration service fail! In case this accidentally happens to you, you will need to recreate or re-issue the PAT token and reconfigure the application configuration service.

### In case the GitHub PAT really doesn't work for you

Ok, you tried accessing the private config repo through the PAT experience, but unfortunately this keeps on failing for you. We've seen this happen from time to time for some people. No worries, you tried, we are very happy that you did. But it would also be great if you could continue through the lab without this PAT constantly failing on you.

Before executing the below fix, do understand that during the execution of the lab your config repo **will** contain secret values for certain resources of the lab. Make sure your do **not** use any password values that you use anywhere else (you shouldn't do that anyways by the way).

So no worries, make your config repo public and proceed! 

## Persisting environment variables in a GitHub Codespace

In case you are using a codespace for running this lab, your environment variables will be lost if the codespace restarts. For persisting these environment variables, you can either use the [guidance that GitHub provides for this](https://docs.github.com/en/enterprise-cloud@latest/codespaces/developing-in-codespaces/persisting-environment-variables-and-temporary-files). We recommend the [single workspace](https://docs.github.com/en/enterprise-cloud@latest/codespaces/developing-in-codespaces/persisting-environment-variables-and-temporary-files#for-a-single-codespace) approach, since that is the easiest to set up and doesn't require workspace restart.

You can find a [samplebashrc file](https://github.com/Azure-Samples/java-microservices-asa-standard-lab/blob/main/solution/samplebashrc) in this repository. You will need to update a couple of values in this file for your specific situation.

Another approach would be to create a dedicated _.azcli_ file where you keep all environment variables. After a workspace restart, you first rerun all the steps in this file and you are good to go again.

You can find a [sampleENVIRONMENT.azcli file](https://github.com/Azure-Samples/java-microservices-asa-standard-lab/blob/main/solution/sampleENVIRONMENT.azcli) in this repository. You will need to update a couple of values in this file for your specific situation.
