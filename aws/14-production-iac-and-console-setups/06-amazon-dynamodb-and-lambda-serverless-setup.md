# Production DynamoDB & Serverless Lambda API: Terraform & Console Setup

Building scalable, event-driven microservices without server overhead is standard practice in cloud engineering. This guide implements an end-to-end serverless REST API: an **Amazon API Gateway HTTP API** routing requests to an **AWS Lambda function** running inside a private VPC, securely reading and writing to an **Amazon DynamoDB** table with Point-in-Time Recovery (PITR) enabled.

---

## Architecture Flow

```
[ Web / Mobile Client ]
           |
           v (HTTPS POST /orders)
[ Amazon API Gateway (HTTP API v2) ] (CORS enabled, JWT Auth optional)
           |
           v (Lambda Proxy Integration)
[ AWS Lambda Function ] (Python 3.11 / ARM64 Graviton)
  ├── Runs inside Private App Subnets (Hyperplane ENI)
  ├── Outbound egress via NAT Gateway
  └── IAM Execution Role with least-privilege access
           |
           v (Single-digit ms PutItem/GetItem)
[ Amazon DynamoDB Table ] (On-Demand Capacity Mode, PITR Enabled, KMS Encrypted)
```

---

## Part 1: Production Terraform Implementation

### 1.1 `variables.tf`
```hcl
variable "aws_region" {
  description = "Target AWS Region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "production"
}

variable "vpc_id" {
  description = "VPC ID where Lambda function will run"
  type        = string
}

variable "app_subnet_ids" {
  description = "Private Subnet IDs for Lambda Hyperplane ENI"
  type        = list(string)
}
```

### 1.2 `main.tf`
```hcl
# ------------------------------------------------------------------------------
# 1. Amazon DynamoDB Table (On-Demand, PITR, KMS Encrypted)
# ------------------------------------------------------------------------------
resource "aws_dynamodb_table" "orders_table" {
  name         = "${var.environment}-orders-table"
  billing_mode = "PAY_PER_REQUEST" # On-Demand Capacity Mode
  hash_key     = "OrderId"
  range_key    = "CreatedAt"

  attribute {
    name = "OrderId"
    type = "S"
  }

  attribute {
    name = "CreatedAt"
    type = "S"
  }

  point_in_time_recovery {
    enabled = true # Enables continuous backups with 35-day restore window
  }

  server_side_encryption {
    enabled     = true
    kms_key_arn = "alias/aws/dynamodb"
  }

  ttl {
    attribute_name = "TimeToLive"
    enabled        = true
  }

  tags = {
    Name        = "${var.environment}-orders-table"
    Environment = var.environment
  }
}

# ------------------------------------------------------------------------------
# 2. Lambda Security Group
# ------------------------------------------------------------------------------
resource "aws_security_group" "lambda_sg" {
  name        = "${var.environment}-lambda-sg"
  description = "Security group for VPC-attached Lambda function"
  vpc_id      = var.vpc_id

  egress {
    description = "Allow all outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.environment}-lambda-sg" }
}

# ------------------------------------------------------------------------------
# 3. IAM Execution Role for Lambda (VPC Access + DynamoDB Least Privilege)
# ------------------------------------------------------------------------------
resource "aws_iam_role" "lambda_role" {
  name_prefix = "orders-lambda-role-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action    = "sts:AssumeRole"
        Effect    = "Allow"
        Principal = { Service = "lambda.amazonaws.com" }
      }
    ]
  })
}

# AWS Managed Policy for creating and managing Hyperplane ENIs in VPC
resource "aws_iam_role_policy_attachment" "lambda_vpc_access" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}

# Inline Least-Privilege Policy for DynamoDB Operations
resource "aws_iam_role_policy" "lambda_dynamodb_policy" {
  name_prefix = "lambda-dynamodb-orders-"
  role        = aws_iam_role.lambda_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "dynamodb:PutItem",
          "dynamodb:GetItem",
          "dynamodb:UpdateItem",
          "dynamodb:Query"
        ]
        Resource = [
          aws_dynamodb_table.orders_table.arn,
          "${aws_dynamodb_table.orders_table.arn}/index/*"
        ]
      }
    ]
  })
}

# ------------------------------------------------------------------------------
# 4. AWS Lambda Function
# ------------------------------------------------------------------------------
# In-line Python code for demonstration
data "archive_file" "lambda_zip" {
  type        = "zip"
  output_path = "${path.module}/lambda_function.zip"
  source {
    content  = <<EOF
import json
import os
import time
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['TABLE_NAME'])

def handler(event, context):
    try:
        body = json.loads(event.get('body', '{}'))
        order_id = body.get('order_id', f"ord-{int(time.time()*1000)}")
        
        item = {
            'OrderId': order_id,
            'CreatedAt': str(int(time.time())),
            'CustomerId': body.get('customer_id', 'anon'),
            'Amount': str(body.get('amount', '0.00')),
            'TimeToLive': int(time.time()) + (30 * 86400) # 30-day TTL
        }
        
        table.put_item(Item=item)
        
        return {
            'statusCode': 201,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({'message': 'Order Created', 'order': item})
        }
    except Exception as e:
        return {
            'statusCode': 500,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({'error': str(e)})
        }
EOF
    filename = "index.py"
  }
}

resource "aws_lambda_function" "orders_lambda" {
  function_name = "${var.environment}-create-order"
  role          = aws_iam_role.lambda_role.arn
  handler       = "index.handler"
  runtime       = "python3.11"
  architectures = ["arm64"] # AWS Graviton (cost & performance advantage)
  timeout       = 10
  memory_size   = 256

  filename         = data.archive_file.lambda_zip.output_path
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  # VPC Configuration (Attaches Lambda to Private Subnets)
  vpc_config {
    subnet_ids         = var.app_subnet_ids
    security_group_ids = [aws_security_group.lambda_sg.id]
  }

  environment {
    variables = {
      TABLE_NAME  = aws_dynamodb_table.orders_table.name
      ENVIRONMENT = var.environment
    }
  }

  depends_on = [aws_iam_role_policy_attachment.lambda_vpc_access]
}

# ------------------------------------------------------------------------------
# 5. Amazon API Gateway HTTP API (v2)
# ------------------------------------------------------------------------------
resource "aws_apigatewayv2_api" "http_api" {
  name          = "${var.environment}-orders-api"
  protocol_type = "HTTP"

  cors_configuration {
    allow_origins = ["*"]
    allow_methods = ["POST", "GET", "OPTIONS"]
    allow_headers = ["Content-Type", "Authorization"]
    max_age       = 300
  }
}

resource "aws_apigatewayv2_stage" "default_stage" {
  api_id      = aws_apigatewayv2_api.http_api.id
  name        = "$default"
  auto_deploy = true
}

resource "aws_apigatewayv2_integration" "lambda_integration" {
  api_id                 = aws_apigatewayv2_api.http_api.id
  integration_type       = "AWS_PROXY"
  integration_uri        = aws_lambda_function.orders_lambda.invoke_arn
  payload_format_version = "2.0"
}

resource "aws_apigatewayv2_route" "create_order_route" {
  api_id    = aws_apigatewayv2_api.http_api.id
  route_key = "POST /orders"
  target    = "integrations/${aws_apigatewayv2_integration.lambda_integration.id}"
}

# Grant API Gateway permission to invoke Lambda
resource "aws_lambda_permission" "apigw_invoke" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.orders_lambda.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_apigatewayv2_api.http_api.execution_arn}/*/*"
}
```

### 1.3 `outputs.tf`
```hcl
output "api_endpoint" {
  description = "The public base URL of the HTTP API"
  value       = aws_apigatewayv2_stage.default_stage.invoke_url
}

output "dynamodb_table_name" {
  description = "Name of the provisioned DynamoDB table"
  value       = aws_dynamodb_table.orders_table.name
}

output "lambda_function_arn" {
  description = "ARN of the provisioned Lambda function"
  value       = aws_lambda_function.orders_lambda.arn
}
```

---

## Part 2: AWS Management Console Step-by-Step Clickthrough Guide

### Step 1: Create the DynamoDB Table
1. Open the **Amazon DynamoDB Console** (`https://console.aws.amazon.com/dynamodb/`).
2. Click **Create table**.
3. **Table name:** `production-orders-table`.
4. **Partition key (HASH):** `OrderId` (Type: **String**).
5. **Sort key (RANGE):** `CreatedAt` (Type: **String**).
6. **Table settings:** Select **Customize settings**.
   * Capacity mode: Select **On-demand**.
   * Secondary indexes: None (can be added later).
   * Encryption at rest: Select **Amazon DynamoDB managed key**.
7. Click **Create table**.
8. Once active, go to the **Backups** tab, click **Edit** under **Point-in-time recovery (PITR)**, check **Turn on Point-in-time recovery**, and click **Save changes**.

---

### Step 2: Create IAM Role for Lambda
1. Open the **IAM Console** -> **Roles** -> **Create role**.
2. Select **AWS service** -> Use case: **Lambda**.
3. Attach policy: Search for and select: **`AWSLambdaVPCAccessExecutionRole`**.
4. Name the role: `production-orders-lambda-role`. Click **Create role**.
5. Select the role, click **Add permissions** -> **Create inline policy** -> JSON tab.
6. Paste the DynamoDB least-privilege policy allowing `PutItem`, `GetItem` on `production-orders-table`. Name it `DynamoDBAccess` and click **Create policy**.

---

### Step 3: Create the VPC-Attached Lambda Function
1. Open the **AWS Lambda Console** (`https://console.aws.amazon.com/lambda/`).
2. Click **Create function**.
3. Select **Author from scratch**.
4. Function name: `production-create-order`.
5. Runtime: **Python 3.11**. Architecture: **arm64**.
6. Permissions: Select **Use an existing role**, and pick `production-orders-lambda-role`.
7. Expand **Advanced settings**:
   * Check **Enable VPC**.
   * VPC: Choose `production-core-vpc`.
   * Subnets: Select both **Private Application Subnets**.
   * Security groups: Select `production-lambda-sg`.
8. Click **Create function**.
9. Under the **Code** tab, paste the Python handler code and click **Deploy**.
10. Under **Configuration** -> **Environment variables**, click **Edit** and add `TABLE_NAME` = `production-orders-table`. Click **Save**.

---

### Step 4: Create API Gateway HTTP API
1. Open **Amazon API Gateway Console** -> Click **Create API**.
2. Under **HTTP API**, click **Build**.
3. Name: `production-orders-api`. Click **Next**.
4. Configure routes: Click **Add route**. Method: **POST**, Resource path: `/orders`. Click **Next**.
5. Attach integrations: Select **Lambda**, and pick `production-create-order`.
6. Configure stages: Keep `$default` (Auto-deploy enabled). Click **Next**, then **Create**.

---

### Step 5: Live Testing
Execute a live HTTP POST to your newly provisioned serverless endpoint:
```bash
API_URL="https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com"
curl -X POST "${API_URL}/orders" \
    -H "Content-Type: application/json" \
    -d '{"customer_id": "cust-8421", "amount": 149.99}'

# Expected output:
# {"message": "Order Created", "order": {"OrderId": "ord-...", "Amount": "149.99", ...}}
```
