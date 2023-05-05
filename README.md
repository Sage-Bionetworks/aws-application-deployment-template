# aws-application-deployment-template

CDK-based template for deploying a containerized application to AWS

## Perequisites

AWS CDK projects require some bootstrapping before synthesis or deployment.
Please review the [bootstapping documentation](https://docs.aws.amazon.com/cdk/v2/guide/getting_started.html#getting_started_bootstrap)
before development.

## Development

The `cdk.json` file tells the CDK Toolkit how to execute your app.

This project is set up like a standard Python project.  The initialization
process also creates a virtualenv within this project, stored under the `.venv`
directory.  To create the virtualenv it assumes that there is a `python3`
(or `python` for Windows) executable in your path with access to the `venv`
package. If for any reason the automatic creation of the virtualenv fails,
you can create the virtualenv manually.

To manually create a virtualenv on MacOS and Linux:

```
$ python3 -m venv .venv
```

After the init process completes and the virtualenv is created, you can use the following
step to activate your virtualenv.

```
$ source .venv/bin/activate
```

If you are a Windows platform, you would activate the virtualenv like this:

```
% .venv\Scripts\activate.bat
```

Once the virtualenv is activated, you can install the required dependencies.

```
$ pip install -r requirements.txt
```

At this point you can now synthesize the CloudFormation template for this code.
An environment context is rquired, check [cdk.json](cdk.json) for available contexts.

```
$ cdk synth --context env=dev
```

## Testing

### Static Analysis
As a pre-deployment step we syntatically validate the CDK json, yaml and
python files with [pre-commit](https://pre-commit.com).

Please install pre-commit, once installed the file validations will
automatically run on every commit.  Alternatively you can manually
execute the validations by running `pre-commit run --all-files`.

### Python Tests
Tests are available in the tests folder. Execute the following to run tests:

```
python -m pytest tests/ -s -v
```

## Secrets

We use the [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
to store secrets for this project.  An AWS best practice is to create secrets
with a unique ID to prevent conflicts when multiple instances of this project
is deployed to the same AWS account.  Our naming convention is
`<cfn stack name>/<environment id>/<secret name>`.  The template in this repo' uses
the secret name, `ecs`, so an example Secrets Manager name is `myapp-dev-DockerFargateStack/dev/ecs`.


## Deployment from GitHub to AWS

To allow the GitHub action in your repository to deploy to AWS, submit a
PR like [this](https://github.com/Sage-Bionetworks-IT/organizations-infra/pull/771/files),
customizing the `StackName`, `Repositories` (name), and the `Account`(s) to
in which the infrastructure will be deployed. More on GitHub OIDC integration
[here](https://github.com/Sage-Bionetworks-IT/organizations-infra/tree/master/org-formation/650-identity-providers).

Once the PR is merged, an IAM role will be created in each AWS account listed in the PR.
Put the ARNs for the roles in the `ROLE_TO_ASSUME` field of `main.yml`, which
allows switching between development and production based on the git branch.

## VPC CIDR

Select a unique IP address range for the VPC (CIDR) following the instructions
[here](https://sagebionetworks.jira.com/wiki/spaces/IT/pages/2850586648/Setup+AWS+VPC).
Enter the range as the `VPC_CIDR` parameter in the file `cdk.json`.


## DNS and Certificates

Obtain the ARN of the ACM Certificate by following the instructions
[here](https://sagebionetworks.jira.com/wiki/spaces/IT/pages/2859302913/Admin+Tasks+for+CDK+Applications)
Set the value of `ACM_CERT_ARN` context variable to the ARN obtained above.

Finally, a DNS CNAME must be created in org-formation after the initial
deployment of the application to make the application available at the desired
URL. The CDK application exports the DNS name of the Application Load Balancer
to be consumed in org-formation. [An example PR setting up a CNAME](https://github.com/Sage-Bionetworks-IT/organizations-infra/pull/739).

Find the `LoadBalancerDNS` name: Navigate to the deployed stack in the AWS CloudFormation
console and click on the `Outputs` tab.  On the row whose key is `LoadBalancerDNS` look for
the value in the `ExportName` column, e.g., `dca-dev-DockerFargateStack-LoadBalancerDNS`.
Now use the name in the `TargetHostName` definition, for example:

```
TargetHostName: !CopyValue [!Sub 'dca-dev-DockerFargateStack-LoadBalancerDNS', !Ref DnTDevAccount]
```

(You would also replace `DnTDevAccount` with the name of the account in which the application is deployed.)


## Debugging

Generally CDK deployments will create cloudformation events during a CDK deploy.
The events can be viewed in the AWS console under the cloudformation service page.
Viewing those events will help with errors during a deployment.  Below are cases
where it might be difficult to debug due to misleading or insufficient error
messages from AWS

### Missing Secrets

Each new environment (dev/staging/prod/etc..) may require adding secrets.  If a
secret is not created for the environment you may get an error with the following
stack trace..
```
Resource handler returned message: "Error occurred during operation 'ECS Deployment Circuit Breaker was triggered'." (RequestToken: d180e115-ba94-d8a2-acf9-abe17a3aaed9, HandlerErrorCode: GeneralServiceException)
    new BaseService (/private/var/folders/qr/ztb40vmn2pncyh8jpsgfnrt40000gp/T/jsii-kernel-4PEWmj/node_modules/aws-cdk-lib/aws-ecs/lib/base/base-service.js:1:3583)
    \_ new FargateService (/private/var/folders/qr/ztb40vmn2pncyh8jpsgfnrt40000gp/T/jsii-kernel-4PEWmj/node_modules/aws-cdk-lib/aws-ecs/lib/fargate/fargate-service.js:1:967)
    \_ new ApplicationLoadBalancedFargateService (/private/var/folders/qr/ztb40vmn2pncyh8jpsgfnrt40000gp/T/jsii-kernel-4PEWmj/node_modules/aws-cdk-lib/aws-ecs-patterns/lib/fargate/application-load-balanced-fargate-service.js:1:2300)
    \_ Kernel._create (/private/var/folders/qr/ztb40vmn2pncyh8jpsgfnrt40000gp/T/tmpqkmckdm2/lib/program.js:9964:29)
    \_ Kernel.create (/private/var/folders/qr/ztb40vmn2pncyh8jpsgfnrt40000gp/T/tmpqkmckdm2/lib/program.js:9693:29)
    \_ KernelHost.processRequest (/private/var/folders/qr/ztb40vmn2pncyh8jpsgfnrt40000gp/T/tmpqkmckdm2/lib/program.js:11544:36)
    \_ KernelHost.run (/private/var/folders/qr/ztb40vmn2pncyh8jpsgfnrt40000gp/T/tmpqkmckdm2/lib/program.js:11504:22)
    \_ Immediate._onImmediate (/private/var/folders/qr/ztb40vmn2pncyh8jpsgfnrt40000gp/T/tmpqkmckdm2/lib/program.js:11505:46)
    \_ processImmediate (node:internal/timers:464:21)
```
