# Entando operator namespace scoped deployment

This Helm chart configuration sets up an Entando Operator instance that only observes events generated
by the Entando custom resources in the same namespace

# When to consider this scenario

This scenario is ideal where developers have the necessary skill to deploy the Entando operator, but they do not
have the necessary privileges to create cluster scoped resources such as clusterroles and clusterrolebindings.
Deploying the Enando operator within namespace scope also allows better isolation of your operators, and it is therefore
easier to have multiple instance of the Entando operator  running concurrently in your cluster. 

# Customizing this Helm chart
This chart can be customized by modifying the supporting secrets, modifying the input `values.yaml` file
and/or modifying the requirements specified in the requirements.yaml file

## Secrets
The operator may require some combination of secrets to be present in its namespace before being deployed.
Some of these secrets are mounted in the operator's filesystem, others are associated to its service account. 
Please read this section before deploying the operator to its namespace.  
 
### [image-pull-secret.yaml](./sample-secrets/image-pull-secret.yaml) 
ImagePullSecrets carry the Docker configuration that allows the various service accounts to pull images from private
registries. This specific secret is simply an example to illustrate how an ImagePullSecret can be 
created and will not give the operator access to any significant registries. You can specify more than one
secret here and simply list their names in the `operator.imagePullSecrets` list in the [values.yaml](./values.yaml) file.
You would typically need at least one secret for every registry you use, and possibly even more if you have fine
grained access control over the images in your registry. This secret is commonly used in conjunction with
the ENTANDO_DOCKER_REGISTRY_OVERRIDE environment variable on the operator's deployment.
For more info about the format of image pull secrets consult https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry

### [ca-certs.yaml](./sample-secrets/ca-certs.yaml) 
This an opaque secret in the Operator's namespace that contains the certificates of all trusted certificate authorities 
in your environment. This is generally used mainly for self signed certificates. As is generally the case for opaque
secrets, there are no constraints on the keys in this secret. However, limit the files to only contain a single
certificate. Certificate chains are therefore not supported and should ideally be split into single files for each
certificate. The operator will load all of these files into a Java keystore that it then configures as the 
trust store for each container that uses Java. Specify the name of this secret in the `operator.tls.caSecretName`
property in the  [values.yaml](./values.yaml) file.

### [tls-key-cert-pair.yaml](./sample-secrets/tls-key-cert-pair.yaml)
This is a standard Kubernetes TLS secret that will be associated with Ingresses created by the operator.
Specify the name of this secret in the `operator.tls.tlsSecretName` property in the  [values.yaml](./values.yaml) file.
In a namespace scoped deployment, this is  could be useful if all HTTPS services are exposed on a single hostname. 
This specific secret 
is simply an example TLS secret with a self signing certificate chain. As such it should not be considered secure
at all. Please do not ever import this certificate chain in any CA truststore as its private key is available
in this repository for anyone to use. 

An alternative approach to establish TLS support is  
to omit the `operator.tls.tlsSecretName` property entirely and use the `ENTANDO_USE_AUTO_CERT_GENERATION=true` 
environment variable on the operator's deployment. This will instruct the operator to leave it to the Kubernetes/
Openshift cluster's router to serve certificates on HTTPS Ingresses.  

For more information about TLS secrets, consult https://kubernetes.io/docs/concepts/services-networking/ingress/#tls

## requirements.yaml, the entando-k8s-controller-coordinator Helm chart and security considerations 

This Helm chart only has one requirement: entando-k8s-controller-coordinator, a Helm chart that
deploys the entando/entando-k8s-controller-coordinator Docker image. This Docker image contains a Java program that
listens to all relevant events in the namespace, that inspects the associated Entando custom resources and then
spawns the relevant "run to completion" Docker image to implement the deployment semantics of the custom resource
in question. In this scenario, the entando-k8s-controller-coordinator Helm chart does not create any
cluster scoped roles or rolebindings. It does however create namespace scoped roles and rolebindings, and
therefore requires the installing user to have the equivalent permissions to do this.
 
In this scenario, the operator requires read/write access to secrets, but not watch or list. This implies that if
the operator does not know the name of a given secret, it cannot access it. 


## [values.yaml](./values.yaml)

This is the standard input yaml file for the Helm chart. The file itself has been annotated with extensive
comments to describe the function of the various settings. Please consult the file itself for more instructions.

# Generating the deployment YAML file
If you have updated the requirements.yaml file, you may need to pull the latest dependencies:
```
    helm dependency update
``` 
With all the configuration files customized to meet your requirements you can now generate the yaml file:

```
      helm template ./ --namespace=entando-sample --name=entando > deployment.yaml
``` 

# Deploying the operator
Before deploying the operator, please first deploy the secrets it may use:

```
    oc apply -f sample-secrets/
```

Then deploy the `deployment.yaml` file previously generated with Helm using the `apply` instruction rather than `create`. Generally
this makes it easier to reapply modified versions of the yaml file in future, e.g.:
```
    oc apply -f deployment.yaml -n entando-sample
```
or
```
    kubectl apply -f deployment.yaml  -n entando-sample
```
