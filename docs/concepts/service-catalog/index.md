---
title: Service Catalog
approvers:
- chenopis
---

{% capture overview %}
{% include templates/glossary/snippet.md term="service-catalog" length="all" %}

{% endcapture %}


{% capture body %}
## Example use case

An [Application Developer](/docs/reference/glossary/?user-type=true#term-application-developer) wants to use a datastore, such as MySQL, as part of their application running in a Kubernetes cluster.
However, they do not want to deal with the overhead of setting one up and administrating it themselves.
Fortunately, there is a cloud provider that offers MySQL databases as a *Managed Service* through their *Service Broker*.

A *Service Broker*, as defined by the [Open Service Broker API spec](https://github.com/openservicebrokerapi/servicebroker/blob/v2.13/spec.md), is an endpoint for a set of Managed Services offered and maintained by a third-party, which could be a cloud provider such as AWS, GCP, or Azure.
Some examples of *Managed Services* are Azure SQL Database, Amazon EC2, and Google Cloud Pub/Sub, but they can be any software offering that can be used by an application, typically available via HTTP REST endpoints.
{: .note}

Using Service Catalog, the Cluster Operator can browse the list of Managed Services offered by a Service Broker, provision a MySQL database instance, and bind with it to make it available to the application within the Kubernetes cluster.
The Application Developer therefore does not need to concern themselves with the implementation details or management of the database.
Their application can simply use it as a service.

## Architecture

Service Catalog is built on the [Open Service Broker API](https://github.com/openservicebrokerapi/servicebroker) and is implemented as an extension API server, controller manager, and etcd operator.
It communicates with Service Brokers via the OSB API and acts as an intermediary for the Kubernetes API Server in order to negotiate the initial provisioning and return the credentials necessary for the application to use a Managed Service.

![Service Catalog Architecture](/images/docs/service-catalog-architecture.svg)


### API Resources

Service Catalog installs the `servicecatalog.k8s.io` API and provides the following Kubernetes resources:

* `ServiceBroker`: An in-cluster representation of a Service Broker, encapsulating its server connection details.
These are created and managed by Cluster Operators who wish to use that broker server to make new types of Managed Services available within their cluster.
* `ServiceClass`: A Managed Service offered by a particular Service Broker.
When a new `ServiceBroker` resource is added to the cluster, the Service Catalog controller connects to the Service Broker to obtain a list of available Managed Services. It then creates a new `ServiceClass` resource corresponding to each Managed Service.
* `ServiceInstance`: A provisioned instance of a `ServiceClass`.
These are created by Cluster Operators to make a specific instance of a Managed Service available for use by one or more in-cluster applications.
When a new `ServiceInstance` resource is created, the Service Catalog controller will connect to the appropriate Service Broker and instruct it to provision the service instance.
* `ServiceBinding`: Access credentials to a `ServiceInstance`.
These are created by Cluster Operators who want their applications to make use of a Service `ServiceInstance`.
Upon creation, the Service Catalog controller will create a Kubernetes `Secret` containing connection details and credentials for the Service Instance, which can be mounted into Pods.

### Mutual TLS encryption

The mutual TLS protocol encrypts communication between the Service Catalog extension API server and the main Kubernetes API server. 

During installation, Service Catalog creates its own certificate authority (CA), and generates its own public and private keys, signed by this CA.
The CA public certificate is installed into the main API server when Service Catalog registers its API.
Service Catalog accesses the main API server CA certificate from the API server ConfigMap, after being granted the *extension-apiserver-authentication-reader* role. 

Certificate rotation is handled by exposing a Service Catalog URL, similar to a `/statusz` endpoint, which returns the number of days until the CA and certificates expire.
Alerts for certificate rotation can be created by monitoring this URL.

### Authentication

Service Catalog supports these methods of authentication: 

* Basic (username/password)
* [OAuth 2.0 Bearer Token](https://tools.ietf.org/html/rfc6750)

## Usage

The Cluster Operator can use the Service Catalog API Resources to provision Managed Services and make them available within the Kubernetes cluster. The steps involved are:

1. Listing the Managed Services available from a Service Broker.
1. Provisioning a new instance of the Managed Service.
1. Binding to the Managed Service, which returns the connection credentials.

### Listing Managed Services

The Cluster Operator must first create a `ServiceBroker` resource within the `servicecatalog.k8s.io` group. This resource contains the URL and connection details necessary to access a Service Broker endpoint.

The following is an example of a `ServiceBroker` resource:

```yaml
apiVersion: servicecatalog.k8s.io/v1alpha1
kind: ServiceBroker
metadata:
  name: cloud-broker
spec:
  url:  https://servicebroker.somecloudprovider.com/v1alpha1/projects/service-catalog/brokers/default
  # Describes the secret which contains the short-lived bearer token
  authInfo:
    bearer:
      secretRef:
        name: cloud-svc-account-secret
        namespace: service-catalog
```

1. Once the `ServiceBroker` resource is added to Service Catalog, it triggers a *List Services* call to the external Service Broker.
1. The Service Broker returns a list of available Managed Services, which is cached locally in a `ServiceClass` resource.
1. The Cluster Operator can then get the list of available Managed Services using the following command:

        kubectl get serviceclasses

![List Services](/images/docs/service-catalog-list.svg){:height="80%" width="80%"}


### Provisioning a new instance

The Cluster Operator can initiate the provisioning of a new instance by creating a `ServiceInstance` resource. 

The following is an example of a `ServiceInstance` resource:

```yaml
apiVersion: servicecatalog.k8s.io/v1alpha1
kind: ServiceInstance
metadata:
  name: cloud-postgresql-instance
  namespace: cloud-apps
spec:
  # References one of the previously returned services
  serviceClassName: postgresql
  planName: postgresql-plan
```

1. Once the `ServiceInstance` resource is created, Service Catalog initiates a *Provision Instance* call to the external Service Broker.
1. The Service Broker creates a new instance of the Service.
1. The Service Consumer can then check the status of the instance to see if it is ready.

![Provision a Service](/images/docs/service-catalog-provision.svg){:height="80%" width="80%"}

### Binding to a Service

1. The Service Consumer requests a binding to the instance by sending a POST command to the Service Catalog API Server, which creates and persists a Binding Resource.
1. Service Catalog in turn requests a binding from the Service Broker using a PUT command.
1. The Service Broker then returns provider-specific information, such as coordinates, credentials, configs, necessary for Kubernetes to connect and access the Service instance.
1. The binding information and credentials are delivered to the Kubernetes API Server as a set of Kubernetes objects, such as a Service, Secret, ConfigMap, or Pod Preset.

![Bind to a Service](/images/docs/service-catalog-bind.png){:height="80%" width="80%"}

### Mapping the connection credentials

Lorem ipsum dolor sit amet, consectetur adipiscing elit. Quisque egestas nisl eget justo eleifend, ac lacinia lectus pellentesque. Donec facilisis, massa suscipit suscipit sollicitudin, augue libero imperdiet lacus, sed hendrerit metus massa mattis nisi. Suspendisse vestibulum massa id hendrerit lacinia. Aliquam eleifend nulla a metus molestie bibendum.

Sed eu blandit leo. Suspendisse ut laoreet elit. Praesent elementum placerat fringilla. Integer convallis metus felis, vitae pulvinar orci viverra nec. Cras auctor luctus eros, sed fringilla nibh convallis non.

{% endcapture %}


{% capture whatsnext %}
* [Install Service Catalog](/docs/tasks/service-catalog/install-service-catalog/) in your Kubernetes cluster.
* Learn how to [provision and bind an external Service with Service Catalog](/docs/tasks/service-catalog/provision-bind-external-service/).
* View [sample service brokers](https://github.com/openservicebrokerapi/servicebroker/blob/master/gettingStarted.md#sample-service-brokers).
* Explore the [kubernetes-incubator/service-catalog](https://github.com/kubernetes-incubator/service-catalog) project.

{% endcapture %}


{% include templates/concept.md %}
