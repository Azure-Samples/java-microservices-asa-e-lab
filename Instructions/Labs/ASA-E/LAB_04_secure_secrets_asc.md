---
lab:
    Title: 'Challenge 04: Secure application secrets using Key Vault'
    Learn module: 'Learn module 4: Secure application secrets using Key Vaults'
---

# Challenge 04: Secure application secrets using Key Vault

# Student manual

## Challenge scenario

Your team is now running a first version of the `spring-petclinic` microservice application in Azure. However you are concerned that your application secrets are stored directly in configuration code. As a matter of fact, GitHub has been generating notifications informing you about this vulnerability. You want to remediate this issue and implement a secure method of storing application secrets that are part of the database connection string. In this unit, you will step through implementing such method.

## Objectives

After you complete this challenge, you will be able to:

- Create an Azure Key Vault instance
- Store your connection string elements as Azure Key Vault secrets
- Create a managed identity for your microservices
- Grant the managed identity permissions to access the Azure Key Vault secrets
- Update application config
- Update, rebuild, and redeploy each app

The below image illustrates the end state you will be building in this challenge.

![Challenge 4 architecture](./images/asa-openlab-4.png)

## Lab Duration

- **Estimated Time**: 60 minutes

## Instructions

During this challenge, you will:

- Create an Azure Key Vault instance
- Store your connection string elements as Azure Key Vault secrets
- Create a managed identity for your microservices
- Grant the managed identity permissions to access the Azure Key Vault secrets
- Update application config
- Update, rebuild, and redeploy each app

   > **Note**: The instructions provided in this exercise assume that you successfully completed the previous exercise and are using the same lab environment, including your Git Bash session with the relevant environment variables already set.

### Create an Azure Key Vault instance

You will start by creating an Azure Key Vault instance that will host your application secrets. You can use the following guidance to perform this task:

- [Create Key Vault](https://learn.microsoft.com/azure/spring-apps/tutorial-managed-identities-key-vault?tabs=system-assigned-managed-identity#set-up-your-key-vault).

<details>
<summary>hint</summary>
<br/>

1. From the Git Bash prompt, run the following command to create an Azure Key Vault instance. Note that the name of the service should be globally unique, so adjust it accordingly in case the randomly generated name is already in use. Keep in mind that the name can contain only lowercase letters, numbers and hyphens. The `$LOCATION` and `$RESOURCE_GROUP` variables contain the name of the Azure region and the resource group into which you deployed the Azure Spring Apps service in the previous exercise of this lab.

   ```bash
   KEYVAULT_NAME=kv-$APPNAME-$UNIQUEID
   az keyvault create \
       --name $KEYVAULT_NAME \
       --resource-group $RESOURCE_GROUP \
       --location $LOCATION \
       --sku standard
   ```

   > **Note**: Wait for the provisioning to complete. This might take about 2 minutes.

</details>

### Store your connection string elements as Azure Key Vault secrets

Now that your Key Vault provisioning is completed, you need to add to it a secret containing the connection string to the database hosted by Azure Database for MySQL Single Server. You can use the following guidance to perform this task:

- [Add a secret to Key Vault](https://learn.microsoft.com/azure/spring-apps/tutorial-managed-identities-key-vault?tabs=user-assigned-managed-identity#set-up-your-key-vault).

These secrets should be called `SPRING-DATASOURCE-URL`, `SPRING-DATASOURCE-USERNAME` and `SPRING-DATASOURCE-PASSWORD`.

<details>
<summary>hint</summary>
<br/>

1. Add the url, username and password of the Azure Database for MySQL Flexible Server admin account as secrets to your Key Vault by running the following commands from the Git Bash prompt:

   ```bash
    az keyvault secret set \
        --name SPRING-DATASOURCE-URL \
        --value "jdbc:mysql://$MYSQL_SERVER_NAME.mysql.database.azure.com:3306/$DATABASE_NAME?useSSL=true&serverTimezone=UTC" \
        --vault-name $KEYVAULT_NAME

   az keyvault secret set \
       --name SPRING-DATASOURCE-USERNAME \
       --value $MYSQL_ADMIN_USERNAME \
       --vault-name $KEYVAULT_NAME

   az keyvault secret set \
       --name SPRING-DATASOURCE-PASSWORD \
       --value $MYSQL_ADMIN_PASSWORD \
       --vault-name $KEYVAULT_NAME
   ```

</details>

### Create a managed identity for your microservices

The apps deployed as the Spring Petclinic microservices will connect to the newly created Key Vault using a managed identity. The process of creating a managed identity will automatically create an Azure Active Directory service principal for your application. Managed identities minimize the overhead associated with managing service principals, since their secrets used for authentication are automatically rotated. In this lab you will make use of user assigned managed identities. These allow you to reuse these identities in case you need to recreate your apps. You can use the following guidance to determine how to assign a managed identity to a Spring Apps service application:

- [Manage user-assigned managed identities for an application in Azure Spring Apps](https://learn.microsoft.com/azure/spring-apps/how-to-manage-user-assigned-managed-identities?tabs=azure-cli&pivots=sc-enterprise).

The following three apps of your application use the database hosted by the Azure Database for MySQL Flexible Server instance, so they will need to be assigned a managed identity:

- `customers-service`
- `vets-service`
- `visits-service`

<details>
<summary>hint</summary>
<br/>

1. Create and assign an identity to each of the three apps by running the following commands from Git Bash shell:

   ```bash
   CUSTOMERS_SERVICE_ID=$(az identity create -g $RESOURCE_GROUP -n customers-svc-uid --query id -o tsv)

    az spring app identity assign \
        --resource-group $RESOURCE_GROUP \
        --name $CUSTOMERS_SERVICE \
        --user-assigned $CUSTOMERS_SERVICE_ID
    
    VISITS_SERVICE_ID=$(az identity create -g $RESOURCE_GROUP -n visits-svc-uid --query id -o tsv)

    az spring app identity assign \
        --resource-group $RESOURCE_GROUP \
        --name $VISITS_SERVICE \
        --user-assigned $VISITS_SERVICE_ID
    
    VETS_SERVICE_ID=$(az identity create -g $RESOURCE_GROUP -n vets-svc-uid --query id -o tsv)

    az spring app identity assign \
        --resource-group $RESOURCE_GROUP \
        --name $VETS_SERVICE \
        --user-assigned $VETS_SERVICE_ID
   ```

    > **Note**: Wait for the operations to complete. This might take about 3 minutes each.

</details>

### Grant the managed identity permissions to access the Azure Key Vault secrets

By now, you have created a managed identity for the `customers-service`, `vets-service` and `visits-service`. In this step, you need to grant these 3 managed identities access to the secrets you added to the Azure Key Vault instance. To accomplish this, you can use the following the guidance: [Grant your app access to Key Vault](https://learn.microsoft.com/azure/spring-apps/tutorial-managed-identities-key-vault?tabs=user-assigned-managed-identity#grant-your-app-access-to-key-vault).

The following three apps of your application use the database hosted by the Azure Database for MySQL Single Server instance, so their managed identities will need to be granted permissions to access the secrets:

- `customers-service`
- `vets-service`
- `visits-service`

<details>
<summary>hint</summary>
<br/>

1. Grant the `get` and `list` secrets permissions in the Azure Key Vault instance to each Spring Apps application's managed identity by using Azure Key Vault access policy:

   ```bash
   CUSTOMERS_SERVICE_UID=$(az identity show -g $RESOURCE_GROUP -n customers-svc-uid --query principalId -o tsv)
   VISITS_SERVICE_UID=$(az identity show -g $RESOURCE_GROUP -n visits-svc-uid --query principalId -o tsv)
   VETS_SERVICE_UID=$(az identity show -g $RESOURCE_GROUP -n vets-svc-uid --query principalId -o tsv)
   
   az keyvault set-policy \
       --name $KEYVAULT_NAME \
       --resource-group $RESOURCE_GROUP \
       --secret-permissions get list  \
       --object-id $CUSTOMERS_SERVICE_UID
   
   az keyvault set-policy \
       --name $KEYVAULT_NAME \
       --resource-group $RESOURCE_GROUP \
       --secret-permissions get list  \
       --object-id $VETS_SERVICE_UID
   
   az keyvault set-policy \
       --name $KEYVAULT_NAME \
       --resource-group $RESOURCE_GROUP \
       --secret-permissions get list  \
       --object-id $VISITS_SERVICE_UID
   ```

</details>

### Update application config

You now have all relevant components in place to switch to the secrets stored in Azure Key Vault and remove them from your config repo. To complete your configuration, you now need to set the config repository to reference the Azure Key Vault instance. You also need to update the **pom.xml** file to ensure that the visits, vets and customers services use the `com.azure.spring:spring-cloud-azure-starter-keyvault-secrets` dependency. You can use the following guidance to accomplish this task:

[Spring Cloud Azure Starter Key Vault Secrets](https://github.com/Azure/azure-sdk-for-java/blob/main/sdk/spring/README.md)
[Build a sample Spring Boot app with Spring Boot starter](https://learn.microsoft.com/azure/spring-apps/tutorial-managed-identities-key-vault?tabs=system-assigned-managed-identity#build-a-sample-spring-boot-app-with-spring-boot-starter)

<details>
<summary>hint</summary>
<br/>

1. From the Git Bash window, in the config repository you cloned locally, use your favorite text editor to open the `application.yml` file. Replace the contents of this file with the contents of this [application.yml](../../../config/04_application.yml) file. This file contains the following changes:

    * The spring.datasource properties are no longer there. These are now in your Key Vault and are no longer needed in the application.yml file.
    * Line 25 to 32 contain new config for your Key Vault. Make sure you replace the `<your-kv-name>` placeholder on line 31 with the name of your Key Vault.
    
1. Save the file and commit and push these changes to your remote config repository.

   ```bash
   git add .
   git commit -m 'added key vault'
   git push
   ```

### Update, rebuild, and redeploy each app

1. From the Git Bash window, in the `Deploying-and-Running-Java-Applications-in-Azure-Spring-Apps` repository you cloned locally, use your favorite text editor to open the `pom.xml` files of the customers, visits and vets services (within the `src/spring-petclinic-customers-service`, `src/spring-petclinic-visits-service`, and `src/spring-petclinic-vets-service` directories). For each, add the following dependencies (within the `<dependencies>...</dependencies>` section) and save the change.

   ```xml
           <dependency>
              <groupId>com.azure.spring</groupId>
              <artifactId>spring-cloud-azure-starter-keyvault-secrets</artifactId>
           </dependency>
   ```

1. From the Git Bash window, in the `spring-petclinic-microservices` repository you cloned locally, use your favorite text editor to open the `pom.xml` file in the `src` directory of the cloned repo. Add to the file a dependency to `com.azure.spring`. This should be added within the `<dependencyManagement><dependencies></dependencies></dependencyManagement>` section.

   ```xml
       <dependencyManagement>
           <dependencies>
               //... existing dependencies

               <dependency>
                   <groupId>com.azure.spring</groupId>
                   <artifactId>spring-cloud-azure-dependencies</artifactId>
                   <version>${version.spring.cloud.azure}</version>
                   <type>pom</type>
                   <scope>import</scope>
               </dependency>

           </dependencies>
       </dependencyManagement>
   ```

1. In the same file, add a property for `version.spring.cloud.azure`. This should be added within the `<properties></properties>` section.

   ```xml
   <version.spring.cloud.azure>5.2.0</version.spring.cloud.azure>
   ```
    
1. Save the changes to the `pom.xml` file and close it.

1. Rebuild the services by running the following command in the root directory of the application.

   ```bash
   cd ~/workspaces/Deploying-and-Running-Java-Applications-in-Azure-Spring-Apps/src
   mvn clean package -DskipTests
   ```

1. Verify that the build succeeds by reviewing the output of the `mvn clean package -DskipTests` command, which should have the following format:

   ```bash
   [INFO] ------------------------------------------------------------------------
   [INFO] Reactor Summary for spring-petclinic-microservices 3.0.2:
   [INFO] 
   [INFO] spring-petclinic-microservices ..................... SUCCESS [  0.249 s]
   [INFO] spring-petclinic-admin-server ...................... SUCCESS [ 16.123 s]
   [INFO] spring-petclinic-customers-service ................. SUCCESS [  6.749 s]
   [INFO] spring-petclinic-vets-service ...................... SUCCESS [  4.845 s]
   [INFO] spring-petclinic-visits-service .................... SUCCESS [  5.063 s]
   [INFO] spring-petclinic-config-server ..................... SUCCESS [  1.777 s]
   [INFO] spring-petclinic-discovery-server .................. SUCCESS [  2.563 s]
   [INFO] spring-petclinic-api-gateway ....................... SUCCESS [ 15.582 s]
   [INFO] ------------------------------------------------------------------------
   [INFO] BUILD SUCCESS
   [INFO] ------------------------------------------------------------------------
   [INFO] Total time:  55.901 s
   [INFO] Finished at: 2023-06-02T14:07:49Z
   [INFO] ------------------------------------------------------------------------
   ```

1. Redeploy the customers, visits and vets services to their respective apps in your Spring Apps service by running the following commands:

   ```bash
   az spring app deploy --name ${CUSTOMERS_SERVICE} \
       --config-file-patterns ${CUSTOMERS_SERVICE} \
       --artifact-path ${CUSTOMERS_SERVICE_JAR} 
   
   az spring app deploy --name ${VETS_SERVICE} \
       --config-file-patterns ${VETS_SERVICE}  \
       --artifact-path ${VETS_SERVICE_JAR}
   
   az spring app deploy --name ${VISITS_SERVICE} \
       --config-file-patterns ${VISITS_SERVICE} \
       --artifact-path ${VISITS_SERVICE_JAR} 
   ```

1. Retest your application through its public endpoint. Ensure that the application is functional, while the connection string secrets are retrieved from Azure Key Vault.

1. In case you don't see data in your application, take a look at the `customers-service` logs to make sure the configuration gets picked up correctly and there are no errors on startup. 

   ```bash
   az spring app logs --name ${CUSTOMERS_SERVICE} --follow 
   ```

   > **Note**: In case you see no errors, you can escape out of the log statement with `Ctrl+C` and you can proceed with the next steps. In case you see errors, review the steps you executed and retry. The [LabTips file](../../../LabTips.md) also contains steps on how to recover from errors.

1. To verify that secrets from Key Vault are picked up, in the Azure Portal, navigate to the page of the Azure Key Vault instance you provisioned. On the Overview page, select the **Monitoring** tab and review the graph representing requests for access to the vault's secrets.

</details>

#### Review

In this lab, you implemented a secure method of storing application secrets that are part of the database connection string of Azure Spring Apps applications.
