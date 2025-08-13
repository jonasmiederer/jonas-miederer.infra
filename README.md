# jonas-miederer.infra

This repository contains the AWS infrastructure as code for [jonas-miederer.de](https://jonas-miederer.de/), defined using [AWS SAM](https://aws.amazon.com/serverless/sam/) and CloudFormation.

## Features

- S3 bucket for static website hosting
- CloudFront distribution with custom domain and ACM certificate
- Route53 DNS records (A, MX, TXT, CNAME)
- API Gateway for contact form submissions
- SNS topic and email notification integration
- Automated deployment via GitHub Actions ([.github/workflows/deploy.yml](.github/workflows/deploy.yml))

## Deployment

You can build and deploy the infrastructure using the [SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html):

```sh
sam build
sam deploy --guided
```

Parameters such as domain name, ACM certificate ARN, and notification email are provided via CLI or GitHub Actions.

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
