---
lab:
    Title: 'Challenge 04b: Connect passwordless'
    Learn module: 'Learn module 4b: Connect passwordless'
---

# Challenge 04b: Connect passwordless

# Student manual

## Challenge scenario

Your team has now stored usernames and passwords in a Key Vault for higher security. However, they learned there is also a way to connect without using passwords to Azure services. In this unit, you will step through implementing passwordless connectivity to the database.

## Objectives

After you complete this challenge, you will be able to:

- Delete secrets from Key Vault
- Create an admin account for the MySQL Flexible Server
- Create service connections from the microservices to the database server
- Update the applications to use passwordless connectivity

The below image illustrates the end state you will be building in this challenge.

![Challenge 4 architecture](./images/asa-openlab-4b.png)

## Lab Duration

- **Estimated Time**: 60 minutes

## Instructions

During this challenge, you will:

- Delete secrets from Key Vault
- Create an admin account for the MySQL Flexible Server
- Create service connections from the microservices to the database server
- Update the applications to use passwordless connectivity

   > **Note**: The instructions provided in this exercise assume that you successfully completed the previous exercise and are using the same lab environment, including your Git Bash session with the relevant environment variables already set.

### Delete secrets from Key Vault

You will start by deleting the existing secrets from the Key Vault. The Key Vault is still there in case other secrets are needed for your apps. For instance for third party services that don't support passwordless connectivity. You can use the following guidance to perform this task:

- [Delete secrets](https://learn.microsoft.com/azure/key-vault/general/key-vault-recovery?tabs=azure-cli#secrets-cli).

<details>
<summary>hint</summary>
<br/>

1. From the Git Bash prompt, run the following command to delete the secrets from your Azure Key Vault instance. The `$KEYVAULT_NAME`, `$LOCATION` and `$RESOURCE_GROUP` variables contain the name of the Key Vault, Azure region and the resource group into which you deployed the Azure Spring Apps service in the previous exercise of this lab.

   ```bash
   az keyvault secret delete --vault-name $KEYVAULT_NAME --name SPRING-DATASOURCE-URL
   az keyvault secret delete --vault-name $KEYVAULT_NAME --name SPRING-DATASOURCE-USERNAME
   az keyvault secret delete --vault-name $KEYVAULT_NAME --name SPRING-DATASOURCE-PASSWORD
   ```

</details>

### Create an admin account for the MySQL Flexible Server

Now that these secrets are deleted from the Key Vault, you will need to create a managed identity to link to the admin account of your MySQL Flexible server. This managed identity will be associated with your own user account in one of the next steps where you create the service connections. You can use the following guidance to perform this task:

- [Set up Azure Active Directory authentication for Azure Database for MySQL - Flexible Server](https://learn.microsoft.com/azure/mysql/flexible-server/how-to-azure-ad).

<details>
<summary>hint</summary>
<br/>

1. Create an admin managed identity by running the following commands from the Git Bash prompt:

   ```bash
   DB_ADMIN_USER_ASSIGNED_IDENTITY_NAME=uid-dbadmin-$APPNAME-$UNIQUEID
   
   ADMIN_IDENTITY_RESOURCE_ID=$(az identity create \
    --name $DB_ADMIN_USER_ASSIGNED_IDENTITY_NAME \
    --resource-group $RESOURCE_GROUP \
    --query id \
    --output tsv)
   ```

</details>

### Create service connections from the microservices to the database server

The apps deployed as the Spring Petclinic microservices will now connect using a service connector to the MySQL Flexible server. A service connector will set up the needed environment variables the service needs to make the connection. You can use the following guidance to create a service connector:

- [Connect an Azure Database for MySQL instance to your application in Azure Spring Apps](https://learn.microsoft.com/azure/spring-apps/how-to-bind-mysql?tabs=Service-Connector).

The following three apps of your application use the database hosted by the Azure Database for MySQL Flexible Server instance, so they will need to be assigned a service connector:

- `customers-service`
- `vets-service`
- `visits-service`

Since each of these apps already has a user assigned managed identity assigned to them, you will make use of this same identity to get access to the database.

<details>
<summary>hint</summary>
<br/>

1. For creating a service connector you will need to add the `serviceconnector-passwordless` extension:

   ```bash
   az extension add --name serviceconnector-passwordless --upgrade
   ```

1. You will also need the `clientId` of each of the user assigned managed identities of your microservices. Store these clientId's in environment variables, by running the following commands from Git Bash shell:

   ```bash
   CUSTOMERS_SERVICE_CID=$(az identity show -g $RESOURCE_GROUP -n customers-svc-uid --query clientId -o tsv)
   VISITS_SERVICE_CID=$(az identity show -g $RESOURCE_GROUP -n visits-svc-uid --query clientId -o tsv)
   VETS_SERVICE_CID=$(az identity show -g $RESOURCE_GROUP -n vets-svc-uid --query clientId -o tsv)
   ```

1. You will also need your subscription ID for creating the service connections:

   ```bash
   SUBID=$(az account show --query id -o tsv)
   ```

1. Create now the service connections for the `customers-service`.

   ```bash
   az spring connection create mysql-flexible \
       --resource-group $RESOURCE_GROUP \
       --service $SPRING_APPS_SERVICE \
       --app $CUSTOMERS_SERVICE \
       --target-resource-group $RESOURCE_GROUP \
       --server $MYSQL_SERVER_NAME \
       --database $DATABASE_NAME \
       --user-identity mysql-identity-id=$ADMIN_IDENTITY_RESOURCE_ID client-id=$CUSTOMERS_SERVICE_CID subs-id=$SUBID
   ```

1. You can test the validity of this new connection with the `validate` command: 

   ```bash
   CONNECTION=$(az spring connection list \
       --resource-group $RESOURCE_GROUP \
       --service $SPRING_APPS_SERVICE \
       --app $CUSTOMERS_SERVICE \
       --query [].id -o tsv)
   
   az spring connection validate \
       --resource-group ${RESOURCE_GROUP} \
       --service ${SPRING_APPS_SERVICE} \
       --app ${CUSTOMERS_SERVICE} \
       --id $CONNECTION
   ```

   The output of this command should show that the connection was made successful.

1. In the same way create the service connections for the `vets-service` and `visits-service`: 

   ```bash
   az spring connection create mysql-flexible \
       --resource-group $RESOURCE_GROUP \
       --service $SPRING_APPS_SERVICE \
       --app $VISITS_SERVICE \
       --target-resource-group $RESOURCE_GROUP \
       --server $MYSQL_SERVER_NAME \
       --database $DATABASE_NAME \
       --user-identity mysql-identity-id=$ADMIN_IDENTITY_RESOURCE_ID client-id=$VISITS_SERVICE_CID subs-id=$SUBID
   
   az spring connection create mysql-flexible \
       --resource-group $RESOURCE_GROUP \
       --service $SPRING_APPS_SERVICE \
       --app $VETS_SERVICE \
       --target-resource-group $RESOURCE_GROUP \
       --server $MYSQL_SERVER_NAME \
       --database $DATABASE_NAME \
       --user-identity mysql-identity-id=$ADMIN_IDENTITY_RESOURCE_ID client-id=$VETS_SERVICE_CID subs-id=$SUBID
   ```

1. In the Azure Portal, navigate to your Spring Apps Service instance. Navigate to `Apps` and open your `customers-service` app. In the `customers-service` app, select the `Service Connector` menu item. Notice in this screen you can see the details of your service connector. Notice that the service connector has all the config values set like `spring.datasource.url`, `spring.datasource.username`, but for instance no `spring.datasource.password`. These values get turned into environment variables at runtime for your app. This is also why you could remove them from the Key Vault. Instead of `spring.datasource.password` it has a `spring.cloud.azure.credential.client-id`, which is the client ID of your managed identity. It also defines 2 additional variables `spring.datasource.azure.passwordless-enabled` and `spring.cloud.azure.credential.managed-identity-enabled` for enabling the passwordless connectivity.

</details>

### Update the applications to use passwordless connectivity

By now all setup on the spring apps service side is done. You will still need to update your microservices to make use of the new passwordless capabilities. To accomplish this, you can use the following the guidance: [Connect an Azure Database for MySQL instance to your application in Azure Spring Apps](https://learn.microsoft.com/azure/spring-apps/how-to-bind-mysql?tabs=Service-Connector).

The following three apps of your application use the database hosted by the Azure Database for MySQL Single Server instance, so they will need to have their code updated:

- `customers-service`
- `vets-service`
- `visits-service`

<details>
<summary>hint</summary>
<br/>

1. From the Git Bash window, in the `Deploying-and-Running-Java-Applications-in-Azure-Spring-Apps` repository you cloned locally, use your favorite text editor to open the `pom.xml` files of the customers, visits and vets services (within the `src/spring-petclinic-customers-service`, `src/spring-petclinic-visits-service`, and `src/spring-petclinic-vets-service` directories). For each, replace the `mysql-connector-j` dependency (within the `<dependencies>...</dependencies>` section)  with the `spring-cloud-azure-starter-jdbc-mysql` dependency and save the file:

   ```xml
        <!-- Replace this dependency -->
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope>
        </dependency>
   ```

   ```xml
        <!-- by this dependency -->
        <dependency>
            <groupId>com.azure.spring</groupId>
            <artifactId>spring-cloud-azure-starter-jdbc-mysql</artifactId>
        </dependency>
   ```

1. Rebuild the microservices by running the below command in the git bash window:

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

1. Once the build is complete, redeploy each of the apps.

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

</details>

#### Review

In this lab, you implemented a secure method of connecting to the database by using passwordless service connections.
