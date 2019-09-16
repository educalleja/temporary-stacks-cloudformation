

Temporary CF stacks on http.
----------------------------

These set of cloudformation templates allow creating stacks via http requests or cli commands.
The created stacks will be deleted automatically after a defined timeout. This is especially useful when we want services to be running during a limited period of time. 

When using this infrastructure, temporary stacks are created on HTTP POST request and then get deleted automatically after a period of time.

**Step 1**. Build the support infrastructure to create cloudformation stacks using an Rest API.

```bash
$ aws cloudformation create-stack --stack-name createDeleteCFStacks --template-body file://createStacks.yml --capabilities CAPABILITY_NAMED_IAM 
```

The previous command will create an API Gateway REST API that is accessible through the following stage endpoint:
```
$ aws cloudformation describe-stacks --stack-name createDeleteCFStacks --query "Stacks[*].Outputs" --output table
```

You can use this Rest API to create temporary stacks that get deleted after a period of time. To achieve this, we first need to upload our cloudformation template to S3.

**Step 2**. Upload your temporary stacks' cloudformation template to AWS s3.
```
$ aws s3 cp my-temporary-stack.yml s3://{bucketName}/my-temporary-stack.yml
$ aws s3 cp template_parameters.yml s3://{bucketName}/template_parameters.json # if needed
```

**Step 3**. Use the resources created on step 1 to create your temporary stack. The temporary stack will be deleted automatically after a specified TTL.

```
    $ curl -d '{"template_url": "https://{bucketName}.s3-{region}.amazonaws.com/my-temporary-stack.yml", "stack_name":"new_temporary_stack", "template_parameters": "https://{bucketName}.s3-{region}.amazonaws.com/stack-parameters.json", "extra_parameters": {"StackTTL": "30"}}' -H "Content-Type: application/json" -X POST https://[restApiId].execute-api.[region].amazonaws.com/prod/stack
```

**Examples:**

Create a new ec2 instance that gets autodeleted after 10 minutes:

```bash
$ curl -d '{"template_url": "https://educalleja-misc.s3-eu-west-1.amazonaws.com/examples/ec2-instance-autodelete.yml", "stack_name":"ec2instance", "extra_parameters": {"StackTTL": "10"}}' -H "Content-Type: application/json" -X POST https://[restapi-id].execute-api.[region].amazonaws.com/prod/stack
```

Start an docker container running Nginx on AWS ECS Fargate
```
payload='{"template_url": "https://educalleja-misc.s3-eu-west-1.amazonaws.com/examples/fargate-stack.yml", "stack_name":"farga", "extra_parameters": {"FargatePublicPrivateVPCURL": "https://educalleja-misc.s3-eu-west-1.amazonaws.com/examples/fargate-public-private-vpc.yml","FargateServiceURL": "https://educalleja-misc.s3-eu-west-1.amazonaws.com/examples/fargate-service.yml", "Route53StackURL": "https://educalleja-misc.s3-eu-west-1.amazonaws.com/examples/route53.yml", "StackTTL": "15", "ImageURL": "educalleja/http-hello-world",  "ContainerPort": "80", "ServiceDomain": "fargateService.educalleja.com", "HostedZoneId": "Z4WXR9EH7LMUD" }}'
curl -d '$payload' -H "Content-Type: application/json" -X POST https://[restapi-id].execute-api.[region].amazonaws.com/prod/stack
```

**Attention:** 
These cloudformation stacks create roles that have AdministratorAccess that could represent a security concern. Use with caution.
