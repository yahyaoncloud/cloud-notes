# AWS Cloud Security Operations
### AWS WAF • Amazon GuardDuty • AWS CloudTrail • Amazon CloudWatch

---

## 1. AWS Security Service Matrix

| Service | Primary Function | Core Telemetry Analyzed | SOC / Detection Value |
| :--- | :--- | :--- | :--- |
| **AWS WAF** | Web Application Firewall | Layer 7 HTTP/S headers, body, URI, IP reputation | OWASP Top 10 blocking, SQLi, XSS, rate-limiting (`RateBasedRule`). |
| **Amazon GuardDuty** | Intelligent Threat Detection | VPC Flow Logs, DNS Logs, CloudTrail Events, EKS Audit | Machine learning anomaly detection, C2 beaconing, port probing, compromised IAM credentials. |
| **AWS CloudTrail** | Management & Data Audit Trail | API calls across Console, CLI, SDKs | Historical forensics: Who invoked which API, from what IP, at what timestamp. |
| **Amazon CloudWatch** | Monitoring & Alerting Engine | CPU, Memory, Disk, Network I/O, Custom App Logs | Metric alarms, log metric filters, event-driven automation via EventBridge. |

---

## 2. AWS WAF Web ACL Configuration Example (Terraform)

```hcl
resource "aws_wafv2_web_acl" "main" {
  name        = "production-web-acl"
  scope       = "REGIONAL"
  description = "OWASP Top 10 Protection and Rate Limiting"

  default_action {
    allow {}
  }

  rule {
    name     = "AWS-AWSManagedRulesCommonRuleSet"
    priority = 10
    override_action { none {} }
    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "CommonRuleSetMetric"
      sampled_requests_enabled   = true
    }
  }

  rule {
    name     = "RateLimitPerIP"
    priority = 20
    action { block {} }
    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimitMetric"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "ProductionWebACLMetric"
    sampled_requests_enabled   = true
  }
}
```
