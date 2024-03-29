---
title: "Policy Enforcement with Gatekeeper and KCC"
date: 2022-04-27
draft: false
summary: Using Gatekeeper to enforce policy on infrastructure in Config Controller with Config Connector resources.
---

## Policy Enforcement with Gatekeeper and KCC

In my [last post](/posts/fun-with-gitops/) we deployed a GKE Cluster using Config Controller and Kubernetes Config Connector Resources. For this Post I wanted to look at enforcing policy in Config Controller so we can prevent non-compliant resources from being deployed. What we'll be doing is taking a look at how to prevent resources from being deployed in unapproved regions.

This would align with [Guardrail # 5]((https://github.com/canada-ca/cloud-guardrails/blob/master/EN/05_Data-Location.md)) of the GC's Cloud Guardrails [Framework](https://github.com/canada-ca/cloud-guardrails) which has the objective of "Establish policies to restrict GC sensitive workloads to approved geographic locations.". What this means in our case is we'll be looking to ensure we can only deploy our resources to a Canadian Region (`northamerica-northeast1`, `northamerica-northeast2`). This policy can also be enforced via an Organization with [Organization Policy Resource Location Restriction](https://cloud.google.com/resource-manager/docs/organization-policy/defining-locations). 

So why if I have this organization policy would I want to enforce is elsewhere? In my opinion this policy is great but only enforces at deploy time, so when you run a `gcloud container clusters create` command, for example, this will start the creation of the desired service and after a while you should get an error which is great, it worked! 

We can enchance this and save you that time spend waiting for the system to let you know things are misconfigured by moving some validation to the front end of the process ([shift left](https://en.wikipedia.org/wiki/Shift-left_testing)).

By doing this (and not replacing the org policies which are still vital) we can get those alerts sooner in the process so we react faster and not waste time waiting for bad configurations to be deployed. 

We can use [Policy Controller](https://cloud.google.com/anthos-config-management/docs/concepts/policy-controller) to help us achieve this early testing and continual testing in Config Controller. Policy Controller is a policy enforcement engine based on [OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/website/docs/) which we can use to write and enforce policy. Without getting too technical how this works is when the request gets made Policy Controller will inspect your configuration and give you a response before an attempt to create the resource is made. If you want to learn more checkout out this documention on [admission controllers](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/).

Let's look at how we can do this with our GKE configuration from last time, I won't put the whole config here but just the relevent snippet. The relevent field we're going to be concerned with here is the `location` key.

```
spec:
  description: Test Cluster for Falco Sidekick
  location: northamerica-northeast1 <--- We want to look at this field
  initialNodeCount: 1
  defaultMaxPodsPerNode: 110
```

In this configuration we're deploying a GKE cluster to `northamerica-northeast1` which is great, I configured it correctly! But what happens if I change that or someone else tries to deploy to another region, for example.

```
spec:
  description: Test Cluster for Falco Sidekick
  location: us-east1
  initialNodeCount: 1
  defaultMaxPodsPerNode: 110
```

How do we prevent that from happening using Policy Controller?

First we'll need to write the policy and deploy it to our Config Controller instance, luckily for you I went ahead and did that for us!

## Policy Overview

The way that Policy Controller works is that we have 2 files a `template` and a `constraint`. The `template` is the rule itself and the `constraint` is an instance of that policy. For example your policies can take variable inputs which allows for multiple instances of the `constraint` to be created with different restrictions depending on the inputs.

For example this `constraint` implements the `template`. This makes things easier for policy writers and platform admins who need to do the implementation.

Constraint

```
parameters:
    locations:
        - "northamerica-northeast1"
        - "northamerica-northeast2"
        - "global"
```

Template

```
violation[{"msg": message}] {
    asset := input.review.object
    not location_match(asset.spec.location, input.parameters.locations)
    not allowedResource(asset.kind, input.parameters.allowedServices)
    message := sprintf("Guardrail # 5: Resource %v ('%v') is located in '%v' when it is required to be in '%v'", [asset.kind, asset.metadata.name, asset.spec.location, input.parameters.locations])
}
```

## Policy Deployment!

First up we'll need to create the config controller cluster. This is abbreviated from the [Setup Guide](https://cloud.google.com/anthos-config-management/docs/how-to/config-controller-setup)

1. Create the required network
```
gcloud compute networks create default --subnet-mode=custom
```

2. Enabe the APIs
```
gcloud services enable krmapihosting.googleapis.com \
    container.googleapis.com \
    cloudresourcemanager.googleapis.com
```

3. Create the Controller instance
```
gcloud anthos config controller create policy-enforced --location northamerica-northeast1
```

The creation of the controller should be about 20 mins.

Once the Config Controller instance is finished provisioning we can proceed with the next step. If you want to get a head start you can open up a new terminal tab and do the next step.

4. Download the Demo Config Files

Now lets get setup with the configuration files which are stored in github with the following `kpt` command. This will download the configuration files from the projects repository on [github](github.com/cartyc/policy-blog-resources).

```
kpt pkg get https://github.com/cartyc/policy-blog-resources.git
```

5. Deploy the Policy Files.

Before we move onto the infrastructure deployment we'll need to get our policy files up and running.

First we'll deploy the templates with the following command:

```
kubectl apply -f infrastructure-policy/templates
```

Once that is deployed we can safely deploy the constraints.
```
kubectl apply -f infrastructure-policy/constraints
```

To view the newly deployed constraints in the instance use `kubectl get constraints` and you will get output similar to the below.

```
NAME                                                              ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
limitegresstraffic.constraints.gatekeeper.sh/limitegresstraffic                        0

NAME                                                  ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
datalocation.constraints.gatekeeper.sh/datalocation                        0
```

## Infrastructure Deployment

All right now that we can try to deploy our GKE infrastructure whose configs exist in the `infrastructure` folder. We'll be trying to deploy a new network, subnet and a GKE cluster. Lets give it a try and see what happens.

```
kubectl apply -f infrastructure
```

Wait we're getting some errors!

```
computenetwork.compute.cnrm.cloud.google.com/demo-net created
Error from server (Forbidden): error when creating "infrastructure/gke.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [limitegresstraffic] GKE is not Private data-location-demo
[datalocation] Guardrail # 5: Resource ContainerCluster ('data-location-demo') is located in 'us-east1' when it is required to be in '["northamerica-northeast1", "northamerica-northeast2", "global"]'
Error from server (Forbidden): error when creating "infrastructure/gke.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [datalocation] Guardrail # 5: Resource ComputeSubnetwork ('demo-subnet') is located in 'us-east1' when it is required to be in '["northamerica-northeast1", "northamerica-northeast2", "global"]'
```

So it looks like the network deployed ok but we had trouble with the subnet and cluster. Let's dig in a little more. 

The first error is a bit of a surprise as this isn't a data location issue but a networking related one, and saying that our cluster isn't private. Let's take a peak at the configuration file. If we scroll down in `infrastructure/gke.yaml` we'll eventually come across a commented out section of yaml (starting line 52).

```
  verticalPodAutoscaling:
    enabled: true
  # privateClusterConfig:
  #   enablePrivateEndpoint: false
  #   enablePrivateNodes: true
  #   masterIpv4CidrBlock: 172.16.0.0/28
  nodeConfig:
```

What's happening here is we are being non-compliant with [Cloud Guardrail # 9](https://github.com/canada-ca/cloud-guardrails/blob/master/EN/09_Network-Security-Services.md) Network Security Services and one of the validation outcomes is limiting the number of public IPs used. So we have a policy in place to enforce private cluster usage and reduce the number of public IPs used.

Next we have two errors we expected to get with `datalocation` related errors. We're being told two things:

```
Resource ContainerCluster ('data-location-demo') is located in 'us-east1'
```

and 

```
Resource ComputeSubnetwork ('demo-subnet') is located in 'us-east1' 
```

This is expected based on the policy we looked at earlier and both of these resources are being deployed in a non-approved region. We'll fix this in a couple sections.

## Testing Locally with Kpt

You'll notice with this method we need to wait for the Config Controller instance to tell us that something is wrong, this is great but I think we can do better!

Using the `kpt` packaging tool we can run the policies against our code locally using the [Gatekeeper](https://catalog.kpt.dev/gatekeeper/v0.2/) function. I've preconfigured this to run in the `Kptfile` in the root of the directory. To test this run `kpt fn render` and you should get output similar to this:

```
[error] container.cnrm.cloud.google.com/v1beta1/ContainerCluster/config-control/data-location-demo: GKE is not Private data-location-demo violatedConstraint: limitegresstraffic
[error] container.cnrm.cloud.google.com/v1beta1/ContainerCluster/config-control/data-location-demo: Guardrail # 5: Resource ContainerCluster ('data-location-demo') is located in 'us-east1' when it is required to be in '["northamerica-northeast1", "northamerica-northeast2", "global"]' violatedConstraint: datalocation
[error] compute.cnrm.cloud.google.com/v1beta1/ComputeSubnetwork/config-control/demo-subnet: Guardrail # 5: Resource ComputeSubnetwork ('demo-subnet') is located in 'us-east1' when it is required to be in '["northamerica-northeast1", "northamerica-northeast2", "global"]' violatedConstraint: datalocation
```

Great! These are the same errors as we got from Config Controller so i think we can be pretty confident that we can go ahead and start fixing the errors and testing locally now. This is really nice because we now don't need to have a running config controller instance to validate our configs and can either do this testing locally or within a CI/CD pipeline without needing to add extra infrastructure.

## Fixing the Issues

To resolve these issues you will need to change the `location` value for the GKE instance and `region` in the subnetwork resource. Thankfully the error is telling us what regions are allowed to be used `'["northamerica-northeast1", "northamerica-northeast2", "global"]'`, choose either `"northamerica-northeast1", "northamerica-northeast2"`.

To fix the private cluster error uncomment the privatecluster section.

First run the tests locally with `kpt` to make sure things are fixed. A passing result will look like.

```
package "policy-blog-resources": 
[RUNNING] "gcr.io/kpt-fn/kubeval:v0.2"
[PASS] "gcr.io/kpt-fn/kubeval:v0.2" in 5s
[RUNNING] "gcr.io/kpt-fn/gatekeeper:v0.2.1"
[PASS] "gcr.io/kpt-fn/gatekeeper:v0.2.1" in 1.3s

Successfully executed 2 function(s) in 1 package(s).
```

 If you don't get any errors you can go ahead and apply the configs and you should get a shiny new Private GKE cluster in a matter of minutes!

## What's Next?

What I've gone through in this post is a pretty limited case but this could be extended to things like ensuring that specific labels or annotations are included with resources for example departments, cost centers, emergency contacts, etc or specific naming conventions on resources, even ITSG related policies.

These are some awesome resources for additional reading
- https://cloud.google.com/anthos-config-management/docs/concepts/policy-controller
- https://www.openpolicyagent.org/docs/latest/
- https://cloud.google.com/anthos-config-management/docs/how-to/using-cis-k8s-benchmark