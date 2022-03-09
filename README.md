# Cloudformation

* A Template is equivalent to a .tf file - i.e. describing the resources to be provisioned
* A Stack provisions the resources described in a template

## Install hook

To install the hook provided in this repository, cd into the `hooks` directory and run:

```shell
cfn submit --set-default
```

When the command above is finished (may take several minutes), copy the value of `TypeArn`
from the response and store it in an environment variable:

```shell
export HOOK_TYPE_ARN="arn:aws:cloudformation:eu-north-1:677200501247:type/hook/Styra-OPA-Hook"
```

Next, set the AWS region and the URL to use for calling OPA. I'm using [ngrok](https://ngrok.com/) 
for exposing an OPA running locally to the public, but there are obviously many ways to accomplish this.

```shell
export AWS_REGION="eu-north-1"
export OPA_URL="http://206d-46-182-202-220.ngrok.io/v1/data/policy/deny"
```

With the configuration variables set, push the configuration to AWS:

```shell
aws cloudformation --region "$AWS_REGION" set-type-configuration \
  --configuration "{\"CloudFormationConfiguration\":{\"HookConfiguration\":{\"TargetStacks\":\"ALL\",\"FailureMode\":\"FAIL\",\"Properties\":{\"OpaUrl\": \"$OPA_URL\"}}}}" \
  --type-arn $HOOK_TYPE_ARN
```

## Testing stack creation

The hook is now installed. You can now try to push the S3 bucket stack provided in the templates directory:

```shell
aws cloudformation create-stack --stack-name cfn-s3 --template-body file://templates/s3.yaml
```

To see the outcome of the stack creation, use `describe-stack-events`:

```shell
aws cloudformation describe-stack-events --stack-name cfn-s3
```

If you want to re-run the stack creation, remember to delete the existing one first:

```shell
aws cloudformation delete-stack --stack-name cfn-s3
```

Or update the stack:

```shell
aws cloudformation update-stack --stack-name cfn-s3 --template-body file://templates/s3.yaml
```

## Rego Policy

The OPA configured to receive requests from the CFN hook will have its input provided in this format:

```json
{
  "input": {
    "action": "create",
    "name": "AWS::S3::Bucket",
    "type": "AWS::S3::Bucket",
    "properties": {"Tags": [{"Value": "Anders", "Key": "Owner"}]}
  }
}
```

Some notes on the above format:
* The "action" is either "create", "update" or "delete"
* I'm not sure whether "name" and "type" ever differs, but it seems like a good idea to provide both
* The properties are exactly as defined in the template - no generated or default values

The hook is currently hardcoded to deal with "deny" style policy responses, i.e. a **set** of messages.
If the set (as represented by a JSON array) is empty, the request is approved. If the set has any entries,
the request is denied, and the messages returned are logged to CloudWatch at error level.

## Logs

Any logs emitted from the Python hook can be found under CloudWatch in your AWS account.

## Docs

See docs on registering the hook here:
https://docs.aws.amazon.com/cloudformation-cli/latest/userguide/registering-hook-python.html

To see all attributes an object (like say, and S3 bucket) may have, consult the AWS resource type ref:
https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html
