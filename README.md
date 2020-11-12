# entando-enterprise-deployment-templates
This project contains a couple of Helm templates that can be used to facilitate some of the typical deployment 
scenarios Entando 6.X supports.

# Cluster scoped deployment.
For scenarios where you need a single Entando operator to process all Entando custom resources in the cluster, 
follow the instructions in this [README.md](./cluster-scoped-operator/README.md)

# Namespace scoped deployment.
For scenarios where you only have a single namespace at your disposal to deploy the Entando operator and
your Entando custom resources, follow the instructions in this [README.md](./namespace-scoped-operator/README.md)
