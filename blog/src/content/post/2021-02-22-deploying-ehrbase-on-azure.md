---
title: Deploying EHRbase on Azure
#author: Jake
date: '2021-02-22'
categories:
  - DevOps
tags:
  - Cloud
  - EHRbase
slug: deploying-ehrbase-on-azure
---

# Deploying EHRbase on Azure

I wanted to learn how to use Azure and deploy applications on a modern cloud platform for a long time. Additionally, the [openEHR](https://www.openehr.org/) community asked about such capabilities regarding our open source Java Clinical Data Repository [EHRbase](https://github.com/ehrbase/ehrbase). So I finally tried do look into those two things, at the same time. After roughly two days I managed to get EHRbase working on Azure - and learned a lot in that time!

After my first Azure learning session I realized with containerized or even VM solutions it would be easily possible to deploy EHRbase on Azure. But EHRbase is supporting docker deployments already, so this couldn't be what the community was looking for. Additionally, I would like to leverage the full potential of the cloud platform, which requires going full PaaS. In consequence, this is what this blog post documents. I tried to make use of managed Azure services as much as possible, resulting in a setup composed of one *App Service* and one *Azure Database for PostgreSQL* instance. If you're looking for something else please have a look at the [EHRbase documentation](https://ehrbase.readthedocs.io/en/latest/) or contact us.

Disclaimer: As per my personal experience there's about no chance the following descriptions will match actual best-practices. Naturally, as I documented my first time working with Azure and trying to setup a non-default Java application on it, there are very likely rookie mistakes included. Please let me know, if you have comments.


## General

During my first steps in the world of Azure I really liked the direct approach of the Azure learning modules (for instance https://docs.microsoft.com/en-us/learn/modules/deploy-java-spring-boot-app-service-mysql/3-exercise-build). So I borrowed many ideas and commands from it, already staring with the following.

The first step is to setup general variables to use during the whole deployment process:

``` bash
AZ_RESOURCE_GROUP=ehrbase-test
AZ_DATABASE_NAME=ehrbasedbUniqueName
AZ_LOCATION=westeurope
AZ_PSQL_USERNAME=ehrbasedbadmin
AZ_PSQL_PASSWORD=SuperSecurePassword
AZ_LOCAL_IP_ADDRESS=LocalIP (check http://whatismyip.akamai.com/)
```

See explanation on top [here](https://docs.microsoft.com/en-us/learn/modules/deploy-java-spring-boot-app-service-mysql/3-exercise-build).

## Azure EHRbase Database

Setup a Azure Postgres resource with Postgres 11 and otherwise more or less default values:

``` bash
az postgres server create \
    --resource-group $AZ_RESOURCE_GROUP \
    --name $AZ_DATABASE_NAME \
    --location $AZ_LOCATION \
    --sku-name B_Gen5_1 \
    --storage-size 5120 \
    --admin-user $AZ_PSQL_USERNAME \
    --admin-password $AZ_PSQL_PASSWORD \
	--version 11.0 \
    | jq
```

### Firewall

To properly use the DB some firewall adjustments are necessary. Allow your local IP, so locally executed DB provision is possible and your local running EHRbase can connect to the remote DB (latter isn't technically necessary but a great intermediate step anyway):

``` bash
az postgres server firewall-rule create \
    --resource-group $AZ_RESOURCE_GROUP \
    --name $AZ_DATABASE_NAME-database-allow-local-ip \
    --server $AZ_DATABASE_NAME \
    --start-ip-address $AZ_LOCAL_IP_ADDRESS \
    --end-ip-address $AZ_LOCAL_IP_ADDRESS \
    | jq
```

Allow all Azure IPs to let the final remote EHRbase connect to the DB:

``` bash
az postgres server firewall-rule create \
    --resource-group $AZ_RESOURCE_GROUP \
    --name allAzureIPs \
    --server $AZ_DATABASE_NAME \
    --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0 \
    | jq
```

### DB Provisioning

EHRbase's DB provisioning isn't made for cloud deployments. Actually, our local deployment assumes a Postgres instance with extensions like temporal table and jsquery, which aren't available on typical cloud managed Postgres instances. The following steps will prepare and provision the DB in a cloud-ready way ([for more details see](https://github.com/ehrbase/ehrbase/tree/develop/base/db-setup)). Let's start with creating an EHRbase user in the DB in two steps.

First, connect to the DB (using `$AZ_PSQL_PASSWORD`):

``` bash
psql --host=$AZ_DATABASE_NAME.postgres.database.azure.com --port=5432 --username=$AZ_PSQL_USERNAME@$AZ_DATABASE_NAME --dbname=postgres
```

Secondly, create an "ehrbase" role and add our admin user to it, both from within psql:

``` sql
CREATE ROLE ehrbase;
GRANT ehrbase TO ehrbasedbadmin;
\q             # to quit
```

Finally run the initial provision. From within the EHRbase repository path (clone it if not done already) execute (using `$AZ_PSQL_PASSWORD`):

``` bash
psql --host=$AZ_DATABASE_NAME.postgres.database.azure.com --port=5432 --username=$AZ_PSQL_USERNAME@$AZ_DATABASE_NAME --dbname=postgres < base/db-setup/cloud-db-setup.sql

```

## Local EHRbase Application

The first milestone is to run EHRbase locally, but connected to the remote DB. To do so, EHRbase needs to be configured to use the remote DB. The flyway integration will then take care of the rest, when EHRbase is packaged. Change host, user and pass in pom.xml accordingly to previously set values:

Note: It is necessary to manually add the real values here. Environment variables won't work at this part of the pom.xml.

``` xml
<database.user>${env.AZ_PSQL_USERNAME}@{$env.AZ_DATABASE_NAME}</database.user>
<database.pass>${env.AZ_PSQL_PASSWORD}</database.pass>
<database.host>${env.AZ_DATABASE_NAME}.postgres.database.azure.com</database.host>
<database.hostname>${env.AZ_DATABASE_NAME}</database.hostname>
```

With the changes from above the application should now be packaged successfully:

``` bash
mvn clean package
```

To run EHRbase the same information need to be set a bit differently for the runtime - here directly combined with starting EHRbase itself:

``` bash
SPRING_PROFILES_ACTIVE=cloud \
SPRING_DATASOURCE_URL="jdbc:postgresql://$AZ_DATABASE_NAME.postgres.database.azure.com:5432/ehrbase" \
SPRING_DATASOURCE_USERNAME=$AZ_PSQL_USERNAME@$AZ_DATABASE_NAME \
SPRING_DATASOURCE_PASSWORD=$AZ_PSQL_PASSWORD \
java -jar application/target/application-0.14.0.jar
```

Now EHRbase should run locally, while the DB is on Azure. In another terminal window, test your setup with:

```
curl --header "PREFER: return=representation" \
	--request POST \
    http://localhost:8080/ehrbase/rest/openehr/v1/ehr | jq
```

(Note: `jq` is not required, it just presents the JSON response more readable.)

The response should look like the following:

``` json
{
  "system_id": {
    "_type": "HIER_OBJECT_ID",
    "value": "3ebc3f08-5ddd-4329-870e-6593f90981c8"
  },
  "ehr_id": {
    "_type": "HIER_OBJECT_ID",
    "value": "ae2203ab-3801-4836-b8a0-c57b3ab4346e"
  },
  "ehr_status": {
    "_type": "EHR_STATUS",
    "name": {
      "_type": "DV_TEXT",
      "value": "EHR Status"
    },
    "subject": {
      "_type": "PARTY_SELF"
    },
    "archetype_node_id": "openEHR-EHR-EHR_STATUS.generic.v1",
    "uid": {
      "_type": "HIER_OBJECT_ID",
      "value": "cce36826-e344-48a0-848e-067167be4aff::local.ehrbase.org::1"
    },
    "is_modifiable": true,
    "is_queryable": true
  },
  "time_created": {
    "_type": "DV_DATE_TIME",
    "value": "2021-02-21T18:19:37,791+01:00"
  }
}
```

## Azure EHRbase Application

So far so good! Now the next step is to push EHRbase itself to Azure too.

Stop EHRbase (`CTRL + c`) and setup the Azure Maven plugin:

**IMPORTANT NOTE**: Currently the following command fails for me, or more prescisly it results in an infinite loop with errors. I reported the problem (see the [issue](https://github.com/microsoft/azure-maven-plugins/issues/1319)). But to continue here: just stop the execution (i.e. close the terminal) and head on for now, because the execution was still able to write the config.

``` bash
mvn com.microsoft.azure:azure-webapp-maven-plugin:1.13.0:config
```

My config was like the following, but the next step will show the result directly in the pom.xml again, so just go ahead.

``` text
AppName : server-1613928333259
ResourceGroup : server-1613928333259-rg
Region : westeurope
PricingTier : F1
OS : Linux
Java : Java 11
Web server stack: Tomcat 8.5
Deploy to slot : false
```

In the next step the defaults from the config plugin need to be adjusted to meet EHRbase's requirements. The following list will give an overview of the changes, but also see the pom.xml excerpt for specific details.

- Change the web container to Java SE
- Set properties for the Java runtime
- Set properties for Spring Boot, so EHRbase runs in the cloud profile and can connect to our previously created Postgres instance
- Modify the deployment config to take the correct jar and copy it into the App Service
- Finally allow the jar update on deploy

``` xml
	<plugin>
        <groupId>com.microsoft.azure</groupId>
        <artifactId>azure-webapp-maven-plugin</artifactId>
        <version>1.12.0</version>
        <configuration>
          <schemaVersion>v2</schemaVersion>
          <subscriptionId>your own UUID</subscriptionId>
          <resourceGroup>server-somenumber-rg</resourceGroup>
          <appName>server-somenumber</appName>
          <pricingTier>F1</pricingTier>
          <region>westeurope</region>
		  <!-- important changes are starting here -->
          <runtime>
            <os>Linux</os>
            <javaVersion>Java 11</javaVersion>
            <webContainer>Java SE</webContainer>
          </runtime>
          <appSettings>
            <property>
            <!-- Tell Azure which port you want to use, required for springboot 
                jar applications -->
                <name>PORT</name>
                <value>8080</value>
            </property>
            <!--JVM OPTIONS -->
            <property>
                <name>JAVA_OPTS</name>
                <value>-Xmx512m -Xms512m</value>
            </property>
            <property>
                <name>SPRING_PROFILES_ACTIVE</name>
                <value>cloud</value>
            </property>
            <property>
                <name>SPRING_DATASOURCE_URL</name>
                <value>jdbc:postgresql://${database.hostname}.postgres.database.azure.com:5432/ehrbase</value>
            </property>
            <property>
                <name>SPRING_DATASOURCE_USERNAME</name>
                <value>${database.user}@${database.hostname}</value>
            </property>
            <property>
                <name>SPRING_DATASOURCE_PASSWORD</name>
                <value>${database.pass}</value>
            </property>
          </appSettings>
          <deployment>
            <resources>
              <resource>
                <directory>${project.basedir}/application/target</directory>
                <includes>
                  <include>*.jar</include>
                </includes>
              </resource>
            </resources>
          </deployment>
          <!-- This is to make sure the jar file can be released at the server side -->
          <stopAppDuringDeployment>true</stopAppDuringDeployment>
        </configuration>
      </plugin>
```

The final step is to execute (important: use version 1.12.0 not 1.13.0 - there is an error with the latter):

``` bash
mvn package com.microsoft.azure:azure-webapp-maven-plugin:1.12.0:deploy
```

**IMPORTANT NOTE AGAIN**: I'm not yet sure how to correctly setup a Maven multi-module project like EHRbase for deployment on Azure. The deployment will go through every module and will try to deploy some artifacts. In our case that's not what we want. My workaround - until I figure the configuration out - is simple: just let it run and eventually fail. The necessary step (i.e. regarding `application-...-.jar`) was executed successfully before the error comes up.

Get the URL of the service from the Azure Web Portal or search in in the terminal output:

``` bash
[INFO] Successfully deployed the artifact to https://server-1614002750559.azurewebsites.net
[INFO] Starting Web App after deploying artifacts...
[INFO] Successfully started Web App.
[INFO] 
[INFO] ----------------------< org.ehrbase.openehr:api >-----------------------
[INFO] Building api 0.14.0                                                [2/9]
[INFO] --------------------------------[ jar ]---------------------------------
```

Spinning up EHRbase took several minutes on a free tier App Service instance for me, so be patient now. Take a peek at "Log stream" in the Web Portal to see what's happening. It even seems to be necessary to do an actual request so the App Service spins up the application. Please try this, if nothing happens otherwise

Finally test EHRbase like:

``` bash
curl --header "PREFER: return=representation" --header "CONTENT-LENGTH: 0"\
        --request POST \
    https://server-1613981647489.azurewebsites.net/ehrbase/rest/openehr/v1/ehr | jq
```

A successful response looks like:

``` json
{
  "system_id": {
    "_type": "HIER_OBJECT_ID",
    "value": "3ebc3f08-5ddd-4329-870e-6593f90981c8"
  },
  "ehr_id": {
    "_type": "HIER_OBJECT_ID",
    "value": "61676e13-1a36-487b-828e-706cdcb9fcef"
  },
  "ehr_status": {
    "_type": "EHR_STATUS",
    "name": {
      "_type": "DV_TEXT",
      "value": "EHR Status"
    },
    "subject": {
      "_type": "PARTY_SELF"
    },
    "archetype_node_id": "openEHR-EHR-EHR_STATUS.generic.v1",
    "uid": {
      "_type": "HIER_OBJECT_ID",
      "value": "84377036-8813-4ea4-870c-9a89239418b8::local.ehrbase.org::1"
    },
    "is_modifiable": true,
    "is_queryable": true
  },
  "time_created": {
    "_type": "DV_DATE_TIME",
    "value": "2021-02-22T11:25:12,293Z"
  }
}
```

## Conclusion

EHRbase is now deployed on Azure, using fully managed *App Service* and *Azure Database for PostgreSQL* instances!

Everything works now, but I learned that this process is rather complex and really needs to be optimized. I hope EHRbase will receive updates to improve such deployments.
