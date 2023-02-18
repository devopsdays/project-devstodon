# Mastodon on AWS

Forked from [widdix/mastodon-on-aws/](https://github.com/widdix/mastodon-on-aws/). Thank you for sharing this fantastic project. 

The architecture consists of the following building blocks.

* Application Load Balancer (ALB)
* ECS and Fargate
* RDS Aurora Serverless
* ElastiCache (Redis)
* S3
* SES
* CloudWatch
* IAM
* KMS
* Route 53
* CloudFront

![Mastodon on AWS: Architeture](architecture.png)

Check out Cloudonaut's blog post [Mastodon on AWS: Host your own instance](https://cloudonaut.io/mastodon-on-aws/) for more details.

## Prerequisites

- AWS account

- A top-level or sub domain where you are able to configure a `NS` record to delegate to the Route 53 nameservers is required. For example, you could register a domain with Rout 53 or use an existing domain and add an `NS` record to the hosted zone.

- [Docker Desktop](https://www.docker.com/get-started/) on your local machine to generate the required secrets.

- Local [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) configured to AWS account.

## Installation

### Deploy the infrastructure

To generate the required secrets and keys use the following commands.

```
# Start Docker container locally
$ docker run -it tootsuite/mastodon:latest sh

# Generate SECRET_KEY_BASE
$ bundle exec rake secret
758a3b431265776b9ab55910890162bb84aec0617724ca611475c3a774965f2d0aca183091d3c1a84ff3640cf7cc438c559034a2735253ee895b7a2308ac450c

# Generate OTP_SECRET
$ bundle exec rake secret
c528b5cbb0236e4b0c2fe38a6d7ed1edc5fa12608c67a45690e225f005bad8bfbabfa99f7b83cb9c0981ba8fcc5fd76c68918d9bc854bd158c2c23fd6df89abc

# Generate VAPID_PRIVATE_KEY and VAPID_PUBLIC_KEY
$ bundle exec rake mastodon:webpush:generate_vapid_key
VAPID_PRIVATE_KEY=am3vlPBGQGv7Rl3xOKXSv7lRYyWfZITItb88FXX9IOs=
VAPID_PUBLIC_KEY=BMGkIr1PaK4v7Kut7q7eoHtWxu9gEBQ5BeV28xOIR9c9VIvDWvOViTn1SV5G2LIEFGWo0f1dQka-UynR58WMn2Y=
```

IaC based on [cfn-modules](https://github.com/cfn-modules/docs).

```
$ npm install
$ aws cloudformation package --template-file mastodon.yaml --s3-bucket <S3_BUCKET> --output-template-file packaged.yml
$ aws cloudformation deploy --template-file packaged.yml --stack-name dod-mastodon-on-aws --capabilities CAPABILITY_IAM --parameter-overrides "DomainName=devopsdays.social" "SecretKeyBase=<SECRET_KEY_BASE>" "OtpSecret=<OTP_SECRET>" "VapidPrivateKey=<VAPID_PRIVATE_KEY>" "VapidPublicKey=<VAPID_PUBLIC_KEY>" "Spot=false" "DatabaseAllocatedStorage=10" "AlertingEmail=yvo@devopsdays.org"

```

### Configure the domain name

Note: The deploy will hang until you perform the steps below as there is a Amazon Certificate Manager (ACM) step in the deployment to generate the appropriate TLS certificates. 

By creating the CloudFormation stack, you also created a Route 53 hosted zone for the `DomainName` you specified as a parameter.

1. Open [Route 53](https://console.aws.amazon.com/route53/v2/home#Dashboard) via the AWS Management Console.
1. Select `Hosted zones` from the sub navigation.
1. Search and open the hosted zone with the domain name of your Mastodon instance (`DomainName` parameter).
1. Open [Cloudflare](https://dash.cloudflare.com) via a web browser.
1. Select the devopsdays.social domain under DNS
1. Copy the values from Route53, the ALB is using an extended `A` record type which allows aliases on Route53, under Cloudflare use the `CNAME` record. 

### Enable the admin user / Accessing tootctl

Use the following instructions to access the Mastodon CLI:

1. Open Elastic Container Service (ECS) via the AWS Management Console.
1. Select the ECS cluster with the name prefixed with the name of your CloudFormation stack (e.g., `mastodon-on-aws-*`).
1. Note down the full name of the cluster (e.g., `mastodon-on-aws-Cluster-1NHBMI9NL62QP-Cluster-pkxgiUVXxLC7`).
1. Select the `Tasks` tab.
1. Search for a task with status `Running` and a task definition containing `*-WebService-*` in its name.
1. Note down the task ID (e.g., `a752b99a4cf843ce8a957c374fc98abf`).
1. Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

Use the following command to connect with the container running the Ruby on Rails (Web) application. Replace <CLUSTER_NAME> with the name of your ECS cluster and <TASK_ID> with the ID of a running ECS task.

```
aws ecs execute-command --cluster <CLUSTER_NAME> --container app --command /bin/bash --interactive --task <TASK_ID>
```

After the session got established you are ready to use the [tootctl](https://docs.joinmastodon.org/admin/tootctl/).

After signing up, you will need to use the command line to give your newly created account admin privileges. Replace `<USERNAME>` with your user name (e.g., `yvo`).

```
RAILS_ENV=production bin/tootctl accounts modify <USERNAME> --role Owner
```

### Activating SES

In case you haven't used SES in your AWS account before, you most likely need to request production access for SES. This is required so that your Mastodon instance is able to send emails (e.g., registration, forgot password, and many more). See [Moving out of the Amazon SES sandbox](https://docs.aws.amazon.com/ses/latest/dg/request-production-access.html) to learn more.

## Update

Here is how you update your infrastructure.

1. Open CloudFormation via the AWS Management Console.
1. Select the CloudFormation stack which is named `mastodon-on-aws` in case you created the stack with our defaults.
1. Press the `Edit` button.
1. Choose the option `Replace current template` with `https://s3.eu-central-1.amazonaws.com/mastodon-on-aws-cloudformation/latest/quickstart.yml`.
1. Go through the rest of the wizard and keep the defaults.


