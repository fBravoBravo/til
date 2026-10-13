# OIDC-Based Cross-Cloud Authentication (AWS ↔ GCP)

## Problem

I needed a service running in GCP to securely invoke an AWS API. Classic approach: create long-lived IAM user credentials, store them as secrets in GCP Secret Manager, rotate them periodically.

Better approach: **OIDC federation** — no static creds at all.

## How It Works

1. **GCP Service Account** mints an OIDC token via its metadata server.
2. **AWS IAM** trusts that token via an `aws_iam_openid_connect_provider`.
3. **AssumeRole** happens on every request. Token expires in ~1 hour.

## Terraform (AWS side)

```hcl
resource "aws_iam_openid_connect_provider" "gcp" {
  url             = "https://accounts.google.com"
  client_id_list  = [var.gcp_service_account_email]
  thumbprint_list = ["087454f1a8e2b3e6e4c5d6e7f8a9b0c1d2e3f4a5"]
}

resource "aws_iam_role" "cross_cloud_role" {
  name = "gcp-to-aws-access"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.gcp.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "accounts.google.com:sub" = var.gcp_service_account_unique_id
        }
      }
    }]
  })
}
```

## Key Insight

The `thumbprint_list` in Terraform needs periodic updating when Google's root CA rotates. AWS used to auto-resolve this; now it requires manual or automated thumbprint fetching.

I now run a monthly GitHub Action that:
1. Fetches the current Google cert chain thumbprint
2. Opens a PR if it changed
3. Terraform plan/apply handles the rest

## Benefit

Zero long-lived credentials. Zero secret rotation. Zero cross-cloud attack surface from leaked static keys.

## See Also

- `terraform/terraform-immutable-infrastructure-parity.md` (the CI/CD pipeline that automates this)
