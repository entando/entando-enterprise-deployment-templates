# entando-enterprise-deployment-templates
This project contains a couple of Helm templates that can be used to facilitate some of the typical deployment 
scenarios Entando 6.X supports.

# Cluster scoped deployment.
For scenarios where you need a single Entando Operator to process all Entando Custom Resources in the cluster, 
follow the instructions in this [README.md](./cluster-scoped-operator/README.md)

## When should I consider this scenario
This deployment scenario is useful in contexts where your Kubernetes/Openshift developers are more restricted in
what they are permitted to do on the cluster. It could be that their Kubernetes/Openshift skills are limited.
Maybe your organization has policies that only certain Docker registries and images may be used, or
that Developers should not be managing operators and their custom resources. In this scenario, it is likely  
you would have cluster administrators that have permissions to deploy the Entando Operator, and there is a 
clear separation between the duties of cluster administrators and developers.

## Advantages
* Developers require much less access to the cluster to deploy Entando Apps, resulting in a more secure setup
* The Entando Operator is better isolated from the Entando application stack
* There is less chance of the Entando Operator accidentally being mis-configured by developers.
* It is generally easier for Developers to deploy Entando apps, and it is even possible to deploy apps without using Helm
charts at all

## Disadvantages
* Managing more than one Entando Operator instance can get quite complicated with this deployment option
* Developers may not be able to configure some of the advanced features of the Entando operator
* The Entando Operator becomes a central point of failure. 
* Upgrades and configuration changes on the Entando Operator will affect Entando applications in the entire cluster and need to be coordinated carefully.
 

# Namespace scoped deployment.
For scenarios where you only have a single namespace at your disposal to deploy the Entando Operator and
your Entando Custom Resources, follow the instructions in this [README.md](./namespace-scoped-operator/README.md)


## When to consider this scenario

This scenario is ideal where developers have the necessary skill and privileges to deploy the Entando Operator
in a given namespace.  

Namespaces are commonly used to ring-fence resources that developers can use in a 
cluster. In such situations, developers are often given free rein within a namespace constrained 
by resource quotas, but are given very few (if any) 
privileges at the cluster level. This is in line with the microservice philosophy of decentralized governance 
and empowering teams to deliver their software all the way into production environments. 

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
 