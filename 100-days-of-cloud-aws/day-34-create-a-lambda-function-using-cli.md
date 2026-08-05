# Day 34: Create a Lambda Function Using CLI

The Nautilus DevOps team continues to explore serverless architecture by setting up another Lambda function. This time, the task must be completed using the AWS Console to familiarize the team with the web interface. The function will return a custom greeting and demonstrate the capabilities of AWS Lambda effectively.

Create Python Script: Create a Python script named `lambda_function.py` with a function that returns the body `Welcome to KKE AWS Labs!` and status code 200.

Zip the Python Script: Zip the script into a file named `function.zip`.

Create Lambda Function: Create a Lambda function named `xfusion-lambda-cli` using the zipped file and specify Python as the runtime.

IAM Role: Use the IAM role named `lambda_execution_role`.

Use AWS CLI which is already configured on the aws-client host.

## Specific Requirements:

1. Create Python Script: Create a Python script named `lambda_function.py` with a function that returns the body `Welcome to KKE AWS Labs!` and status code 200.
2. Zip the Python Script: Zip the script into a file named `function.zip`.
3. Create Lambda Function: Create a Lambda function named `xfusion-lambda-cli` using the zipped file and specify Python as the runtime.
4. IAM Role: Use the IAM role named `lambda_execution_role`.
5. Use AWS CLI which is already configured on the aws-client host.

## Solution

This is the **AWS CLI** counterpart to the previous Console-based Lambda challenge (Day 33): same idea, but every step is a command on the `aws-client` host. (The challenge body text mentions the Console, but the title and the final instruction make clear this one is done with the AWS CLI — that's what we follow.) The workflow is the classic package-and-deploy cycle: write the Python handler, zip it into a deployment package, and hand that ZIP to `create-function`.

Unlike Day 33, the IAM role `lambda_execution_role` **already exists** here (it's the role the challenge tells us to *use*, not create), so we look it up and reuse it rather than creating a new one. The function returns a dictionary with `statusCode: 200` and the greeting `body`.

> **Note — AWS CLI v1 vs v2.** The `aws-client` host runs **AWS CLI v1**. On v1, `aws lambda invoke` takes the output file as a positional argument and does **not** accept `--cli-binary-format` (that flag is v2-only and would fail with `Unknown options: --cli-binary-format`). The commands below use the v1 form; on v2 you'd add `--cli-binary-format raw-in-base64-out`.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
FUNCTION_NAME="xfusion-lambda-cli"
ROLE_NAME="lambda_execution_role"
RUNTIME="python3.12"
HANDLER="lambda_function.lambda_handler"
```

> **Why:** These are the challenge-fixed names (`xfusion-lambda-cli`, `lambda_execution_role`) plus the settings we pick to satisfy "specify Python as the runtime": `python3.12` and the handler `lambda_function.lambda_handler` — that's `<file>.<function>`, so Lambda calls `lambda_handler` inside `lambda_function.py`. `AWS_REGION` is pinned to `us-east-1` because the KodeKloud lab always runs there.

### 🔑 Step 1: Get the account ID and build the role ARN

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/${ROLE_NAME}"
```

Real values from the lab run:

```
ACCOUNT_ID=961017402449
ROLE_ARN=arn:aws:iam::961017402449:role/lambda_execution_role
```

> **Why:** `aws sts get-caller-identity` returns the identity making the call; `--query Account` extracts the 12-digit **account ID**. `create-function` needs the execution role's full **ARN** (Amazon Resource Name), which embeds the account ID, so we assemble it up front.

### 🔎 Step 2: Confirm the execution role exists

```bash
aws iam get-role --role-name "$ROLE_NAME" --query "Role.Arn" --output text
```

The role already existed, so it's reused as-is:

```
arn:aws:iam::961017402449:role/lambda_execution_role
```

> **Why:** The challenge says to *use* `lambda_execution_role`, implying it's already provisioned. `aws iam get-role` is a read-only lookup that returns the role's details (or fails with `NoSuchEntity` if it's absent). It returned the ARN, confirming the role is present — so we don't create a duplicate. (Had it been missing, we would create it with a trust policy allowing `lambda.amazonaws.com` to `sts:AssumeRole` and attach the managed `AWSLambdaBasicExecutionRole` policy, exactly as in Day 33.)

### 📝 Step 3: Create the Python script

```bash
cat > lambda_function.py <<'EOF'
def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': 'Welcome to KKE AWS Labs!'
    }
EOF
```

> **Why:** `lambda_handler(event, context)` is the entry point Lambda invokes — `event` holds the input, `context` the runtime info. It returns a dictionary with `statusCode: 200` and the required `body` string. The filename must be `lambda_function.py` to match the `lambda_function` part of the handler setting; a mismatch causes an import error at runtime.

### 🗜️ Step 4: Zip the script into function.zip

```bash
zip function.zip lambda_function.py
```

Real value from the lab run: `function.zip` is `293` bytes.

> **Why:** Lambda deploys code as a **deployment package** — a ZIP archive containing the handler file (and any dependencies). `zip function.zip lambda_function.py` produces the `function.zip` the challenge names, which `create-function` uploads in the next step.

### 🚀 Step 5: Create the Lambda function

```bash
aws lambda create-function --region "$AWS_REGION" \
  --function-name "$FUNCTION_NAME" \
  --runtime "$RUNTIME" \
  --role "$ROLE_ARN" \
  --handler "$HANDLER" \
  --zip-file fileb://function.zip
```

Real value from the lab run: `FunctionArn=arn:aws:lambda:us-east-1:961017402449:function:xfusion-lambda-cli`.

> **Why:** `create-function` deploys the function from our ZIP. `--function-name` sets the required name `xfusion-lambda-cli`; `--runtime python3.12` selects the Python runtime; `--role` is the execution role's ARN; `--handler lambda_function.lambda_handler` tells Lambda which function to call; and `--zip-file fileb://function.zip` uploads the package (`fileb://` reads it as raw binary). Because the role already existed and had propagated, this succeeds on the first try (no assume-role retry needed, unlike a freshly created role).

### ⏳ Step 6: Wait for the function to be active

```bash
aws lambda wait function-active-v2 --region "$AWS_REGION" \
  --function-name "$FUNCTION_NAME"
```

> **Why:** After creation Lambda provisions the execution environment, moving the function from `Pending` to `Active`. `wait function-active-v2` blocks until it reports `Active`, so we don't invoke it too early. For a small function this is nearly instant.

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

The configuration matches the requirements and the invocation returns the greeting with `statusCode` 200:

```
---------------------------------------------------------------------
|                            GetFunction                            |
+---------+---------------------------------------------------------+
|  Handler|  lambda_function.lambda_handler                         |
|  Name   |  xfusion-lambda-cli                                     |
|  Role   |  arn:aws:iam::961017402449:role/lambda_execution_role   |
|  Runtime|  python3.12                                             |
|  State  |  Active                                                 |
+---------+---------------------------------------------------------+

200
{"statusCode": 200, "body": "Welcome to KKE AWS Labs!"}
```

> **Why:** `get-function` reads the deployed configuration; the `--query` projection confirms the name, Python runtime, handler, the reused `lambda_execution_role`, and `State: Active`. `aws lambda invoke` runs the function: `--query StatusCode` prints the invocation status (`200` = executed without error) and the return value is written to `response.json`. Printing that file shows the function's own output — `statusCode: 200` and `body: Welcome to KKE AWS Labs!` — confirming the code works. On AWS CLI v1 the output file is a positional argument and no `--cli-binary-format` flag is used.

## Best Practices

- **Reuse the designated role.** When a challenge (or a real project) hands you a role to *use*, look it up and reuse it rather than creating a parallel one — `aws iam get-role` is the check.
- **Keep the deployment package minimal.** Zip only what the function needs; a lean `function.zip` deploys faster and reduces cold-start time.
- **Match handler to filename.** `lambda_function.lambda_handler` requires a `lambda_function.py` with a `lambda_handler` function — the most common cause of an "Unable to import module" error is a mismatch here.
- **Mind AWS CLI version differences.** `lambda invoke` syntax differs between v1 and v2; know which the host runs to avoid the `--cli-binary-format` error.
- **Pin a supported runtime.** Use a current Python version (`python3.12`); AWS deprecates old runtimes over time.

### 📚 Official Documentation

- [Deploy Python Lambda functions with .zip file archives](https://docs.aws.amazon.com/lambda/latest/dg/python-package.html)
- [Lambda execution role](https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html)
- [create-function — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/lambda/create-function.html)
- [invoke — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/lambda/invoke.html)
- [get-role — AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/iam/get-role.html)
