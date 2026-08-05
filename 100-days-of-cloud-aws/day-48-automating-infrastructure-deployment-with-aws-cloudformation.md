# Day 48: Automating Infrastructure Deployment with AWS CloudFormation

The Nautilus DevOps team needs to implement a Lambda function using a CloudFormation stack. Create a CloudFormation template named `/root/datacenter-lambda.yml` on the AWS client host and configure it to create the following components. The stack name must be `datacenter-lambda-app`.

Create a Lambda function named `datacenter-lambda`.
Use the Runtime Python.
The function should print the body Welcome to KKE AWS Labs!.
Ensure the status code is 200.
Create and use the IAM role named `lambda_execution_role`.

## Specific Requirements:

1. Create the CloudFormation template on the AWS client host at `/root/datacenter-lambda.yml`.
2. The stack name must be `datacenter-lambda-app`.
3. Create a Lambda function named `datacenter-lambda`.
4. Use a Python runtime.
5. The function should return the body `Welcome to KKE AWS Labs!`.
6. Ensure the status code is `200`.
7. Create and use an IAM role named `lambda_execution_role`.

## Solution

This is a minimal, self-contained CloudFormation stack: one IAM role and one Lambda function. The function's code is tiny and contains only single quotes, so it embeds cleanly **inline** via `Code: ZipFile` — no need for the placeholder-plus-`update-function-code` dance that nested quotes would force (see Day 47). The handler just returns a dictionary with `statusCode: 200` and the required `body` string; a synchronous invoke echoes that dictionary back so we can confirm both values.

One sandbox constraint carries over: the KodeKloud lab user is denied `iam:PutRolePolicy`, so the role attaches an **AWS managed policy** (`AWSLambdaBasicExecutionRole`) instead of an inline one.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
STACK_NAME="datacenter-lambda-app"
TEMPLATE_FILE="/root/datacenter-lambda.yml"
FUNCTION_NAME="datacenter-lambda"
```

### 📄 Step 1: Write the CloudFormation template

```bash
cat > /root/datacenter-lambda.yml <<'EOF'
AWSTemplateFormatVersion: '2010-09-09'
Description: Lambda function deployed via CloudFormation

Resources:
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: lambda_execution_role
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

  DatacenterLambda:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: datacenter-lambda
      Runtime: python3.12
      Handler: index.lambda_handler
      Role: !GetAtt LambdaExecutionRole.Arn
      Timeout: 10
      Code:
        ZipFile: |
          def lambda_handler(event, context):
              return {
                  'statusCode': 200,
                  'body': 'Welcome to KKE AWS Labs!'
              }
EOF
```

> **Why:** A CloudFormation **template** declaratively describes the resources you want; CloudFormation reads it and provisions them together as a **stack**. Each entry under `Resources` has a `Type` (the AWS resource kind) and `Properties`.
>
> - `AWS::IAM::Role` with `RoleName: lambda_execution_role` creates the **execution role** — the identity the function assumes at runtime. `AssumeRolePolicyDocument` is the **trust policy**: it declares that the Lambda service (`lambda.amazonaws.com`) is allowed to `sts:AssumeRole`, i.e. to take on this role. `ManagedPolicyArns` attaches a ready-made AWS policy; `AWSLambdaBasicExecutionRole` grants the baseline permission every function needs — writing its logs to CloudWatch. We use a managed policy because the sandbox denies inline-policy creation.
> - `AWS::Lambda::Function` defines the function. `FunctionName` sets its name. `Runtime: python3.12` selects the Python version (satisfying the "Runtime Python" requirement). `Handler: index.lambda_handler` is `<file>.<function>` — the file part must be `index` because CloudFormation's inline `ZipFile` always writes the code into a file named `index`. `Role: !GetAtt LambdaExecutionRole.Arn` wires in the role's **ARN** (its unique identifier), obtained with the `!GetAtt` intrinsic function; this also makes CloudFormation create the role before the function. `Timeout: 10` allows up to 10 seconds per invocation (far more than this instant function needs). `Code.ZipFile` carries the source inline; the handler returns a dictionary whose `statusCode` is `200` and whose `body` is the required string.

### 🚀 Step 2: Create the stack

```bash
aws cloudformation create-stack \
  --stack-name "$STACK_NAME" \
  --template-body file:///root/datacenter-lambda.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region "$AWS_REGION"
```

> **Why:** `create-stack` submits the template and tells CloudFormation to provision everything in it. `--template-body file://...` loads the template from local disk. `--capabilities CAPABILITY_NAMED_IAM` is required because the template creates an IAM resource with an **explicit name** (`RoleName: lambda_execution_role`) — CloudFormation makes you acknowledge the creation of named IAM identities as a guardrail against unintended privilege grants. Omitting it rejects the call with `InsufficientCapabilities`.

### ⏳ Step 3: Wait for the stack to finish

```bash
aws cloudformation wait stack-create-complete \
  --stack-name "$STACK_NAME" \
  --region "$AWS_REGION"
```

> **Why:** `create-stack` returns immediately while provisioning continues in the background. `wait stack-create-complete` blocks until the stack reaches `CREATE_COMPLETE` (or fails), so verification doesn't run against a half-built stack. On failure this exits non-zero and you'd inspect `aws cloudformation describe-stack-events --stack-name "$STACK_NAME"` to find the offending resource.

### ✅ Step 4: Verify

```bash
aws lambda invoke --function-name "$FUNCTION_NAME" /tmp/out.json && cat /tmp/out.json
```

> **Why:** `lambda invoke` runs the function synchronously and writes its return value to `/tmp/out.json`. The command's own JSON shows `"StatusCode": 200` — meaning Lambda executed the function without error — and the payload file shows what the code returned.

Expected output:

```
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
{"statusCode": 200, "body": "Welcome to KKE AWS Labs!"}
```

The returned payload carries `statusCode: 200` and the exact body `Welcome to KKE AWS Labs!` — both requirements satisfied.

## Best Practices

- **Inline code is fine for tiny, quote-safe functions.** This handler is a few lines with only single quotes, so `Code: ZipFile` embeds it cleanly and keeps the stack self-contained. Reach for a real zip + `update-function-code` only when the code has multiple files, dependencies, or characters (nested quotes) that a YAML block scalar can mangle.
- **Prefer managed policies in restricted sandboxes.** The lab user cannot create inline IAM policies (`iam:PutRolePolicy` is denied). `AWSLambdaBasicExecutionRole` supplies the CloudWatch Logs permissions every function needs without hitting that wall. In production, scope down to a least-privilege customer-managed policy.
- **Acknowledge IAM capabilities explicitly.** Naming an IAM resource requires `CAPABILITY_NAMED_IAM`; treat that flag as a prompt to review which identities and permissions your template creates.
- **Verify by invoking, not just by stack status.** `CREATE_COMPLETE` only means the resources were provisioned — CloudFormation does not validate the Python. A synchronous `invoke` is what actually proves the function returns the right status code and body.

### 📚 Official Documentation

- [AWS::Lambda::Function](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-lambda-function.html)
- [AWS::Lambda::Function Code (ZipFile)](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-properties-lambda-function-code.html)
- [AWS::IAM::Role](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-iam-role.html)
- [CloudFormation CreateStack — Capabilities](https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_CreateStack.html)
- [Invoke a Lambda function (AWS CLI)](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-awscli.html)
