# jonas-miederer.infra

This repository contains the AWS infrastructure as code for [jonas-miederer.de](https://jonas-miederer.de/), defined using [AWS SAM](https://aws.amazon.com/serverless/sam/) and CloudFormation.

## Features

- S3 bucket for static website hosting
- CloudFront distribution with custom domain and ACM certificate
- Route53 DNS records (A, MX, TXT, CNAME)
- API Gateway for contact form submissions
- SNS topic and email notification integration
- Automated deployment via GitHub Actions ([.github/workflows/deploy.yml](.github/workflows/deploy.yml))
- Standalone us-east-1 ACM certificate stack ([certificate/template.yaml](certificate/template.yaml))
- Subdomains, each served from its own S3 bucket + CloudFront distribution (currently `doubletop.jonas-miederer.de`)

## Deployment

You can build and deploy the infrastructure using the [SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html):

```sh
sam build
sam deploy --guided
```

Parameters such as domain name, ACM certificate ARN, and notification email are provided via CLI or GitHub Actions.

## Certificate stack (us-east-1)

CloudFront `ViewerCertificate` ACM certificates must live in **us-east-1**, but the main stack
(`jonas-miederer-infra`) deploys to **eu-central-1**. The certificate therefore lives in a separate
stack defined in [certificate/template.yaml](certificate/template.yaml).

- Stack name: `jonas-miederer-certificate`, region `us-east-1`.
- Issues a wildcard certificate `*.jonas-miederer.de` with the apex `jonas-miederer.de` as a
  Subject Alternative Name, using `DNS` validation. The wildcard covers the apex site and **every**
  `*.jonas-miederer.de` subdomain, so new subdomains need no certificate changes.
- Deployed manually via the [.github/workflows/deploy-certificate.yml](.github/workflows/deploy-certificate.yml)
  workflow (`workflow_dispatch` only — it does not run on push).

### Certificate ARN hand-off

The certificate stack exports its ARN as the `CertificateArn` output. Copy that value into the
GitHub Actions secret **`ACMCERTIFICATEARN`**, which the main `deploy.yml` passes to the stack as the
`AcmCertificateArn` parameter (shared by the apex and every subdomain CloudFront distribution).

### Deploy order

1. Run `deploy-certificate.yml` (manual) and complete the ACM DNS validation. The validation records
   go into the Route53 hosted zone already managed by the main stack.
2. Copy the `CertificateArn` output into the `ACMCERTIFICATEARN` GitHub secret.
3. Deploy the main stack by pushing to `main` — this creates/updates the apex and subdomain S3
   buckets, CloudFront distributions, OAIs, bucket policies, and the DNS records.
4. Deploy each subdomain's app from its own repository (for Double Top, push to `main` in the
   [Double-Top](https://github.com/jonasmiederer/Double-Top) repository).

## Subdomains

Each subdomain of `jonas-miederer.de` is served from its own S3 bucket and CloudFront distribution,
all defined in the main `template.yaml` and all covered by the shared wildcard certificate. A
subdomain is a self-contained set of resources mirroring the main site:

- a private S3 bucket named after the subdomain (all public access blocked),
- a dedicated CloudFront Origin Access Identity,
- a CloudFront distribution (same cache policy, redirect-to-https, price class, and the shared
  `AcmCertificateArn`),
- a bucket policy granting the OAI read access,
- an A-alias record in the hosted zone pointing the subdomain at its distribution.

Each subdomain's static files are deployed separately, from that app's own repository.

### Current subdomains

| Subdomain | App repository | Resource prefix |
| --- | --- | --- |
| `doubletop.jonas-miederer.de` | [Double-Top](https://github.com/jonasmiederer/Double-Top) | `Doubletop*` |

### Adding a subdomain

1. In `template.yaml`, copy the `Doubletop*` resources (bucket, OAI, distribution, bucket policy)
   and rename them with a new prefix (e.g. `Foo*`), updating the alias and bucket name to
   `foo.${DomainName}`.
2. Add an A-alias record for the subdomain into the existing `RecordSetGroup` (HostedZoneId
   `Z2FDTNDATAQYW2`, the fixed CloudFront alias zone).
3. Add outputs for the new bucket name and distribution id if the app's deploy workflow needs them.
4. No certificate change is required — the wildcard already covers `*.jonas-miederer.de`.
5. In the app's repository, add a deploy workflow that syncs the static files to the new bucket and
   invalidates the distribution (looking it up by its alias).

## Local Testing

```sh
sam local start-api
```

## Cleanup

To remove the stack:

```sh
aws cloudformation delete-stack --stack-name jonas-miederer-infra
```

## License
