# An example Entando Fullstack App deployment

Ultimately the goal of deploying Entando custom resources is to get your EntandoApps and
your EntandoPlugins up and running in a Kubernetes/Openshift cluster. 
A typical Entando Fullstack App requires a couple of additional deployments to support these
EntandoApps and EntandoPlugins, such as shared database services, Keycloak, and Entando's own
abstraction layer on top of Kubernetes, the Entando Kubernetes Service. This Helm chart illustrates
how you can go about getting the entire stack up and running
  
# A note about the use of Helm

One of the most significant design goals of the Entando operator is to simplify the deployment
of fullstack Entando Apps. As such, it is entirely conceivable, perhaps even desirable for
developers to create the Entando custom resource yaml files by hand, rather than using a 
Helm template to generate it. This is the case especially for cluster scoped deployments 
where developers are not required to have any knowledge or skill in deploying the Operator \
itself. 

For developers that want to deploy their own EntandoApps and EntandoPlugins in their own
namespaces, there is no requirement to know  Helm
as it is fairly easy to hand craft the necessary yaml files to deploy their apps and plugins. 
For most deployment scenarios that leverage a cluster scoped operator, we don't
anticipate much benefit would come from introducing the extra abstraction that Helm provides.

However, in order to illustrate how similar Entando configurations can be deployed in a cluster
we have decided to present these options as a Helm chart. Please feel free to inspect the resulting
yaml files, copy them, modify them and experiment until you can meet the requirements of your specific
deployment scenario.

# Customizing this Helm chart
As with the Operator's deployment chart, this chart can be customized by modifying the supporting secrets or
modifying the input `values.yaml` file. A significant difference between this chart and the Entando operator's
deployment chart is that it is entirely self contained and can therefore also be customized by editing
the yaml files in the templates directory directly.

## [shared-database-credentials](./sample-secrets/shared-database-credentials.yaml)
The only secret we need to be concerned about when deploying an Entando fullstack app is the shared
database secret.  This secret is optional and only used if you have an existing external database
you want the Entando fullstack to use. If this is the case, please provide the username and
password of a superuser in your database management system of choice. At the very least,
this user requires the necessary privileges to create new users and schemas, (or in the case
of MySQL, new databases). The secret itself can then be made available to the operator
by specifying its name in the `sharedDatabase.secretName` property in the
[values.yaml](values.yaml) file. Please verify that the `app.dbms` property reflects the database
management system of the database you are pointing the Entando fullstack app to. We currently
support the values `postgresql`, `mysql` or `oracle`.

## [values.yaml](values.yaml)
### Common properties

`app.dbms` 

Use this property to specify which DBMS drivers the various deployments should be using. When using a 
shared and/or external database specified with an  EntandoDatabaseService resource, this property will  be used
to match the deployments with the appropriate EntandoDatabaseService. This property is required. 
 
`app.singleHostName`

This property will force all the Ingresses to expose the underlying HTTP service on the same hostname.
If you leave this property out entirely, the default routing suffix configured with the 
`ENTANDO_DEFAULT_ROUTING_SUFFIX`  environment variable will be used to dynamically generate a valid
hostname for deployments.

`app.operatorId` 

The value of this property needs to correspond with the value of the `ENTANDO_K8S_OPERATOR_ID` 
environment variable on the Entando operator's deployment. This value will be propagated to
the standard `entando.org/operator-id` annotation on all the resources to ensure the previously
deployed operator won't ignore these resources.

### Database configuration options
This Helm chart supports three possible database configurations. It can be configured to use an
existing, shared database with the credentials supplied, it can be configured to dynamically 
deploy a shared database container, or it can be configured to let every deployed service use
its own dedicated database.

#### Shared existing external database
In order to point the entire deployment to an existing database, please provide values for the 
following properties: 

`sharedDatabase.databaseName`: the name of the database (optional for MySQL)

`sharedDatabase.host`: the database server name

`sharedDatabase.port`: the port the database service is exposed on

`sharedDatabase.secretName`: the secret that carries the super user's credentials

Please comment out the property named `sharedDatabase.createDeployment` as you will not be 
creating a deployment.  

#### Shared database deployed on demand in the namespace
For a shared database deployed in the namespace, just activate this property:

`sharedDatabase.createDeployment: true`


This will result in the standard database container for the property `app.dbms` being  
deployed in the namespace. All subsequent deployments that match the `app.dbms` property
will then use this database service, and create schemas and users on demand. 

#### Dedicated databases deployed for each Entando custom resource

If you prefer to have dedicated database for each service deployed by the Entando operator, you
can just go ahead and remove the entire `sharedDatabase` section. 
 
## The shared database service: [shared-database-service.yaml](./templates/shared-database-service.yaml)
This resource is only deployed if the `sharedDatabase` section mentioned above has been put in place.
Read more about this custom resource [here](https://dev.entando.org/v6.2/docs/concepts/custom-resources.html#entandodatabaseservice)  

## Keycloak: [keycloak-server.yaml](./templates/keycloak-server.yaml)
This resource is deploys a Keycloak server in the namespace.
Read more about this custom resource [here](https://dev.entando.org/v6.2/docs/concepts/custom-resources.html#entandokeycloakserver)  
If there is only one Keycloak deployment in a namespace, it will automatically be used by other services deployed in
this namespace. If more than one Keycloak is deployed in a namespace, one can use the `spec.keycloakToUse.name` and
optionally the `spec.keycloakToUse.namespace` properties to point to specific Keycloak instances. A third approach is
to set the poperty `spec.isDefault` on the EntandoKeycloakServer to true, in which case
this Keycloak instance will be used for  all deployments that have neither a value for the `spec.keycloakToUse.name` 
property, nor a EntandoKeycloakServer instance in its own namespace. 


## Entando's cluster infrastrcuture: [cluster-infrastructure.yaml](./templates/cluster-infrastructure.yaml)
This resource is deploys the entando-k8s-service  in the namespace.
Read more about this custom resource [here](https://dev.entando.org/v6.2/docs/concepts/custom-resources.html#entandoclusterinfrastructure)  
As with the EntandoKeycloakServer custom resource, this deployment will be used by other services in this namespace 
if there is only one EntandoClusterInfrastructure deployment in a namespace. If more than one EntandoClusterInfrastructure
is deployed in a namespace, one can use the `spec.clusterInfrastructure.name` and
optionally the `spec.clusterInfrastructure.namespace` properties to point to specific EntandoClusterInfrastructure instances.
A third approach is 
to set the poperty `spec.isDefault` on the EntandoClusterInfrastructure to true, in which case
this entando-k8s-service  will be used for  all deployments that have neither a value for the `spec.clusterInfrastructure.name` 
property, nor a EntandoClusterInfrastructure instance in its own namespace. 
 
## The Entando App: [entando-app.yaml](./templates/entando-app.yaml)
This resource deploys the actual EntandoApp. 
Read more about this custom resource [here](https://dev.entando.org/v6.2/docs/concepts/custom-resources.html#entandoapp)  

## The Entando Plugin: [entando-plugin.yaml](./templates/entando-plugin.yaml)
This resource deploys a sample EntandoPlugin. Read 
Read more about this custom resource [here](https://dev.entando.org/v6.2/docs/concepts/custom-resources.html#entandoplugin)  

## Linking tye plugin and the app: [app-plugin-link.yaml](./templates/app-plugin-link.yaml) the App and the Plugin 
This resource deploys a links the previously deployed EntandoPlugin to the EntandoApp by exposing it on the
EntandoApp's Ingress. 
Read more about this custom resource [here](https://dev.entando.org/v6.2/docs/concepts/custom-resources.html#entandoapppluginlink)  

## Putting it all together: [entando-deployment-sequence.yaml](./templates/entando-deployment-sequence.yaml)
This custom resource ensures that all the previously deployed resources are processed in a very specific
sequence by the operator. The sequence in which they are process is determined by their position in the 
`spec.components` property in the EntandoCompositeApp. Use this only for the initial deployment of the Entando
custom resources. After these services have been deployed, you are free to deploy them individually. Generally,
it is better to orchestrate  deployments independently, and not to assume any sequence, so it is best to only
use this resource once and only with the initial deployment. 
Read more about this custom resource [here](https://dev.entando.org/v6.2/docs/concepts/custom-resources.html#entandocompositeapp)  

# Deploying the fullstack app

If needed, deploy the sample external database secret(s) that you have prepared for the apps. 
```
     oc apply -f fullstack-app-deployment/sample-secrets/shared-database-credentials.yaml -n entando-sample-app 

```
Then generate the yaml files containing the Entando custom resources
```
      mkdir sample/ 
      helm template ./fullstack-app-deployment --namespace=entando-sample-app  --name=blue --output-dir=sample
```


Deploy the Entando fullstack app. Make sure to use the `apply` action to allow you to reapply in future without
unnecessary restarts of the Operator:
```
      oc apply -f sample/entando-fullstack/templates/ -n entando-sample-app
```

It is important to understand what is happening in this last command. This command creates all the Entando custom
resources at the same time. However, all of them except the one specified in the `entando-deployment-sequence.yaml` file 
are then ignored by the operator. You will note that the property `entandoStatus.entandoDeploymentPhase` on all of these
resources is specified as `ignored`. The operator only processes resources when the value of this property is
 `requested` 

# Modifying your deployment files

As mentioned earlier, you are now free to modify and redeploy the Entando custom resources individually. There
may be several reasons why you would like to update these custom resources. It may 
take some experimentation to find the optimal resource usage settings. Maybe you want to experiment with your
own custom images for some of the deployments. Or maybe you want to upgrade the version of one of the images.
Whatever your reasons, you can edit these custom resources manually. Simply make the modifications required,
set the `entandoStatus.entandoDeploymentPhase` on the resource to `requested`, and apply the changes using 
the `oc apply -f` or `kubectl apply -f` command with the correct filename. This will trigger a redeployment
of the Kubernetes resources involved from the operator.

Alternatively, if you prefer the commandline, you can also use the `oc edit` or `kubectl edit` command with these
custom resources.