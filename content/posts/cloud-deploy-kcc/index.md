---
title: "Cloud Deploy Kubernetes Config Connector"
date: 2022-03-06T09:01:46-05:00
draft: true
summary: Deploying KCC resources to Config Controller using Cloud Deploy.
---

## Why, what is Cloud Deploy?

- New CD tool for deploying to GKE (Cloud Run in Preview).
- Goal is too deploy to K8s without needing to touch Kubernetes Directly
- Gives a nice UI for the release process.
- Approvals.
- Gives more of a familiar click to deploy feel

## Create Config Controller Instance

Like in the last post we'll need to get a Config Controller instance up and running. Skip this step if you already have one up and running.

Before we create the instance we'll need to enable the required APIs.

```
gcloud services enable krmapihosting.googleapis.com \
    container.googleapis.com \
    cloudresourcemanager.googleapis.com
```

Once that is complete we can create the instance. This should take about 15 to 20 mins.

```
gcloud anthos config controller create main --location us-east1
```

WHen it comes up we'll need to set some IAM Permssions to the Service Account it's using. We'll give it only the permissions required to so we do our best to stick to the rule of least privilege.

```
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:connector@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/container.clusterAdmin"
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:connector@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/pubsub.admin"
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:connector@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/source.admin"
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:connector@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/iam.securityAdmin"
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:connector@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/gkehub.admin"
gcloud iam service-accounts add-iam-policy-binding \
connector@${PROJECT_ID}.iam.gserviceaccount.com \
    --member="serviceAccount:${PROJECT_ID}.svc.id.goog[cnrm-system/cnrm-controller-manager]" \
    --role="roles/iam.workloadIdentityUser"
```

Next up we'll download the example config files and start configuring the files for cloud deploy.

## Get example configurations

```
kpt package get ...
```



## Configure Cloud Deploy

```
gcloud deploy apply --file clouddeploy.yaml --region=northamerica-northeast1
```

## Release the kraken

```
gcloud deploy releases create initial-release \
  --region=northamerica-northeast1 \
  --delivery-pipeline=kcc-infra \
  --gcs-source-staging-dir=gs://config-controller-host-336713_alpha/source
```

Additional Reading
- Skaffold
- Cloud Deploy