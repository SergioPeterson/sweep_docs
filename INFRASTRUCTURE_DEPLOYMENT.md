# Infrastructure & Deployment

## Infrastructure Overview

### Cloud Provider: AWS
**Why AWS:**
- Mature ecosystem
- Extensive service catalog
- Best-in-class database (RDS/Aurora)
- Global CDN (CloudFront)
- Strong compliance certifications

**Primary Region:** `us-west-1` (N. California - closest to SF)
**DR Region:** `us-west-2` (Oregon)

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                 Route 53 (DNS)                          │
│              api.yourdomain.com                         │
│              app.yourdomain.com                         │
└──────────────────┬──────────────────────────────────────┘
                   │
        ┌──────────┴──────────┐
        │                     │
   ┌────▼─────┐         ┌────▼─────┐
   │CloudFront│         │   ALB    │
   │   (CDN)  │         │(API LB)  │
   └────┬─────┘         └────┬─────┘
        │                    │
   ┌────▼─────┐         ┌────▼──────────────┐
   │  Vercel  │         │ ECS Fargate      │
   │(Frontend)│         │ ┌──────────────┐ │
   └──────────┘         │ │ API Service  │ │
                        │ │ (2-4 tasks)  │ │
                        │ └──────────────┘ │
                        │ ┌──────────────┐ │
                        │ │Worker Service│ │
                        │ │ (1-2 tasks)  │ │
                        │ └──────────────┘ │
                        └─────┬────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
   ┌────▼────┐          ┌─────▼─────┐        ┌─────▼─────┐
   │   RDS   │          │   Redis   │        │    S3     │
   │Postgres │          │ElastiCache│        │  Bucket   │
   │Multi-AZ │          │           │        │           │
   └────┬────┘          └───────────┘        └───────────┘
        │
   ┌────▼────┐
   │  RDS    │
   │ Replica │
   │(Read)   │
   └─────────┘

External Services:
- Stripe (Payments)
- Twilio (SMS)
- SES (Email)
- Sentry (Error tracking)
- PostHog (Analytics)
```

---

## Compute Layer

### ECS Fargate (API Service)

**Why Fargate over EC2:**
- No server management
- Auto-scaling built-in
- Pay per use
- Fast deployment

**Task Definition (API Service):**
```json
{
  "family": "cleaning-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "123456789.dkr.ecr.us-west-1.amazonaws.com/cleaning-api:latest",
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        { "name": "BUN_ENV", "value": "production" },
        { "name": "PORT", "value": "3000" }
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:us-west-1:xxx:secret:db-url"
        },
        {
          "name": "STRIPE_SECRET_KEY",
          "valueFrom": "arn:aws:secretsmanager:us-west-1:xxx:secret:stripe-key"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/cleaning-api",
          "awslogs-region": "us-west-1",
          "awslogs-stream-prefix": "api"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3
      }
    }
  ]
}
```

**Auto-Scaling Configuration:**
```hcl
resource "aws_appautoscaling_target" "api" {
  service_namespace  = "ecs"
  resource_id        = "service/cleaning-cluster/cleaning-api"
  scalable_dimension = "ecs:service:DesiredCount"
  min_capacity       = 2
  max_capacity       = 10
}

resource "aws_appautoscaling_policy" "api_cpu" {
  name               = "api-cpu-scaling"
  service_namespace  = "ecs"
  resource_id        = aws_appautoscaling_target.api.resource_id
  scalable_dimension = "ecs:service:DesiredCount"

  target_tracking_scaling_policy_configuration {
    target_value       = 70.0
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
  }
}
```

---

### ECS Fargate (Worker Service)

**Task Definition (Worker):**
```json
{
  "family": "cleaning-worker",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "containerDefinitions": [
    {
      "name": "worker",
      "image": "123456789.dkr.ecr.us-west-1.amazonaws.com/cleaning-worker:latest",
      "environment": [
        { "name": "BUN_ENV", "value": "production" },
        { "name": "WORKER_TYPE", "value": "notifications" }
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:us-west-1:xxx:secret:db-url"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/cleaning-worker",
          "awslogs-region": "us-west-1",
          "awslogs-stream-prefix": "worker"
        }
      }
    }
  ]
}
```

**Worker Types:**
1. **Notifications Worker** - Email, SMS, push notifications
2. **Payouts Worker** - Process pending payouts
3. **Search Indexer** - Update search index (if using OpenSearch)
4. **Bonus Calculator** - Monthly bonus calculations

---

## Database Layer

### RDS PostgreSQL (Primary)

**Instance Configuration:**
```hcl
resource "aws_db_instance" "primary" {
  identifier             = "cleaning-db-primary"
  engine                 = "postgres"
  engine_version         = "15.4"
  instance_class         = "db.t4g.medium"  # 2 vCPU, 4 GB RAM
  allocated_storage      = 100              # GB
  storage_type           = "gp3"
  storage_encrypted      = true
  kms_key_id             = aws_kms_key.db.arn

  # Credentials
  username               = "cleaning_admin"
  password               = random_password.db_password.result

  # High Availability
  multi_az               = true
  db_subnet_group_name   = aws_db_subnet_group.private.name
  vpc_security_group_ids = [aws_security_group.db.id]

  # Backups
  backup_retention_period = 30
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"

  # Performance Insights
  performance_insights_enabled    = true
  performance_insights_retention_period = 7

  # Enhanced Monitoring
  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn

  # Deletion Protection
  deletion_protection = true
  skip_final_snapshot = false
  final_snapshot_identifier = "cleaning-db-final-snapshot"

  # Parameter Group (for PostGIS)
  parameter_group_name = aws_db_parameter_group.postgres15.name

  tags = {
    Name        = "cleaning-db-primary"
    Environment = "production"
  }
}
```

**Parameter Group (PostGIS + Performance):**
```hcl
resource "aws_db_parameter_group" "postgres15" {
  name   = "cleaning-postgres15"
  family = "postgres15"

  parameter {
    name  = "shared_preload_libraries"
    value = "pg_stat_statements,postgis"
  }

  parameter {
    name  = "max_connections"
    value = "200"
  }

  parameter {
    name  = "work_mem"
    value = "16384"  # 16 MB
  }

  parameter {
    name  = "maintenance_work_mem"
    value = "524288"  # 512 MB
  }

  parameter {
    name  = "effective_cache_size"
    value = "3145728"  # 3 GB (75% of instance memory)
  }
}
```

---

### RDS Read Replica

**Configuration:**
```hcl
resource "aws_db_instance" "replica" {
  identifier             = "cleaning-db-replica"
  replicate_source_db    = aws_db_instance.primary.id
  instance_class         = "db.t4g.medium"
  publicly_accessible    = false
  skip_final_snapshot    = true

  # Can be in different AZ for better read distribution
  availability_zone      = "us-west-1b"

  tags = {
    Name        = "cleaning-db-replica"
    Environment = "production"
  }
}
```

**Usage Pattern:**
- **Primary:** All writes, critical reads (bookings, payments)
- **Replica:** Analytics queries, dashboard stats, reports

---

## Caching Layer

### ElastiCache Redis

**Configuration:**
```hcl
resource "aws_elasticache_replication_group" "redis" {
  replication_group_id       = "cleaning-redis"
  replication_group_description = "Redis cluster for caching"

  engine                     = "redis"
  engine_version             = "7.0"
  node_type                  = "cache.t4g.micro"  # 0.5 GB RAM
  num_cache_clusters         = 2                   # Primary + 1 replica
  automatic_failover_enabled = true
  multi_az_enabled           = true

  subnet_group_name          = aws_elasticache_subnet_group.redis.name
  security_group_ids         = [aws_security_group.redis.id]

  # Backups
  snapshot_retention_limit   = 5
  snapshot_window            = "03:00-05:00"

  # Encryption
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true

  # Parameter Group
  parameter_group_name       = "default.redis7"

  tags = {
    Name        = "cleaning-redis"
    Environment = "production"
  }
}
```

**Scaling Plan:**
- Phase 1 (MVP): `cache.t4g.micro` (0.5 GB)
- Phase 2 (Growth): `cache.t4g.small` (1.5 GB)
- Phase 3 (Scale): `cache.r7g.large` (13 GB) + cluster mode

---

## Storage Layer

### S3 Buckets

**Bucket Structure:**
```
cleaning-production-uploads/
├── profile-images/
│   ├── cleaners/
│   └── customers/
├── booking-photos/
│   ├── before/
│   └── after/
├── documents/
│   ├── background-checks/
│   └── insurance/
└── receipts/
```

**Configuration:**
```hcl
resource "aws_s3_bucket" "uploads" {
  bucket = "cleaning-production-uploads"

  tags = {
    Name        = "cleaning-uploads"
    Environment = "production"
  }
}

# Versioning
resource "aws_s3_bucket_versioning" "uploads" {
  bucket = aws_s3_bucket.uploads.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "uploads" {
  bucket = aws_s3_bucket.uploads.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Lifecycle Policy (delete old versions after 30 days)
resource "aws_s3_bucket_lifecycle_configuration" "uploads" {
  bucket = aws_s3_bucket.uploads.id

  rule {
    id     = "delete-old-versions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days = 30
    }
  }
}

# CORS for direct uploads
resource "aws_s3_bucket_cors_configuration" "uploads" {
  bucket = aws_s3_bucket.uploads.id

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["PUT", "POST"]
    allowed_origins = ["https://app.yourdomain.com"]
    max_age_seconds = 3000
  }
}
```

**CloudFront Distribution (CDN):**
```hcl
resource "aws_cloudfront_distribution" "uploads" {
  enabled = true
  aliases = ["cdn.yourdomain.com"]

  origin {
    domain_name = aws_s3_bucket.uploads.bucket_regional_domain_name
    origin_id   = "S3-uploads"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.uploads.cloudfront_access_identity_path
    }
  }

  default_cache_behavior {
    target_origin_id       = "S3-uploads"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    min_ttl     = 0
    default_ttl = 86400    # 1 day
    max_ttl     = 31536000 # 1 year
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    acm_certificate_arn = aws_acm_certificate.cdn.arn
    ssl_support_method  = "sni-only"
  }
}
```

---

## Queue Layer

### SQS Queues

**Queue Configuration:**
```hcl
# Notifications Queue
resource "aws_sqs_queue" "notifications" {
  name                       = "cleaning-notifications"
  delay_seconds              = 0
  max_message_size           = 262144  # 256 KB
  message_retention_seconds  = 1209600 # 14 days
  receive_wait_time_seconds  = 20      # Long polling
  visibility_timeout_seconds = 300     # 5 minutes

  # Dead Letter Queue
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.notifications_dlq.arn
    maxReceiveCount     = 3
  })

  tags = {
    Name        = "cleaning-notifications"
    Environment = "production"
  }
}

# Dead Letter Queue
resource "aws_sqs_queue" "notifications_dlq" {
  name                      = "cleaning-notifications-dlq"
  message_retention_seconds = 1209600 # 14 days
}

# Payouts Queue
resource "aws_sqs_queue" "payouts" {
  name                       = "cleaning-payouts"
  visibility_timeout_seconds = 600  # 10 minutes (longer for payout processing)

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.payouts_dlq.arn
    maxReceiveCount     = 5  # More retries for payment operations
  })
}

resource "aws_sqs_queue" "payouts_dlq" {
  name = "cleaning-payouts-dlq"
}
```

---

## Scheduled Jobs (EventBridge + ECS)

### EventBridge Scheduler

Scheduled jobs run as one-off ECS Fargate tasks triggered by EventBridge rules.

```hcl
# Hold cleanup - every 1 minute
resource "aws_cloudwatch_event_rule" "hold_cleanup" {
  name                = "cleaning-hold-cleanup"
  schedule_expression = "rate(1 minute)"
}

resource "aws_cloudwatch_event_target" "hold_cleanup" {
  rule     = aws_cloudwatch_event_rule.hold_cleanup.name
  arn      = aws_ecs_cluster.main.arn
  role_arn = aws_iam_role.eventbridge_ecs.arn

  ecs_target {
    task_definition_arn = aws_ecs_task_definition.job_runner.arn
    launch_type         = "FARGATE"
    task_count          = 1
    network_configuration {
      subnets          = [aws_subnet.private_a.id, aws_subnet.private_b.id]
      security_groups  = [aws_security_group.ecs.id]
      assign_public_ip = false
    }
  }

  input = jsonencode({
    containerOverrides = [{
      name    = "job-runner"
      command = ["bun", "run", "workers/job-runner.ts", "hold-cleanup"]
    }]
  })
}

# Booking reminders - every 15 minutes
resource "aws_cloudwatch_event_rule" "booking_reminders" {
  name                = "cleaning-booking-reminders"
  schedule_expression = "rate(15 minutes)"
}

resource "aws_cloudwatch_event_target" "booking_reminders" {
  rule     = aws_cloudwatch_event_rule.booking_reminders.name
  arn      = aws_ecs_cluster.main.arn
  role_arn = aws_iam_role.eventbridge_ecs.arn

  ecs_target {
    task_definition_arn = aws_ecs_task_definition.job_runner.arn
    launch_type         = "FARGATE"
    task_count          = 1
    network_configuration {
      subnets          = [aws_subnet.private_a.id, aws_subnet.private_b.id]
      security_groups  = [aws_security_group.ecs.id]
      assign_public_ip = false
    }
  }

  input = jsonencode({
    containerOverrides = [{
      name    = "job-runner"
      command = ["bun", "run", "workers/job-runner.ts", "booking-reminders"]
    }]
  })
}

# Payment reconciliation - every 5 minutes
resource "aws_cloudwatch_event_rule" "payment_reconciliation" {
  name                = "cleaning-payment-reconciliation"
  schedule_expression = "rate(5 minutes)"
}

resource "aws_cloudwatch_event_target" "payment_reconciliation" {
  rule     = aws_cloudwatch_event_rule.payment_reconciliation.name
  arn      = aws_ecs_cluster.main.arn
  role_arn = aws_iam_role.eventbridge_ecs.arn

  ecs_target {
    task_definition_arn = aws_ecs_task_definition.job_runner.arn
    launch_type         = "FARGATE"
    task_count          = 1
    network_configuration {
      subnets          = [aws_subnet.private_a.id, aws_subnet.private_b.id]
      security_groups  = [aws_security_group.ecs.id]
      assign_public_ip = false
    }
  }

  input = jsonencode({
    containerOverrides = [{
      name    = "job-runner"
      command = ["bun", "run", "workers/job-runner.ts", "payment-reconciliation"]
    }]
  })
}

# Monthly bonus calculation - 1st of each month at 3 AM PT
resource "aws_cloudwatch_event_rule" "bonus_calculator" {
  name                = "cleaning-bonus-calculator"
  schedule_expression = "cron(0 11 1 * ? *)"  # 3 AM PT = 11 AM UTC
}

resource "aws_cloudwatch_event_target" "bonus_calculator" {
  rule     = aws_cloudwatch_event_rule.bonus_calculator.name
  arn      = aws_ecs_cluster.main.arn
  role_arn = aws_iam_role.eventbridge_ecs.arn

  ecs_target {
    task_definition_arn = aws_ecs_task_definition.job_runner.arn
    launch_type         = "FARGATE"
    task_count          = 1
    network_configuration {
      subnets          = [aws_subnet.private_a.id, aws_subnet.private_b.id]
      security_groups  = [aws_security_group.ecs.id]
      assign_public_ip = false
    }
  }

  input = jsonencode({
    containerOverrides = [{
      name    = "job-runner"
      command = ["bun", "run", "workers/job-runner.ts", "bonus-calculator"]
    }]
  })
}
```

### Job Runner Task Definition

```hcl
resource "aws_ecs_task_definition" "job_runner" {
  family                   = "cleaning-job-runner"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([{
    name  = "job-runner"
    image = "${aws_ecr_repository.api.repository_url}:latest"
    command = ["bun", "run", "workers/job-runner.ts"]
    secrets = [
      { name = "DATABASE_URL", valueFrom = aws_secretsmanager_secret.db_url.arn },
      { name = "STRIPE_SECRET_KEY", valueFrom = aws_secretsmanager_secret.stripe_key.arn }
    ]
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = "/ecs/cleaning-jobs"
        "awslogs-region"        = "us-west-1"
        "awslogs-stream-prefix" = "job"
      }
    }
  }])
}
```

### Job Failure Alarms

```hcl
resource "aws_cloudwatch_metric_alarm" "job_failures" {
  alarm_name          = "cleaning-job-failures"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "FailedInvocations"
  namespace           = "AWS/Events"
  period              = 300
  statistic           = "Sum"
  threshold           = 3
  alarm_description   = "Scheduled job failed 3+ times in 5 min"
  alarm_actions       = [aws_sns_topic.alerts.arn]
}
```

---

## Networking

### VPC Configuration

```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "cleaning-vpc"
  }
}

# Public Subnets (ALB, NAT Gateway)
resource "aws_subnet" "public_a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-west-1a"

  tags = { Name = "public-a" }
}

resource "aws_subnet" "public_b" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-west-1b"

  tags = { Name = "public-b" }
}

# Private Subnets (ECS, RDS, Redis)
resource "aws_subnet" "private_a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.10.0/24"
  availability_zone = "us-west-1a"

  tags = { Name = "private-a" }
}

resource "aws_subnet" "private_b" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.11.0/24"
  availability_zone = "us-west-1b"

  tags = { Name = "private-b" }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = { Name = "cleaning-igw" }
}

# NAT Gateway (for outbound internet from private subnets)
resource "aws_eip" "nat" {
  domain = "vpc"
}

resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public_a.id

  tags = { Name = "cleaning-nat" }
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = { Name = "public-rt" }
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }

  tags = { Name = "private-rt" }
}
```

---

### Security Groups

**ALB Security Group:**
```hcl
resource "aws_security_group" "alb" {
  name   = "cleaning-alb-sg"
  vpc_id = aws_vpc.main.id

  # Allow HTTPS from internet
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Allow HTTP (redirect to HTTPS)
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

**ECS Security Group:**
```hcl
resource "aws_security_group" "ecs" {
  name   = "cleaning-ecs-sg"
  vpc_id = aws_vpc.main.id

  # Allow traffic from ALB
  ingress {
    from_port       = 3000
    to_port         = 3000
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

**RDS Security Group:**
```hcl
resource "aws_security_group" "db" {
  name   = "cleaning-db-sg"
  vpc_id = aws_vpc.main.id

  # Allow PostgreSQL from ECS
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.ecs.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## Load Balancer

### Application Load Balancer

```hcl
resource "aws_lb" "api" {
  name               = "cleaning-api-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = [aws_subnet.public_a.id, aws_subnet.public_b.id]

  enable_deletion_protection = true
  enable_http2              = true

  tags = {
    Name = "cleaning-api-alb"
  }
}

# HTTPS Listener
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.api.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.api.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }
}

# HTTP Listener (redirect to HTTPS)
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.api.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# Target Group
resource "aws_lb_target_group" "api" {
  name        = "cleaning-api-tg"
  port        = 3000
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    enabled             = true
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
    matcher             = "200"
  }

  deregistration_delay = 30
}
```

---

## Frontend Hosting (Vercel)

**Why Vercel:**
- Best Next.js support (made by same team)
- Global Edge Network
- Automatic preview deployments
- Zero-config SSL
- Fast cold starts

**Configuration:**
```json
{
  "buildCommand": "npm run build",
  "outputDirectory": ".next",
  "framework": "nextjs",
  "installCommand": "npm install",
  "env": {
    "NEXT_PUBLIC_API_URL": "https://api.yourdomain.com/v1",
    "NEXT_PUBLIC_STRIPE_KEY": "@stripe-publishable-key",
    "NEXT_PUBLIC_MAPBOX_TOKEN": "@mapbox-token"
  },
  "regions": ["sfo1", "iad1"],
  "rewrites": [
    {
      "source": "/api/:path*",
      "destination": "https://api.yourdomain.com/v1/:path*"
    }
  ]
}
```

**Alternative: AWS Amplify**
If you prefer AWS-native:
```hcl
resource "aws_amplify_app" "frontend" {
  name       = "cleaning-frontend"
  repository = "https://github.com/yourorg/cleaning-frontend"

  environment_variables = {
    NEXT_PUBLIC_API_URL = "https://api.yourdomain.com/v1"
  }

  custom_rule {
    source = "/<*>"
    status = "404"
    target = "/index.html"
  }
}
```

---

## Secrets Management

### AWS Secrets Manager

```hcl
# Database URL
resource "aws_secretsmanager_secret" "db_url" {
  name = "cleaning/production/database-url"
}

resource "aws_secretsmanager_secret_version" "db_url" {
  secret_id     = aws_secretsmanager_secret.db_url.id
  secret_string = "postgresql://${aws_db_instance.primary.username}:${random_password.db_password.result}@${aws_db_instance.primary.endpoint}/cleaning"
}

# Stripe Secret Key
resource "aws_secretsmanager_secret" "stripe_key" {
  name = "cleaning/production/stripe-secret-key"
}

# Clerk JWT verification config
resource "aws_secretsmanager_secret" "clerk_jwks_url" {
  name = "cleaning/production/clerk-jwks-url"
}

resource "aws_secretsmanager_secret" "clerk_issuer" {
  name = "cleaning/production/clerk-issuer"
}

resource "aws_secretsmanager_secret" "clerk_audience" {
  name = "cleaning/production/clerk-audience"
}
```

**IAM Policy for ECS Tasks:**
```hcl
resource "aws_iam_policy" "secrets_access" {
  name = "cleaning-secrets-access"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue"
        ]
        Resource = [
          aws_secretsmanager_secret.db_url.arn,
          aws_secretsmanager_secret.stripe_key.arn,
          aws_secretsmanager_secret.clerk_jwks_url.arn,
          aws_secretsmanager_secret.clerk_issuer.arn,
          aws_secretsmanager_secret.clerk_audience.arn
        ]
      }
    ]
  })
}
```

---

## CI/CD Pipeline

### GitHub Actions

**.github/workflows/deploy-api.yml:**
```yaml
name: Deploy API

on:
  push:
    branches: [main]
    paths:
      - 'backend/**'

env:
  AWS_REGION: us-west-1
  ECR_REPOSITORY: cleaning-api
  ECS_SERVICE: cleaning-api
  ECS_CLUSTER: cleaning-cluster
  CONTAINER_NAME: api

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build, tag, and push image to Amazon ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          cd backend
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

      - name: Download task definition
        run: |
          aws ecs describe-task-definition \
            --task-definition cleaning-api \
            --query taskDefinition > task-definition.json

      - name: Fill in the new image ID in the task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ steps.build-image.outputs.image }}

      - name: Deploy to Amazon ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true

      - name: Notify Slack
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "API Deployment ${{ job.status }}: ${{ github.sha }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

**.github/workflows/deploy-frontend.yml:**
```yaml
name: Deploy Frontend

on:
  push:
    branches: [main]
    paths:
      - 'apps/web/**'

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'
          working-directory: ./apps/web
```

---

## Monitoring & Observability

### CloudWatch Dashboards

**Key Metrics:**
```hcl
resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = "cleaning-production"

  dashboard_body = jsonencode({
    widgets = [
      {
        type = "metric"
        properties = {
          metrics = [
            ["AWS/ECS", "CPUUtilization", { stat = "Average" }],
            [".", "MemoryUtilization", { stat = "Average" }]
          ]
          period = 300
          stat   = "Average"
          region = "us-west-1"
          title  = "ECS CPU/Memory"
        }
      },
      {
        type = "metric"
        properties = {
          metrics = [
            ["AWS/RDS", "DatabaseConnections", { stat = "Average" }],
            [".", "CPUUtilization", { stat = "Average" }]
          ]
          period = 300
          region = "us-west-1"
          title  = "RDS Metrics"
        }
      }
    ]
  })
}
```

### Alarms

**Critical Alarms:**
```hcl
# High CPU
resource "aws_cloudwatch_metric_alarm" "ecs_cpu_high" {
  alarm_name          = "cleaning-ecs-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ECS"
  period              = 300
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "ECS CPU > 80%"
  alarm_actions       = [aws_sns_topic.alerts.arn]
}

# Database connections near limit
resource "aws_cloudwatch_metric_alarm" "db_connections_high" {
  alarm_name          = "cleaning-db-connections-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "DatabaseConnections"
  namespace           = "AWS/RDS"
  period              = 300
  statistic           = "Average"
  threshold           = 180  # 90% of max_connections (200)
  alarm_description   = "RDS connections > 180"
  alarm_actions       = [aws_sns_topic.alerts.arn]
}

# ALB 5XX errors
resource "aws_cloudwatch_metric_alarm" "alb_5xx" {
  alarm_name          = "cleaning-alb-5xx-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "HTTPCode_Target_5XX_Count"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  statistic           = "Sum"
  threshold           = 10
  alarm_description   = "More than 10 5XX errors in 1 minute"
  alarm_actions       = [aws_sns_topic.alerts.arn]
}
```

**SNS Topic for Alerts:**
```hcl
resource "aws_sns_topic" "alerts" {
  name = "cleaning-alerts"
}

resource "aws_sns_topic_subscription" "slack" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "https"
  endpoint  = "https://hooks.slack.com/services/xxx"
}

resource "aws_sns_topic_subscription" "pagerduty" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "https"
  endpoint  = "https://events.pagerduty.com/integration/xxx"
}
```

---

### Application Monitoring (Sentry)

**Backend Integration:**
```typescript
import * as Sentry from '@sentry/bun'

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.BUN_ENV,
  tracesSampleRate: 0.1,  // 10% of transactions
})

// Use in Elysia error handler (see BACKEND_ARCHITECTURE.md)
```

**Frontend Integration:**
```typescript
import * as Sentry from '@sentry/nextjs'

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1,
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0
})
```

---

## Disaster Recovery

### Backup Strategy

**RDS Automated Backups:**
- Daily snapshots at 3 AM
- 30-day retention
- Cross-region replication to `us-west-2`

**Manual Backup (before major changes):**
```bash
aws rds create-db-snapshot \
  --db-instance-identifier cleaning-db-primary \
  --db-snapshot-identifier cleaning-db-$(date +%Y%m%d-%H%M%S)
```

### Recovery Procedures

**RDS Point-in-Time Recovery:**
```bash
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier cleaning-db-primary \
  --target-db-instance-identifier cleaning-db-restored \
  --restore-time 2024-01-15T10:30:00Z
```

**S3 Versioning Recovery:**
```bash
aws s3api list-object-versions \
  --bucket cleaning-production-uploads \
  --prefix profile-images/

aws s3api restore-object \
  --bucket cleaning-production-uploads \
  --key profile-images/uuid.jpg \
  --version-id xxx
```

---

## Cost Estimation

### Phase 1: MVP (0-500 bookings/day)

| Service | Configuration | Monthly Cost |
|---------|--------------|--------------|
| ECS Fargate (API) | 2 tasks × 1vCPU, 2GB | $60 |
| ECS Fargate (Worker) | 1 task × 0.5vCPU, 1GB | $15 |
| RDS PostgreSQL | db.t4g.medium, Multi-AZ | $120 |
| ElastiCache Redis | cache.t4g.micro | $15 |
| S3 + CloudFront | 100GB storage, 1TB transfer | $20 |
| ALB | 1 load balancer | $20 |
| NAT Gateway | 1 gateway, 100GB data | $50 |
| Secrets Manager | 5 secrets | $2 |
| CloudWatch | Logs + metrics | $30 |
| **Vercel** | Pro plan | $20 |
| **Stripe** | 2.9% + $0.30/transaction | Variable |
| **Twilio** | SMS ($0.0075/msg) | $50 |
| **SES** | Email ($0.10/1000) | $5 |
| **Sentry** | Team plan | $26 |
| **Total (fixed)** | | **~$433/month** |

### Phase 2: Growth (500-5,000 bookings/day)

| Service | Configuration | Monthly Cost |
|---------|--------------|--------------|
| ECS Fargate (API) | 4 tasks × 1vCPU, 2GB | $120 |
| RDS PostgreSQL | db.r6g.large + replica | $400 |
| ElastiCache Redis | cache.t4g.small | $30 |
| S3 + CloudFront | 500GB storage, 5TB transfer | $80 |
| OpenSearch | 2 × t3.medium.search | $120 |
| **Total (fixed)** | | **~$1,200/month** |

### Phase 3: Scale (5,000+ bookings/day)

Estimated **$5,000-10,000/month** depending on traffic.

---

## Security Hardening

### WAF (Web Application Firewall)

```hcl
resource "aws_wafv2_web_acl" "api" {
  name  = "cleaning-api-waf"
  scope = "REGIONAL"

  default_action {
    allow {}
  }

  # Rate limiting
  rule {
    name     = "rate-limit"
    priority = 1

    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }

    action {
      block {}
    }
  }

  # AWS Managed Rules
  rule {
    name     = "aws-managed-rules"
    priority = 2

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }
  }
}
```

### SSL/TLS Certificates

```hcl
resource "aws_acm_certificate" "api" {
  domain_name       = "api.yourdomain.com"
  validation_method = "DNS"

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_acm_certificate" "cdn" {
  provider          = aws.us-east-1  # CloudFront requires us-east-1
  domain_name       = "cdn.yourdomain.com"
  validation_method = "DNS"
}
```

---

## Runbook

### Common Operations

**Deploy new API version:**
```bash
# 1. Build and push image
docker build -t cleaning-api:v1.2.0 .
docker tag cleaning-api:v1.2.0 123456789.dkr.ecr.us-west-1.amazonaws.com/cleaning-api:v1.2.0
docker push 123456789.dkr.ecr.us-west-1.amazonaws.com/cleaning-api:v1.2.0

# 2. Update ECS service
aws ecs update-service \
  --cluster cleaning-cluster \
  --service cleaning-api \
  --force-new-deployment
```

**Database migration:**
```bash
# Connect to bastion host
ssh -i bastion-key.pem ec2-user@bastion.yourdomain.com

# Run migration
cd /app/backend
bun prisma migrate deploy
```

**Scale up ECS service:**
```bash
aws ecs update-service \
  --cluster cleaning-cluster \
  --service cleaning-api \
  --desired-count 6
```

**View logs:**
```bash
aws logs tail /ecs/cleaning-api --follow
```

---

## Next Steps

1. **Provision infrastructure** using Terraform
2. **Set up CI/CD** pipelines in GitHub Actions
3. **Configure monitoring** dashboards and alerts
4. **Deploy backend** to ECS
5. **Deploy frontend** to Vercel
6. **Run smoke tests** on production
7. **Set up on-call** rotation (PagerDuty)

See related docs:
- [System Design Overview](./SYSTEM_DESIGN_OVERVIEW.md)
- [Backend Architecture](./BACKEND_ARCHITECTURE.md)
- [Frontend Architecture](./FRONTEND_ARCHITECTURE.md)
- [Database Schema](./DATABASE_SCHEMA.md)
- [API Specification](./API_SPECIFICATION.md)
