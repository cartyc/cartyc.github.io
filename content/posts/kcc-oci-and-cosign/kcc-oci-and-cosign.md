---
title: "KCC Oci and Cosign"
date: 2022-11-16T20:45:56-05:00
draft: false
---

# Signing OCI KCC Artifacts Using Cosign

I've been meaning to try out [cosign](https://github.com/sigstore/cosign) to sign OCI artifacts for a while now and recently had a chance to give it a whirl and well it's pretty awesome.

My usecase I wanted to apply this too was to use the new OCI support in [Config Sync](https://cloud.google.com/anthos-config-management/docs/how-to/sync-oci-artifacts-from-artifact-registry) to deploy [Kubernetes Config Connector](https://cloud.google.com/config-connector/docs/overview) resources and I thought it would be really interesting to start signing those OCI artifacts for added security and well applying that to infrastructure was really intriguing and exciting!

While this is a very GCP centric view on things I think it could be equally applied to [crossplane](https://github.com/crossplane/crossplane) resources as well!

Before we start I am assuming you have a GCP project and are the owner of said project.

To kick things off we'll set some environment variables that we'll use through the examples. We'll be creating some keys in KMS for use during signing and and Artifact Registry to store the OCI artifact and the `cosign` signature.

## Set env variables

```
PROJECT_ID=<targetprojectid>
KEYRING=<keyring>
KEYNAME=<keyname>
REGION=<region>
REGISTRY_NAME=<registry-name>
CONTAINER_NAME=<container-name>
```

Next up let's enable the required services.

## Enable Required Services
```
gcloud services enable artifactregistry.googleapis.com
gcloud services enable cloudkms.googleapis.com
```

## create keyring 

For this part we'll need to create a keyring and then a key that `cosign` will use to sign the artifact.

```
gcloud kms keyrings create $KEYRING \
    --location $REGION
```

## Create Key
```
gcloud kms keys create $KEYNAME \
    --keyring $KEYRING \
    --location northamerica-northeast1 \
    --purpose "asymmetric-signing" \
    --default-algorithm=ec-sign-p256-sha256
```

## Create Artifact Registry

And the last piece of infrastructure we'll need, the registry.

```
gcloud artifacts repositories create ${REGISTRY_NAME} \
    --repository-format=docker \
    --location $REGION
```

and login to the registry so we can push to it later

```
gcloud auth configure-docker ${REGION}-docker.pkg.dev
```

## Install Oras and Cosign

Finally we can do the fun stuff. This little snippet will download and install `oras` which we'll use to push the file to Artifact Registry and `cosign`.

```
echo "Install Cosign"
wget https://github.com/sigstore/cosign/releases/download/v1.13.1/cosign-linux-amd64
chmod +x cosign-linux-amd64
mv cosign-linux-amd64 cosign
echo "Install Oras"
curl -LO https://github.com/oras-project/oras/releases/download/v0.16.0/oras_0.16.0_linux_amd64.tar.gz
mkdir -p oras-install/
tar -zxf oras_0.16.0_*.tar.gz -C oras-install/
mv oras-install/oras /usr/local/bin/
rm -rf oras_0.16.0_*.tar.gz oras-install/
```

##  Create some YAML and tar it

For my nefarious use case I want to deploy a GKE Autopilot cluster thats using Anthos Service Mesh, we won't actually be deploying this but if you want to you can check out [config controller](https://cloud.google.com/anthos-config-management/docs/concepts/config-controller-overview) (one of my favorite services).

First let's make a new folder so we can keep things neat and tidy.

```
mkdir cosign-oci/
```

And then we'll `cat` the configs into a file called `gke.yaml`.

```
cat <<EOF > ./cosign-oci/gke.yaml
apiVersion: container.cnrm.cloud.google.com/v1beta1
kind: ContainerCluster
metadata:
  name: oci-cosigned
spec:
  description: An OCI'd autopilot cluster.
  enableAutopilot: true
  location: $REGION
  releaseChannel:
    channel: REGULAR
---
apiVersion: gkehub.cnrm.cloud.google.com/v1beta1
kind: GKEHubFeatureMembership
metadata:
  name: oci-cosigned-mesh-membership
spec:
  projectRef:
    external: $PROJECT_ID
  location: global
  membershipRef:
    name: oci-cosigned-hub
  featureRef:
    name: servicemesh
  mesh:
    management: MANAGEMENT_AUTOMATIC
---
apiVersion: gkehub.cnrm.cloud.google.com/v1beta1
kind: GKEHubMembership
metadata:
  name: oci-cosigned-hub
spec:
  location: global
  authority:
    issuer: 'https://container.googleapis.com/v1/projects/${PROJECT_ID}/locations/${REGION}/clusters/oci-cosigned'
  description: oci-cosigned Cluster
  endpoint:
    gkeCluster:
      resourceRef:
        name: oci-cosigned
EOF
```

Now that we have that done all that's left is to `tar` the file and tell `oras` to push it to artifact registry.

```
tar -cf ./example.tar ./cosign-oci/gke.yaml
```

Now we should be good to push the file to the registry!

I've wrapped the command like this so we can quickly grab the `sha` digest that `oras` produces so we can drop that into the `cosign` command easier.

```
SHA=$(oras push ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REGISTRY_NAME}/${CONTAINER_NAME} example.tar | grep Digest: | cut -f2 -d" ")
```

## Set up Cosign

For `cosign` to work right it will need you to set the `application-default` credentials.
```
gcloud auth application-default login
```

Once logged in you can go ahead and create the keypair we'll use for signing.

```
cosign generate-key-pair --kms gcpkms://projects/${PROJECT_ID}/locations/${REGION}/keyRings/${KEYRING}/cryptoKeys/${KEYNAME}
```

and finally sign the artifact.

```
cosign sign --key gcpkms://projects/${PROJECT_ID}/locations/${REGION}/keyRings/${KEYRING}/cryptoKeys/${KEYNAME} ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REGISTRY_NAME}/${CONTAINER_NAME}@${SHA}
```

## Verify what was created

Now that we've signed the artifact, how do we verify it? Luckily it's pretty simple in that we just need to run the following command from a place that has access to view the key we created earlier and the artifact registry.

```
cosign verify --key gcpkms://projects/${PROJECT_ID}/locations/${REGION}/keyRings/${KEYRING}/cryptoKeys/${KEYNAME} ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REGISTRY_NAME}/${CONTAINER_NAME}@${SHA}
```

Voilia you've now signed your KCC infra with cosign!

## Set Up Cosign and Cloud Build

For additional fun and profit I set up a test cloud build pipeline that runs through this.

  Cloud Build Permissions
Key verifier/signer
key viewer


```
steps:
- name: ubuntu
  script: |
    tar -cf /workspace/example.tar environments
- name: ubuntu
  env:
    - 'PROJECT_ID=$PROJECT_ID'
    - 'COMMIT_SHA=$COMMIT_SHA'
  script: |
    apt update
    apt install wget curl -y
    echo "Install Cosign"
    wget https://github.com/sigstore/cosign/releases/download/v1.13.1/cosign-linux-amd64
    chmod +x cosign-linux-amd64
    mv cosign-linux-amd64 cosign
    echo "Install Oras"
    curl -LO https://github.com/oras-project/oras/releases/download/v0.16.0/oras_0.16.0_linux_amd64.tar.gz
    mkdir -p oras-install/
    tar -zxf oras_0.16.0_*.tar.gz -C oras-install/
    mv oras-install/oras /usr/local/bin/
    rm -rf oras_0.16.0_*.tar.gz oras-install/
    SHA=$(oras push ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REGISTRY_NAME/${CONTAINER_NAME} example.tar | grep Digest: | cut -f2 -d" ")
    echo "SHA"
    echo $SHA
    ./cosign generate-key-pair --kms gcpkms://projects/${PROJECT_ID}/locations/${REGION}/keyRings/${KEYRING}/cryptoKeys/${KEYNAME}
    ./cosign sign --key gcpkms://projects/${PROJECT_ID}/locations/${REGION}/keyRings/${KEYRING}/cryptoKeys/${KEYNAME} ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REGISTRY_NAME/${CONTAINER_NAME@${SHA}
```