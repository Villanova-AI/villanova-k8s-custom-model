[![Build Status](https://img.shields.io/endpoint?url=https%3A%2F%2Fstatusbadge-jx.apps.serv.run%2Fvillanova-k8s%2Fvillanova-k8s-custom-model)](https://github.com/villanova-k8s/devops-results/tree/logs/jenkins-x/logs/villanova-k8s/villanova-k8s-custom-model/master)
[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=villanova-k8s_villanova-k8s-custom-model&metric=alert_status)](https://sonarcloud.io/dashboard?id=villanova-k8s_villanova-k8s-custom-model)
[![Coverage](https://sonarcloud.io/api/project_badges/measure?project=villanova-k8s_villanova-k8s-custom-model&metric=coverage)](https://villanova-k8s.github.io/devops-results/villanova-k8s-custom-model/master/jacoco/index.html)
[![Vulnerabilities](https://sonarcloud.io/api/project_badges/measure?project=villanova-k8s_villanova-k8s-custom-model&metric=vulnerabilities)](https://villanova-k8s.github.io/devops-results/villanova-k8s-custom-model/master/dependency-check-report.html)
[![Code Smells](https://sonarcloud.io/api/project_badges/measure?project=villanova-k8s_villanova-k8s-custom-model&metric=code_smells)](https://sonarcloud.io/dashboard?id=villanova-k8s_villanova-k8s-custom-model)
[![Security Rating](https://sonarcloud.io/api/project_badges/measure?project=villanova-k8s_villanova-k8s-custom-model&metric=security_rating)](https://sonarcloud.io/dashboard?id=villanova-k8s_villanova-k8s-custom-model)
[![Technical Debt](https://sonarcloud.io/api/project_badges/measure?project=villanova-k8s_villanova-k8s-custom-model&metric=sqale_index)](https://sonarcloud.io/dashboard?id=villanova-k8s_villanova-k8s-custom-model)

# villanova-k8s-custom-model
Villanova's Custom Resource Definition model for Kubernetes. Used in our custom controllers as well as our Kubernetes 
infrastructure. Use this project when integrating Villanova and Kubernetes.

Generally this project adheres to a similar builder pattern as the one used in the Fabric8 Kubernetes/Java client. 

# Usage

## Step 1: resolving the correct CustomResourceOperation

The first step in reading and/or manipulating Villanova's custom resources, you need to resolve the 
CustomResourceOperation in the correct state. This object is required to use the Fabric8 Java client to
interact with custom resources in Kubernetes in a type safe manner.  
In order to do this, you may need to first deploy the correct 
custom resource definition. Here is a code snippet to 'lazily' deploy the custom resource definition for the 
VillanovaApp custom resource, and then to resolve the correct CustomResourceOperation

```java
        villanovaAppCrd = client.customResourceDefinitions().withName(VillanovaApp.CRD_NAME).get();
        if (villanovaAppCrd == null) {
            List<HasMetadata> list = client.load(Thread.currentThread().getContextClassLoader()
                    .getResourceAsStream("crd/VillanovaAppCRD.yaml")).get();
            villanovaAppCrd = (CustomResourceDefinition) list.get(0);
            // see issue https://github.com/fabric8io/kubernetes-client/issues/1486
            villanovaAppCrd.getSpec().getValidation().getOpenAPIV3Schema().setDependencies(null);
            client.customResourceDefinitions().create(villanovaAppCrd);
        }
        return (CustomResourceOperationsImpl<VillanovaApp, VillanovaAppList, DoneableVillanovaApp>) client
            .customResources(villanovaAppCrd, VillanovaApp.class, VillanovaAppList.class, DoneableVillanovaApp.class);

```

## Step 2: Building and creating a custom resource against the custom resource definition

There are different approaches one can use to create a new instance of custom resource. one approach is to use
the 'builder' class of that custom resource. This class can be identified by the name of the custom resource 
definition, e.g. `VillanovaApp`, suffixed with the word `Builder`, e.g. `VillanovaAppBuilder`. Here is an example
of how to build and create an `VillanovaApp`

```java
        CustomResourceOperationsImpl<VillanovaApp, VillanovaAppList, DoneableVillanovaApp> villanovaApps=...
        VillanovaApp villanovaApp = new VillanovaAppBuilder()
                .withNewMetadata().withName(MY_APP)
                .withNamespace(MY_NAMESPACE)
                .endMetadata()
                .withNewSpec()
                .withDbms(DbmsImageVendor.MYSQL)
                .withVillanovaImageVersion(VILLANOVA_IMAGE_VERSION)
                .withJeeServer(JeeServer.WILDFLY)
                .withReplicas(5)
                .withTlsEnabled(true)
                .withIngressHostName(MYINGRESS_COM)
                .withKeycloakServer(MYKEYCLOAKNAMESPACE, MY_KEYCLOAK)
                .endSpec()
                .build();
        villanovaApps.inNamespace(MY_NAMESPACE).create(villanovaApp);

```

## Step 3: Retrieving and editing a custom resource against the custom resource definition

The Fabric8 Kubernetes Java client requires a class that exposes the 'builder' for the custom resource, but with
a callback to update the resource when the `done` method is invoked. The resulting fluent syntax looks very similar
to the traditional builder, but also allows for the retrieved model object to be updated on completion, e.g. 

```java
        villanovaApps.inNamespace(MY_NAMESPACE).withName(MY_APP).edit()
                .editMetadata()
                    .addToLabels(MY_LABEL, MY_VALUE)
                .endMetadata()
                .editSpec()
                    .withDbms(DbmsImageVendor.MYSQL)
                    .withVillanovaImageVersion(VILLANOVA_IMAGE_VERSION)
                    .withJeeServer(JeeServer.WILDFLY)
                    .withReplicas(5)
                    .withTlsEnabled(true)
                    .withIngressHostName(MYINGRESS_COM)
                    .withKeycloakServer(MYKEYCLOAKNAMESPACE, MY_KEYCLOAK)
                .endSpec()
                .withStatus(new WebServerStatus("some-qualifier"))
                .withStatus(new DbServerStatus("another-qualifier"))
                .withPhase(VillanovaDeploymentPhase.STARTED)
                .done();

```

# Known issues

## High parameter count in constructors

Please note that this pattern leads to constructors with an excessive parameter count. This constructor is declared
at package scope and is intended for internal use only. If you find yourself 
using one of these  constructors, please rather look at the builder class associated with the model class in 
question and build the model object from there

## Old version of Fabric8 Kubernetes Java client

Villanova's operators are currently limited to the use of version 4.1.2 of the Fabric8 Kubernetes Java client. This
constraint is a result of using the Microbeans operators framework. We are actively looking at migrating to the
more active JVM Operators framework.

## HTTP 409 and 422 errors on 'edit'

During development of the Villanova Kubernetes controllers, it was found that the `metadata.resourceVersion` of 
a custom resource can at times go out of sync with what it needs to be for an updated. This tends to happen
when an `HTTP PATCH` (e.g. using `DoneableResource.done()`) operation was issued on the resource in 
question, but with no differences compared  to the original resource. If you do encounter an 
`HTTP 409` or `HTTP 422` error code from Kubernetes, please  inspect previous edits to the 
resource to verify that the state has indeed changed. If no state changes can be found, 
put the necessary checks in place in your code to ensure that it does not issue the obsolete `HTTP PATCH` 
to the Kubernetes server

## HTTP 404 errors on 'edit' running against the Fabric8 Kubernetes Mock server


During our testing efforts, we have encountered a bug in the Fabric8 Kubernetes Mock server. When running an
`edit` operation against a custom resource whilst using the Mock server, it does not seem to find to resource
in question. It can however be found using the `list` operation. Currently there is no workaround for this problem.

