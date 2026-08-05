# Day 33: Create a Lambda Function

The Nautilus DevOps team is embracing serverless architecture by integrating AWS Lambda into their operational tasks. They have decided to deploy a simple Lambda function that will return a custom greeting to demonstrate serverless capabilities effectively. This function is crucial for showcasing rapid deployment and easy scalability features of AWS Lambda to the team.

Create Lambda Function: Create a Lambda function named `datacenter-lambda`.

Runtime: Use the Runtime Python.

Deploy: The function should print the body `Welcome to KKE AWS Labs!`.

Status Code: Ensure the status code is 200.

IAM Role: Create and use the IAM role named `lambda_execution_role`.

Use the AWS Console to complete this task.

## Specific Requirements:

1. Create Lambda Function: Create a Lambda function named `datacenter-lambda`.
2. Runtime: Use the Runtime Python.
3. Deploy: The function should print the body `Welcome to KKE AWS Labs!`.
4. Status Code: Ensure the status code is 200.
5. IAM Role: Create and use the IAM role named `lambda_execution_role`.

## Solution

**AWS Lambda** runs your code without any servers to manage — you upload a function, and Lambda executes it on demand. Two pieces are always required: the **code** (here a tiny Python handler returning a `statusCode` of `200` and the greeting `body`) and an **execution role** — an IAM role Lambda *assumes* while running, which at minimum needs permission to write logs to CloudWatch. The challenge names that role `lambda_execution_role`.

> **Note on Console vs CLI.** This challenge is officially meant to be completed through the **AWS Console** (the next challenge, Day 34, is the one KodeKloud designates for the AWS CLI). For the practical purposes of this repository — keeping every solution scriptable, reproducible, and consistent with the rest of the track — we solve it here with the **AWS CLI** instead. The end result (function, role, runtime, and response) is identical to what the Console produces.

Two things to know going in. First, the KodeKloud sandbox denies `iam:PutRolePolicy`, so we grant permissions with the AWS-**managed** policy `AWSLambdaBasicExecutionRole` instead of an inline policy. Second, a freshly created IAM role takes a few seconds to **propagate**, so `create-function` can briefly fail with an "assume role" error right after the role is made — simply retrying resolves it.

> **Gotcha — AWS CLI v1 vs v2.** The `aws-client` host in this lab runs **AWS CLI v1**, and `aws lambda invoke` behaves differently across major versions. On **v2** you must pass `--cli-binary-format raw-in-base64-out` (and it errors without it in some cases); on **v1** that flag **does not exist** and the command fails with `Unknown options: --cli-binary-format`. Our first invoke attempt used the v2 syntax and hit exactly that error — the fix was to drop the flag and use the v1 form shown in Step 7. If you're on v2, add `--cli-binary-format raw-in-base64-out` to the invoke command.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
FUNCTION_NAME="datacenter-lambda"
ROLE_NAME="lambda_execution_role"
RUNTIME="python3.12"
HANDLER="lambda_function.lambda_handler"
```

> **Why:** These are the challenge-fixed names (`datacenter-lambda`, `lambda_execution_role`) plus the settings we choose to satisfy "Runtime Python": `python3.12` (a current Python runtime) and the handler `lambda_function.lambda_handler`, which is `<file>.<function>` — Lambda will look for a function named `lambda_handler` in a file named `lambda_function.py`. `AWS_REGION` is pinned to `us-east-1` because the KodeKloud lab always runs there.

### 🔑 Step 1: Get the account ID and build the role ARN

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/${ROLE_NAME}"
```

Real values from the lab run:

```
ACCOUNT_ID=850865896881
ROLE_ARN=arn:aws:iam::850865896881:role/lambda_execution_role
```

> **Why:** `aws sts get-caller-identity` asks AWS "who am I?" and returns the identity making the call; we pull the 12-digit **account ID** with `--query Account`. An IAM role's **ARN** (Amazon Resource Name) embeds that account ID, and `create-function` needs the role's ARN, so we assemble it now rather than parsing it out of a later response.

### 🛡️ Step 2: Create the IAM execution role

```bash
cat > trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "lambda.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role --role-name "$ROLE_NAME" \
  --assume-role-policy-document file://trust-policy.json \
  --query "Role.Arn" --output text
```

The role was created and returned its ARN:

```
arn:aws:iam::850865896881:role/lambda_execution_role
```

> **Why:** An **IAM role** is a set of permissions an AWS service can temporarily assume — Lambda can't run your function until you give it one. The **trust policy** (the JSON) declares *who* may assume the role: `"Service": "lambda.amazonaws.com"` with the `sts:AssumeRole` action means "the Lambda service is allowed to assume this role." `aws iam create-role` creates it — `--role-name` sets the required name and `--assume-role-policy-document file://...` attaches the trust policy from the local file. IAM is a **global** service, so no `--region` is needed. This role has no permissions yet; the next step grants them.

### 📎 Step 3: Attach the basic execution policy

```bash
aws iam attach-role-policy --role-name "$ROLE_NAME" \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

> **Why:** `attach-role-policy` links a permissions policy to the role. `AWSLambdaBasicExecutionRole` is an **AWS-managed policy** (maintained by AWS) that grants exactly the baseline a function needs — permission to create and write CloudWatch Logs. We use a managed policy rather than an inline one because the KodeKloud sandbox user is denied `iam:PutRolePolicy` (inline-policy creation). This follows least privilege: the function gets logging permission and nothing more.

### 📦 Step 4: Write and package the function code

```bash
cat > lambda_function.py <<'EOF'
def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': 'Welcome to KKE AWS Labs!'
    }
EOF

zip function.zip lambda_function.py
```

> **Why:** The handler `lambda_handler(event, context)` is the entry point Lambda calls on every invocation; `event` carries the input and `context` runtime info. It returns a dictionary with `statusCode: 200` and the required `body` string — exactly what the challenge asks for. Lambda expects the code as a **deployment package** (a ZIP), so `zip` bundles `lambda_function.py` into `function.zip`. The filename `lambda_function.py` must match the `lambda_function` part of the handler setting.

### 🚀 Step 5: Create the Lambda function

```bash
aws lambda create-function --region "$AWS_REGION" \
  --function-name "$FUNCTION_NAME" \
  --runtime "$RUNTIME" \
  --role "$ROLE_ARN" \
  --handler "$HANDLER" \
  --zip-file fileb://function.zip
```

Real value from the lab run: `FunctionArn=arn:aws:lambda:us-east-1:850865896881:function:datacenter-lambda`.

> **Why:** `create-function` deploys the function. `--function-name` sets the required name `datacenter-lambda`; `--runtime python3.12` selects the Python runtime; `--role` is the execution role's ARN from Step 1; `--handler lambda_function.lambda_handler` tells Lambda which function to call; and `--zip-file fileb://function.zip` uploads the package (`fileb://` reads the file as raw binary). Because the role was created moments earlier, this call can briefly fail with an "assume role" error while the role propagates through IAM — simply re-running it succeeds once propagation completes.

### ⏳ Step 6: Wait for the function to be active

```bash
aws lambda wait function-active-v2 --region "$AWS_REGION" \
  --function-name "$FUNCTION_NAME"
```

> **Why:** After creation, Lambda provisions the function's execution environment, moving it through a `Pending` state before `Active`. `wait function-active-v2` blocks until the function reports `Active`, so we don't invoke it before it's ready. For a small function this is nearly instant.

### ✅ Step 7: Verify

```bash
aws lambda get-function --region "$AWS_REGION" \
  --function-name "$FUNCTION_NAME" \
  --query "Configuration.{Name:FunctionName,Runtime:Runtime,Handler:Handler,Role:Role,State:State}" \
  --output table

aws lambda invoke --region "$AWS_REGION" \
  --function-name "$FUNCTION_NAME" \
  --query "StatusCode" --output text \
  response.json

cat response.json
```

The configuration is correct and the invocation returns the greeting with `statusCode` 200:

```
---------------------------------------------------------------------
|                            GetFunction                            |
+---------+---------------------------------------------------------+
|  Handler|  lambda_function.lambda_handler                         |
|  Name   |  datacenter-lambda                                      |
|  Role   |  arn:aws:iam::850865896881:role/lambda_execution_role   |
|  Runtime|  python3.12                                             |
|  State  |  Active                                                 |
+---------+---------------------------------------------------------+

200
{"statusCode": 200, "body": "Welcome to KKE AWS Labs!"}
```

> **Why:** `get-function` reads the deployed configuration; the `--query` projection confirms the name, Python runtime, handler, the `lambda_execution_role`, and `State: Active`. `aws lambda invoke` actually runs the function: `--query StatusCode` prints the **invocation** status (`200` means Lambda executed it without error) and the response body is written to `response.json`. Printing that file shows the function's own return value — `statusCode: 200` and `body: Welcome to KKE AWS Labs!` — proving the code behaves as required. (On AWS CLI v1, `invoke` takes the output file as a positional argument and needs no `--cli-binary-format` flag, which is v2-only.)

## Best Practices

- **Use a managed policy for basic logging.** `AWSLambdaBasicExecutionRole` grants exactly the CloudWatch Logs permissions every function needs; reach for a custom least-privilege policy only when the function accesses other services.
- **One dedicated execution role per function (or tight group).** Avoid sharing a broad role across unrelated functions so each has only the access it needs.
- **Expect IAM propagation delay.** A brand-new role isn't instantly usable; retry `create-function` rather than assuming a real failure.
- **Match the handler to the filename.** `lambda_function.lambda_handler` requires a `lambda_function.py` file containing `lambda_handler` — a mismatch causes a runtime import error.
- **Pin a supported runtime.** Use a current Python version (e.g. `python3.12`); AWS deprecates old runtimes, and deploying to a deprecated one eventually fails.

### 📚 Official Documentation

- [Create a Lambda function with the console](https://docs.aws.amazon.com/lambda/latest/dg/getting-started.html)
- [Lambda execution role](https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html)
- [Building Lambda functions with Python](https://docs.aws.amazon.com/lambda/latest/dg/lambda-python.html)
- [create-function — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/lambda/create-function.html)
- [invoke — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/lambda/invoke.html)
