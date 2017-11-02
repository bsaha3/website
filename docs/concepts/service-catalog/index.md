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

An [Application Developer](/docs/reference/glossary/?user-type=true#term-application-developer) wants to use a datastore, such as MySQL, as part of their application running in a Kubernetes cluster. However, they do not want to deal with the overhead of setting one up and administrating it themselves. Fortunately, there is a cloud provider that offers MySQL databases as a *Managed Service* through their *Service Broker*.

A *Service Broker*, as defined by the [Open Service Broker API spec](https://github.com/openservicebrokerapi/servicebroker/blob/v2.13/spec.md), is an endpoint for a set of Managed Services offered and maintained by a third-party, which could be a cloud provider such as AWS, GCP, or Azure. Some examples of *Managed Services* are Azure SQL Database, Amazon EC2, and Google Cloud Pub/Sub, but they can be any software offering that can be used by an application and are typically available via HTTP REST endpoints.
{: .note}

Using Service Catalog, the Cluster Operator can browse the list of Managed Services offered by a Service Broker, provision a MySQL database instance, and bind with it to make it available to the application. The Application Developer therefore does not need to concern themselves with the implementation details or management of the database. Their application can simply use it as a service.

## Architecture

Service Catalog is built on the [Open Service Broker API](https://github.com/openservicebrokerapi/servicebroker) and is implemented as a Kubernetes API extension server. It communicates with Service Brokers via the OSB API and acts as an intermediary for the Kubernetes API Server in order to negotiate the initial provisioning and return the connection details and credentials necessary for the application to use the Managed Service.  

![Service Catalog Architecture](/images/docs/service-catalog-architecture.png)

### API Resources

Service Catalog installs the `servicecatalog.k8s.io` API and provides the following Kubernetes resources:

* `ServiceBroker`: An in-cluster representation of a broker server. A resource of this
type encapsulates connection details for that broker server. These are created
and managed by cluster operators who wish to use that broker server to make new
types of Managed Services available within their cluster.
* `ServiceClass`: A *type* of Managed Service offered by a particular broker.
Each time a new `ServiceBroker` resource is added to the cluster, the Service Catalog
controller connects to the corresponding broker server to obtain a list of
service offerings. A new `ServiceClass` resource will automatically be created
for each.
* `ServiceInstance`: A provisioned instance of a `ServiceClass`. These are created
by cluster users who wish to make a new concrete *instance* of some *type* of
Managed Service to make that available for use by one or more in-cluster
applications. When a new `ServiceInstance` resource is created, the Service Catalog
controller will connect to the appropriate broker server and instruct it to
provision the service instance.
* `ServiceBinding`: Access credential to a `ServiceInstance`. These
are created by cluster users who want their applications to make use of a
Service `ServiceInstance`. Upon creation, the Service Catalog controller will
create a Kubernetes `Secret` containing connection details and credentials for
the Service Instance. Such `Secret`s can be mounted into pods as usual.


## Usage

The Cluster Operator can use the Service Catalog API Resources to provision Managed Services and make them available within the Kubernetes cluster. Generally, the steps involved are:

1. Add a Broker Resource.
2. List the Managed Services available from a Service Broker.
3. Provision a new instance of the Managed Service.
4. Bind to the Managed Service, which returns the connection credentials.
5. Map the connection credentials into the Kubernetes application.

### List Managed Services



The API call sequence is:

1. The Cluster Operator must first add Broker Resources to the Service Catalog API Server. These resources identify available Service Brokers and point to their URL endpoints.
1. Service Catalog then requests a list of Services from the Service Broker.
1. The Service Broker returns a list of Services to the ServiceClass resource, which is created and persisted in the Service Catalog API Server.
1. The Service Consumer can then locally query the ServiceClass Resource for a list of available Services.

![List Services](/images/docs/service-catalog-list.svg){:height="80%" width="80%"}

### Provision a new Service

1. The Service Consumer provisions a new instance by sending a POST command to the Service Catalog API Server, which creates and persists an Instance Resource.
1. Service Catalog then requests an instance from the Service Broker by sending an PUT command.
1. The Service Broker creates a new instance of the Service. 
1. If the creation was successful, the Service Broker returns an HTTP 200 response.
1. The Service Consumer can then check the status of the instance to see if it is ready.

![Provision a Service](/images/docs/service-catalog-provision.svg){:height="80%" width="80%"}

#### Bind to a Service

1. The Service Consumer requests a binding to the instance by sending a POST command to the Service Catalog API Server, which creates and persists a Binding Resource.
1. Service Catalog in turn requests a binding from the Service Broker using a PUT command.
1. The Service Broker then returns provider-specific information, such as coordinates, credentials, configs, necessary for Kubernetes to connect and access the Service instance.
1. The binding information and credentials are delivered to the Kubernetes API Server as a set of Kubernetes objects, such as a Service, Secret, ConfigMap, or Pod Preset.

![Bind to a Service](/images/docs/service-catalog-bind.svg){:height="80%" width="80%"}


### Performing cleanup

Once a Service instance is no longer needed, it can be released and cleaned up by using  the *Unbind* and *Deprovision* operations.

#### Unbinding the instance

1. The Service Consumer sends a DELETE command to Service Catalog API Server.
1. Service Catalog in turn sends a DELETE command to the Service Broker.
1. The Service Broker performs any relevant business logic and removes the binding.
1. The Service Broker returns an HTTP 202 response, which triggers the Service Catalog Controller to remove any corresponding binding resources from Kubernetes and run any finalization tasks.
1. The Service Catalog API Server deletes the Binding Resource and returns a response to the Service Consumer.

![Unbind the instance](/images/docs/service-catalog-unbind.svg){:height="80%" width="80%"}

#### Deprovisioning the instance

1. The Service Consumer can then send a DELETE command to the Service Catalog API Server to deprovision the Service instance.
1. Service Catalog in turn sends a DELETE command to the Service Broker.
1. The Service Broker deletes the Service instance and performs its own cleanup procedure.
1. The Service Broker returns an HTTP 202 response, which triggers the Service Catalog Controller run any finalization tasks.
1. The Service Catalog API Server deletes the Instance Resource and returns a response to the Service Consumer.

![Deprovision the instance](/images/docs/service-catalog-deprovision.svg){:height="80%" width="80%"}

{% endcapture %}


{% capture whatsnext %}
* [Install Service Catalog](/docs/tasks/service-catalog/install-service-catalog/) in your Kubernetes cluster.
* Learn how to [provision and bind an external Service with Service Catalog](/docs/tasks/service-catalog/provision-bind-external-service/).
* View [sample service brokers](https://github.com/openservicebrokerapi/servicebroker/blob/master/gettingStarted.md#sample-service-brokers).
* Explore the [kubernetes-incubator/service-catalog](https://github.com/kubernetes-incubator/service-catalog) project.

{% endcapture %}


{% include templates/concept.md %}
