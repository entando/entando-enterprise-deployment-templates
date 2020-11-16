# Entando Operator namespace scoped deployment

This Helm chart sets up an Entando Operator instance that only observes events generated
by the Entando Custom Resources in its own namespace.

## When to consider this scenario

This scenario is ideal where developers have the necessary skill and privileges to deploy the Entando Operator
in a given namespace.  

Namespaces are commonly used to ring-fence resources that developers can use in a 
cluster. In such situations, developers are often given free rein within a namespace constrained 
by resource quotas, but are given very few (if any) 
privileges at the cluster level. This is in line with the microservice philosophy of decentralized governance 
and empowering teams to deliver and deploy their software all the way into production environments. 

Deploying the Entando operator within namespace scope also allows better isolation of your Entando Operator instances, as
two Entando Operator instances will never interfere with resources from each other's namespaces. It is therefore
easier to have multiple instance of the Entando Operator  running concurrently in your cluster.

The Entando operator can also be considered to be a fairly lightweight Docker image. It really just triggers
run-to-completion Pods to apply the deployment semantics of an Entando Custom Resource, and as such does not 
consume much memory or CPU time. Deploying multiple operators in a single cluster does therefore not consume 
much more resources than deploying a single operator instance. 

## Advantages
* Entando Operator instances are isolated from each other. Upgrades and configuration changes on the one won't affect 
Entando Application stacks in another namespace. 
* Developers would be able to configure some of the more advanced features of the Entando Operator.
* The Entando Operator is a point of failure only within the scope of the namespace.  
* Upgrades and configuration changes on the Entando Operator will only affect Entando applications in the same namespace.

## Disadvantages
* The Entando Operator and the Entando application stack run in the same namespace and changes to shared Secrets 
and ConfigMaps could affect the execution of the one or the other.
* The Entando Operator can accidentally being mis-configured by developers, although they would be able to fix it themselves.
* Deploying the full Entando application stack and the Entando Operator requires a higher level of skill from Developers.
  
# Customizing this Helm chart
This chart can be customized by modifying the supporting Secrets, modifying the input `values.yaml` file
and/or modifying the requirements specified in the requirements.yaml file

## Secrets
The Entando Operator may require some combination of Secrets to be present in its namespace before being deployed.
Some of these Secrets are mounted in the Entando Operator's filesystem, others are associated to its service account. 
Please read this section before deploying the Entando Operator to its namespace.  
 
### [image-pull-secret.yaml](./sample-secrets/image-pull-secret.yaml) 
ImagePullSecrets carry the Docker configuration that allows the various service accounts to pull images from private
registries. This specific Secret is simply an example to illustrate how an ImagePullSecret can be 
created and will not give the Entando Operator access to any significant registries. You can specify more than one
Secret here and simply list their names in the `operator.imagePullSecrets` list in the [values.yaml](./values.yaml) file.
You would typically need at least one Secret for every registry you use, and possibly even more if you have 
fine-grained access control over the images in your registry. This Secret is commonly used in conjunction with
the ENTANDO_DOCKER_REGISTRY_OVERRIDE environment variable on the Entando Operator's deployment.
For more info about the format of image pull Secrets consult https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry

### [ca-certs.yaml](./sample-secrets/ca-certs.yaml) 
This an opaque Secret in the Entando Operator's namespace that contains the certificates of all trusted certificate authorities 
in your environment. This is generally used mainly for self signed certificates. As is generally the case for opaque
Secrets, there are no constraints on the keys in this Secret. However, limit the values associated with these keys
to only contain a single certificate. Certificate chains are therefore not supported and should ideally be 
split into single files for each
certificate. The Entando Operator will load all of these certificates into a Java keystore that it then configures as the 
trust store for each container that uses Java. Specify the name of this Secret in the `operator.tls.caSecretName`
property in the  [values.yaml](./values.yaml) file.

### [tls-key-cert-pair.yaml](./sample-secrets/tls-key-cert-pair.yaml)
This is a standard Kubernetes TLS Secret that will be associated with Ingresses created by the Entando Operator.
Specify the name of this Secret in the `operator.tls.tlsSecretName` property in the  [values.yaml](./values.yaml) file.
In a namespace scoped deployment, this could be useful if all HTTPS services are exposed on a single hostname. 
This specific Secret 
is simply an example TLS Secret with a self signing certificate chain. As such it should not be considered secure
at all. Please do not ever import this certificate chain in any CA truststore as its private key is available
in this repository for anyone to use. 

An alternative approach to establish TLS support is  
to omit the `operator.tls.tlsSecretName` property entirely and use the `ENTANDO_USE_AUTO_CERT_GENERATION=true` 
environment variable on the Entando Operator's deployment. This will instruct the Entando Operator to leave it to the Kubernetes/
Openshift cluster's router to serve certificates on HTTPS Ingresses.  

For more information about TLS Secrets, consult https://kubernetes.io/docs/concepts/services-networking/ingress/#tls

## requirements.yaml, the entando-k8s-controller-coordinator Helm chart and security considerations 

This Helm chart only has one requirement: entando-k8s-controller-coordinator, a Helm chart that
deploys the entando/entando-k8s-controller-coordinator Docker image. This Docker image contains a Java program that
listens to all relevant events in the namespace, that inspects the associated Entando Custom Resources and then
spawns the relevant "run to completion" Docker image to implement the deployment semantics of the custom resource
in question. In this scenario, the entando-k8s-controller-coordinator Helm chart does not create any
cluster-scoped ClusterRoles or ClusterRoleBindings. It does however create namespace-scoped Roles
and RoleBindings, and therefore requires the installing user to have the equivalent permissions to do this.

In this scenario, the Entando Operator requires read/write access to Secrets, but not watch or list. This implies that if
the Entando Operator does not know the name of a given Secret, it cannot access it. 

## [values.yaml](./values.yaml)

This is the standard input yaml file for the Helm chart. The file itself has been annotated with extensive
comments to describe the function of the various settings. Please consult the file itself for more instructions.

# Generating the deployment yaml files
If you have updated the requirements.yaml file, you may need to pull the latest dependencies:
```
      helm dependency update
``` 
With all the configuration files customized to meet your requirements you can now generate the yaml file:

```   
      mkdir deployment
      helm template ./ --namespace=entando-sample --name=entando --output-dir=deployment
``` 

After running this command, the `deployment/` folder will contain yaml files will contain deployment information 
for both the Entando Operator and for the various Entando Custom Resource required to deploy a typical 
Entando fullstack  application in the same namespace as the Entando Operator.   

# Deploying the Entando Operator
Before deploying the Entando Operator and Entando Custom Resource, please first deploy the Secrets they require:

```
    oc apply -f sample-secrets/
```

Then deploy the files in the `deployment/` directory previously generated with Helm using the `apply` instruction rather than `create`. Generally
this makes it easier to reapply modified versions of the yaml file the future. It is recommended to first deploy
the Entando Operator itself:
```
    kubectl apply -f deployment/entando-bootstrap/charts/operator/templates -n entando-sample/  -n entando-sample
```
and wait for the Entando Operator Pod to be in a read state:

```
    watch kubectl get pods -n entando-sample 
```
Now deploy the Entando Custom Resources
```
    kubectl apply -f deployment/entando-bootstrap/templates -n entando-sample
```
#Some notes on the use of Helm

In a Namespace-scoped deployment of the Entando Operator, the use of the Entando Operator's own Helm chart is 
unavoidable. In this deployment scenarios, developers would therefore require some knowledge of the use of Helm.
It could also be that there are some variables that can be shared between the Entando Operator and the Entando
Custom Resources.

On the plus side, Developers can therefore also modify the Entando Custom Resources included as tempaltes 
in this specific Helm chart. Please look at the the [templates/](./templates) directory and feel free to
customize these templates to meet your requirements.

Please consult our [official documentation](https://dev.entando.org/v6.2/docs/concepts/custom-resources.html) for a
more exhaustive overview of the Entando Custom Resources. 

The [section on the deployment of full Entando application stack](../cluster-scoped-operator/fullstack-app-deployment/README.md#customizing-the-entando-custom-resource-templates) 
in the cluster scoped deployment scenario also applies to these templates.      


# Modifying your generated Entando Custom Resource files

Even though you can modify the Entando Custom Resource templates as mentioned above, you also have to option
 to modify and redeploy the Entando Custom Resources individually from the generated yaml files. There
may be several reasons why you would like to update these custom resources. It may 
take some experimentation to find the optimal resource usage settings. Maybe you want to experiment with your
own custom images for some deployments. Or maybe you want to upgrade the version of one of the images.
Whatever your reasons, you can edit these custom resources manually. Simply make the modifications required,
set the `entandoStatus.entandoDeploymentPhase` on the resource to `requested`, and apply the changes using 
the `oc apply -f` or `kubectl apply -f` command with the correct filename. This will trigger a redeployment
of the Kubernetes resources involved from the Entando Operator.

Alternatively, if you prefer the commandline, you can also use the `oc edit` or `kubectl edit` command with these
custom resources.