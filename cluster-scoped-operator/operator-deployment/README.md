# Entando operator cluster scoped deployment

This Helm chart configuration sets up an Entando Operator instance that observes events generated
by the Entando custom resources in the entire cluster.

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
In a cluster scoped deployment, this is  only useful for a wildcard certificate that correspond with the 
`ENTANDO_DEFAULT_ROUTING_SUFFIX` environment variable specified on the operator's deployment. This specific secret 
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
listens to all relevant events in the cluster, that inspects the associated Entando custom resources and then
spawns the relevant "run to completion" Docker image to implement the deployment semantics of the custom resource
in question. The entando-k8s-controller-coordinator Helm chart also creates cluster scoped roles
and rolebindings. It is therefore of absolute  importance to consult your cluster administrators
and security specialists to verify that these cluster roles and rolebindings
fall within the constraints of your organization's security governance. None of the  cluster roles in question
grant the operator `escalate` privileges on roles or `bind` privileges on rolebindings. This implies that
the operator will never be able to give privileges to any other actors such as serviceaccounts that it doesn't have
itself already. The operator also does not have the privileges to read secrets in any namespaces other than its own,
and it also cannot list or watch secrets. 


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
      helm template ./ --namespace=entando-operator --name=entando > operator-deployment.yaml
``` 

# Deploying the operator
Before deploying the operator, please first deploy the secrets it may use:

```
    oc apply -f sample-secrets/
```

Then deploy the `operator-deployment.yaml` previously generated by Helm using the `apply` instruction rather than `create`. Generally
this makes it easier to reapply modified versions of the yaml file in future, e.g.:
```
    oc apply -f operator-deployment.yaml -n entando-operator
```
or
```
    kubectl apply -f operator-deployment.yaml  -n entando-operator
```
