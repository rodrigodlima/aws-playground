# Amazon S3 Files Access PoC

## Overview

This Proof of Concept explores [Amazon S3 Files Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files.html), a recently released AWS feature that exposes Amazon S3 objects as a shared file system to AWS compute resources. It enables standard file system operations (read, write, list) against S3 data through a local mount path, removing the need to interact with the S3 API directly from application code.

## Goal

The primary goal is to validate Amazon S3 Files Access as a file system layer for AWS Lambda functions, observing how it behaves under a real workflow and how it integrates with infrastructure managed declaratively from a Kubernetes control plane.

Beyond the feature itself, this PoC also serves as an end-to-end exercise in GitOps-driven AWS provisioning: defining cloud infrastructure as Kubernetes resources, reconciling them through a controller, and delivering them via a continuous deployment pipeline.

## Scope

The PoC will combine the following components:

- **Kind (Kubernetes in Docker)** — Local Kubernetes cluster used as the control plane for the entire setup. All other tooling runs on top of it.
- **ArgoCD** — GitOps continuous delivery operator responsible for reconciling the desired state of the cluster (and the AWS resources it manages) from this Git repository.
- **Crossplane** — Kubernetes-native infrastructure-as-code engine used to declare and provision AWS resources (S3 buckets, IAM roles, Lambda functions, S3 Files Access mounts) directly from Kubernetes manifests.
- **AWS Lambda** — Compute target for the experiment. Lambda functions will mount S3 Files Access and exercise the file system interface.
- **Amazon S3 Files Access** — The feature under evaluation. It will be configured against an S3 bucket and mounted into the Lambda runtime so the function can access objects as regular files.

## What this PoC intends to demonstrate

- That a non-trivial AWS workload can be fully described and deployed through a GitOps + Crossplane pipeline running on a local Kind cluster.
- That AWS Lambda can consume S3 data through the new S3 Files Access mount as a drop-in file system, including the operational characteristics (permissions, mount lifecycle, IAM model) that come with it.
- The trade-offs of using S3 Files Access compared to direct S3 SDK calls in a serverless context.

## Before vs. after comparison

To make the value of S3 Files Access concrete, the PoC will include a side-by-side comparison of the same use case implemented two ways:

- **Before** — a Lambda function that handles files using the traditional approach: downloading objects from S3 through the AWS SDK into `/tmp`, operating on them locally, and uploading results back. This highlights the boilerplate, the size and lifetime constraints of the Lambda ephemeral storage, and the explicit S3 API calls the developer has to manage.
- **After** — the equivalent Lambda function rewritten to read and write directly against the S3 Files Access mount as if it were a local directory, with no SDK calls in the data path.

The goal of this comparison is to surface the differences in code complexity, runtime behavior, and operational footprint between both approaches.

# Install resources

Exec the script create-poc.sh

## Access ArgocdCD

```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Access https://localhost:8080

## Get the Argocd password

```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
