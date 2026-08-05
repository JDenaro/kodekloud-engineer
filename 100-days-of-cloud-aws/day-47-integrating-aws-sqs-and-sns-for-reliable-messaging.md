# Day 47: Integrating AWS SQS and SNS for Reliable Messaging

The Nautilus DevOps team needs to implement priority queuing using Amazon SQS and SNS. The goal is to create a system where messages with different priorities are handled accordingly. You are required to use AWS CloudFormation to deploy the necessary resources in your AWS account. The CloudFormation template should be created on the AWS client host at `/root/devops-priority-stack.yml`, the stack name must be `devops-priority-stack` and it should create the following resources:

Two SQS queues named `devops-High-Priority-Queue` and `devops-Low-Priority-Queue`.
An SNS topic named `devops-Priority-Queues-Topic`.
A Lambda function named `devops-priorities-queue-function` that will consume messages from the SQS queues. The Lambda function code is provided in `/root/index.py` on the AWS client host.
An IAM role named `lambda_execution_role` that provides the necessary permissions for the Lambda function to interact with SQS and SNS.

Once the stack is deployed, to test the same you can publish messages to the SNS topic, invoke the Lambda function and observe the order in which they are processed by the Lambda function. The high-priority message must be processed first.

## Specific Requirements:

1. Create the CloudFormation template on the AWS client host at `/root/devops-priority-stack.yml`.
2. The stack name must be `devops-priority-stack`.
3. Two SQS queues named `devops-High-Priority-Queue` and `devops-Low-Priority-Queue`.
4. An SNS topic named `devops-Priority-Queues-Topic`.
5. A Lambda function named `devops-priorities-queue-function` that consumes messages from the SQS queues, using the code provided in `/root/index.py`. The Lambda timeout must be set to 10 seconds.
6. An IAM role named `lambda_execution_role` granting the Lambda the permissions it needs to interact with SQS and SNS.

## Solution

The whole design hinges on **fan-out with filtering**: a single SNS topic broadcasts every published message, and each SQS queue subscribes to it with a **subscription filter policy** on the `priority` message attribute — so a message tagged `high` lands only in `devops-High-Priority-Queue` and one tagged `low` only in `devops-Low-Priority-Queue`. The provided Lambda (`/root/index.py`) reads two environment variables, `high_priority_queue` and `low_priority_queue` (each a queue **URL**), and **always polls the high-priority queue first**; it only touches the low-priority queue when the high one is empty. That ordering is what makes high-priority messages get processed first.

**Why SNS at all — can't we just publish straight to SQS?** Technically yes: you could `sqs:SendMessage` directly to each queue and skip SNS entirely. The point of putting SNS in front is to move the *routing decision out of the producer and into the infrastructure*. The producer publishes **once** to a single topic, tagging the message with a `priority` attribute, and the subscription filter policies decide which queue it belongs in — the producer never needs to know the queues exist, how many there are, or the priority logic. That buys you **decoupling** (producer ↔ queues), **fan-out** (one message can reach many subscribers at once, not just one queue), and **declarative, centralized routing** (change the routing by editing a filter policy, not by redeploying the producer). SQS is still essential here as the **buffer** that holds messages until the Lambda drains them, and whose "which queue do I empty first" ordering encodes the priority. In short: SQS alone could move the bytes, but SNS is what turns "send to a queue" into "route by attribute," which is exactly the priority-queuing behavior the challenge is modeling.

Two decisions worth calling out up front:

- **The Lambda code is delivered as a real zip, not inline.** CloudFormation's `AWS::Lambda::Function` requires a `Code` property, so we give it a tiny valid placeholder inline (`ZipFile`) and then upload the actual `/root/index.py` with `update-function-code` after the stack is created. The provided code contains nested quotes (`"'"`), and embedding it inline inside a YAML block scalar is fragile — a single mangled character produces a `Runtime.UserCodeSyntaxError` at deploy time. Shipping the untouched file as a zip sidesteps that entirely and keeps the stack focused on infrastructure. See **Best Practices** for the trade-off.
- **The IAM role uses AWS managed policies, not inline policies.** The KodeKloud sandbox user is denied `iam:PutRolePolicy`, so inline role policies fail with `AccessDenied`. Attaching AWS managed policies (`ManagedPolicyArns`) avoids that restriction.

### 📦 Variables

```bash
AWS_REGION="us-east-1"
STACK_NAME="devops-priority-stack"
TEMPLATE_FILE="/root/devops-priority-stack.yml"
FUNCTION_NAME="devops-priorities-queue-function"
CODE_FILE="/root/index.py"
ZIP_FILE="/root/function.zip"
```

### 📝 Step 1: Review the provided Lambda code

```bash
cat /root/index.py
```

> **Why:** Before wiring anything, we read the function the challenge hands us so the template matches what the code expects. This Lambda calls `sqs.receive_message` then `sqs.delete_message`, and it reads two environment variables — `high_priority_queue` and `low_priority_queue` — each holding a **queue URL** (the HTTPS address that uniquely identifies an SQS queue). `lambda_handler` polls the high-priority queue first and falls back to the low-priority one only when the high one returns no messages. Knowing this tells us exactly which env vars to set, that their values must be queue URLs (not ARNs), and which IAM permissions the role needs (`sqs:ReceiveMessage` and `sqs:DeleteMessage`).

### 📄 Step 2: Write the CloudFormation template

```bash
cat > /root/devops-priority-stack.yml <<'EOF'
AWSTemplateFormatVersion: '2010-09-09'
Description: Priority queuing with SQS, SNS and a Lambda consumer

Resources:
  HighPriorityQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: devops-High-Priority-Queue

  LowPriorityQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: devops-Low-Priority-Queue

  PriorityTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: devops-Priority-Queues-Topic

  HighQueueSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref PriorityTopic
      Protocol: sqs
      Endpoint: !GetAtt HighPriorityQueue.Arn
      RawMessageDelivery: true
      FilterPolicy:
        priority:
          - high

  LowQueueSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref PriorityTopic
      Protocol: sqs
      Endpoint: !GetAtt LowPriorityQueue.Arn
      RawMessageDelivery: true
      FilterPolicy:
        priority:
          - low

  QueuePolicy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      Queues:
        - !Ref HighPriorityQueue
        - !Ref LowPriorityQueue
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: sns.amazonaws.com
            Action: sqs:SendMessage
            Resource:
              - !GetAtt HighPriorityQueue.Arn
              - !GetAtt LowPriorityQueue.Arn
            Condition:
              ArnEquals:
                aws:SourceArn: !Ref PriorityTopic

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
        - arn:aws:iam::aws:policy/AmazonSQSFullAccess
        - arn:aws:iam::aws:policy/AmazonSNSFullAccess
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

  PriorityLambda:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: devops-priorities-queue-function
      Runtime: python3.12
      Handler: index.lambda_handler
      Role: !GetAtt LambdaExecutionRole.Arn
      Timeout: 10
      Environment:
        Variables:
          high_priority_queue: !Ref HighPriorityQueue
          low_priority_queue: !Ref LowPriorityQueue
      Code:
        ZipFile: |
          def lambda_handler(event, context):
              return "placeholder - real code uploaded via update-function-code"
EOF
```

> **Why:** A CloudFormation **template** is a declarative description of the resources you want; CloudFormation reads it and provisions everything as one unit called a **stack**. `AWSTemplateFormatVersion` pins the template language version. Each entry under `Resources` is a resource with a `Type` (the AWS resource kind) and `Properties`.
>
> - `AWS::SQS::Queue` with `QueueName` creates the two named queues. **SQS** (Simple Queue Service) is a managed message queue that stores messages until a consumer reads and deletes them.
> - `AWS::SNS::Topic` with `TopicName` creates the topic. **SNS** (Simple Notification Service) is a pub/sub service that fans out each published message to all its subscribers.
> - `AWS::SNS::Subscription` connects the topic to a queue: `Protocol: sqs` and `Endpoint: !GetAtt <Queue>.Arn` point the subscription at the queue's **ARN** (Amazon Resource Name, the resource's globally unique identifier — obtained here with the `!GetAtt` intrinsic function). `RawMessageDelivery: true` tells SNS to deliver the original message body as-is, instead of wrapping it in SNS's JSON envelope, so the Lambda's `response['Messages'][0]['Body']` is the plain text we published. `FilterPolicy` is the **subscription filter policy**: SNS evaluates each message's attributes against it and delivers only matches, so `priority: [high]` routes only high-tagged messages to this queue.
> - `AWS::SQS::QueuePolicy` attaches a resource-based policy to both queues allowing the SNS service principal (`sns.amazonaws.com`) to `sqs:SendMessage`. Without this, SNS cannot deliver into the queues. The `Condition` with `ArnEquals` on `aws:SourceArn` scopes that permission to *our* topic only, so no other topic can push into these queues.
> - `AWS::IAM::Role` with `RoleName: lambda_execution_role` creates the **execution role** — the identity Lambda assumes at runtime. `AssumeRolePolicyDocument` is the **trust policy**: it says the Lambda service (`lambda.amazonaws.com`) is allowed to `sts:AssumeRole`, i.e. to become this role. `ManagedPolicyArns` attaches ready-made AWS policies granting the permissions: `AmazonSQSFullAccess` (receive/delete messages), `AmazonSNSFullAccess`, and `AWSLambdaBasicExecutionRole` (write logs to CloudWatch).
> - `AWS::Lambda::Function` defines the function. `Runtime: python3.12` selects the language version; `Handler: index.lambda_handler` is `<file>.<function>` — it must start with `index` because CloudFormation's inline `ZipFile` always names the file `index`, and our uploaded zip likewise contains `index.py`. `Timeout: 10` allows up to 10 seconds per invocation (the code waits up to 3 seconds per poll). `Environment.Variables` injects the two queue **URLs** — `!Ref` on an `AWS::SQS::Queue` returns its URL, exactly the value type the code expects. `Code.ZipFile` carries a minimal valid placeholder so the template is deployable; the real code is uploaded in Step 5.

### 🚀 Step 3: Create the stack

```bash
aws cloudformation create-stack \
  --stack-name "$STACK_NAME" \
  --template-body file:///root/devops-priority-stack.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region "$AWS_REGION"
```

> **Why:** `create-stack` submits the template and tells CloudFormation to provision every resource in it. `--template-body file://...` reads the template from local disk (the `file://` prefix means "load the content from this path"). `--capabilities CAPABILITY_NAMED_IAM` is mandatory here: whenever a template creates IAM resources with **explicit names** (our role is named `lambda_execution_role`), CloudFormation requires you to explicitly acknowledge that it may create named IAM identities — a guardrail against accidentally granting privileges. Without it the call is rejected with an `InsufficientCapabilities` error.

### ⏳ Step 4: Wait for the stack to finish

```bash
aws cloudformation wait stack-create-complete \
  --stack-name "$STACK_NAME" \
  --region "$AWS_REGION"
```

> **Why:** `create-stack` returns immediately while provisioning continues in the background. `wait stack-create-complete` blocks until the stack reaches `CREATE_COMPLETE` (or fails), so the next steps don't run against half-built resources. If the stack failed, this command exits non-zero and you'd inspect `aws cloudformation describe-stack-events --stack-name "$STACK_NAME"` to see which resource errored.

### 📦 Step 5: Upload the real Lambda code

```bash
cd /root && zip -j function.zip index.py

aws lambda update-function-code \
  --function-name "$FUNCTION_NAME" \
  --zip-file fileb:///root/function.zip \
  --region "$AWS_REGION"

aws lambda wait function-updated \
  --function-name "$FUNCTION_NAME" \
  --region "$AWS_REGION"
```

> **Why:** Now we replace the placeholder with the untouched provided code. `zip -j function.zip index.py` packages the file into a deployment zip (`-j` — "junk paths" — stores just `index.py`, with no directory prefix, so the runtime finds the `index` module). `update-function-code` swaps the function's code for the new zip; `--zip-file fileb://...` uploads it as **raw bytes** (`fileb://`, not `file://`, because a zip is binary). `wait function-updated` blocks until the update finishes propagating, since Lambda briefly reports `LastUpdateStatus: InProgress` and invoking during that window can fail.

### ✅ Step 6: Verify

Publish two high- and two low-priority messages to the topic:

```bash
topicarn=$(aws sns list-topics --query "Topics[?contains(TopicArn, 'devops-Priority-Queues-Topic')].TopicArn" --output text)

aws sns publish --topic-arn $topicarn --message 'High Priority message 1' --message-attributes '{"priority" : { "DataType":"String", "StringValue":"high"}}'
aws sns publish --topic-arn $topicarn --message 'High Priority message 2' --message-attributes '{"priority" : { "DataType":"String", "StringValue":"high"}}'
aws sns publish --topic-arn $topicarn --message 'Low Priority message 1' --message-attributes '{"priority" : { "DataType":"String", "StringValue":"low"}}'
aws sns publish --topic-arn $topicarn --message 'Low Priority message 2' --message-attributes '{"priority" : { "DataType":"String", "StringValue":"low"}}'
```

Then invoke the Lambda once per message and read each result:

```bash
aws lambda invoke --function-name "$FUNCTION_NAME" /tmp/out1.json && cat /tmp/out1.json
aws lambda invoke --function-name "$FUNCTION_NAME" /tmp/out2.json && cat /tmp/out2.json
aws lambda invoke --function-name "$FUNCTION_NAME" /tmp/out3.json && cat /tmp/out3.json
aws lambda invoke --function-name "$FUNCTION_NAME" /tmp/out4.json && cat /tmp/out4.json
```

> **Why:** `list-topics` with a JMESPath `--query` filter finds our topic's ARN by matching its name, avoiding hardcoding the account ID. `sns publish` sends a message; `--message-attributes` attaches the `priority` attribute that the subscription filter policies key on — SNS routes each message to the matching queue based on this value. `lambda invoke` runs the function synchronously and writes its return value to the output file. Because the handler drains the high-priority queue before ever touching the low-priority one, the four invocations process both `high` messages first and only then the `low` ones.

The four invocations return, in order:

```
"Message 'High Priority message 2' deleted"
"Message 'High Priority message 1' deleted"
"Message 'Low Priority message 1' deleted"
"Message 'Low Priority message 2' deleted"
```

Both high-priority messages are processed before either low-priority one — success. (The two `high` messages come out as `2` then `1` rather than strictly in order because a standard SQS queue does not guarantee ordering *within* a queue; the priority *between* queues is what the design guarantees, and that holds.)

## Best Practices

- **Inline code vs. zip upload — pick based on the code.** CloudFormation's `AWS::Lambda::Function` requires a `Code` property, and inline `ZipFile` is great for tiny, dependency-free stubs. But the moment the code has multiple files, external dependencies, or tricky characters (nested quotes, backslashes) that a YAML block scalar can mangle, ship a real zip instead. Here we used a placeholder inline to satisfy the required property, then `update-function-code` with the untouched `/root/index.py` — the stack owns the infrastructure, the zip owns the code. (An even more "pure CloudFormation" alternative is `aws cloudformation package`, which uploads the zip to S3 and rewrites `Code` to `S3Bucket`/`S3Key` for you.)
- **Prefer managed policies in restricted sandboxes.** The KodeKloud lab user cannot create inline IAM policies (`iam:PutRolePolicy` is denied). Attaching AWS managed policies via `ManagedPolicyArns` grants the needed access without hitting that wall. In production, scope down to least-privilege customer-managed policies rather than `*FullAccess`.
- **Let SNS filter, don't filter in code.** Subscription filter policies push the routing decision into SNS, so each queue only ever holds messages of its priority. This keeps the consumer simple and avoids paying to receive-and-discard irrelevant messages.
- **Use SNS→SQS for decoupled fan-out, not because SQS needs it.** You can publish directly to SQS, but fronting the queues with an SNS topic lets producers publish once (tagged with an attribute) while the infrastructure routes and fans out. It decouples producers from the queue topology and lets one message reach many subscribers — reach for it when routing or multi-consumer delivery is a requirement, and skip it for a simple point-to-point queue.
- **Scope the queue policy to the source topic.** The `aws:SourceArn` condition ensures only *your* SNS topic can enqueue messages, not any principal that guesses the queue ARN — a small but important resource-policy hardening.
- **Acknowledge IAM capabilities explicitly.** Naming an IAM resource requires `CAPABILITY_NAMED_IAM`; treat that prompt as a reminder to review what identities and permissions your template creates.

### 📚 Official Documentation

- [Amazon SNS subscription filter policies](https://docs.aws.amazon.com/sns/latest/dg/sns-subscription-filter-policies.html)
- [Applying a subscription filter policy in Amazon SNS](https://docs.aws.amazon.com/sns/latest/dg/message-filtering-apply.html)
- [AWS::Lambda::Function Code (ZipFile)](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-properties-lambda-function-code.html)
- [CloudFormation CreateStack — Capabilities](https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_CreateStack.html)
- [AWS::SNS::Subscription](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-sns-subscription.html)
- [AWS::SQS::QueuePolicy](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-sqs-queuepolicy.html)
