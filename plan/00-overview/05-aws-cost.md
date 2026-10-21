# 05 — AWS cost estimate

Architecture: Go ETL in GitHub Actions + static site on S3/CloudFront
(D6, option B). Prices: AWS `us-east-1`, October 2026.

## Monthly estimate

| Item | Expected usage | Cost / month |
|---|---|---|
| S3 storage | site (~1 MB) + 5 kept data versions × (JSON ~2 MB + photos ~2.5 MB + SQLite ~10–30 MB) ≈ < 200 MB | < US$0.01 |
| S3 requests | ~4 refreshes × ~150 PUTs + site deploys + CloudFront origin GETs | < US$0.01 |
| CloudFront data transfer + requests | **Always-free tier: 1 TB out and 10 M requests per month** | US$0.00 |
| CloudFront Function | always-free tier: 2 M invocations per month | US$0.00 |
| CloudFront invalidations | first 1,000 paths per month free; we invalidate 1–2 paths per run | US$0.00 |
| IAM, OIDC | free | US$0.00 |
| GitHub Actions | public repository → free minutes; the large downloads happen on GitHub's runner, not in AWS | US$0.00 |
| **Total** | | **≈ US$0.01 – 0.10** |

## Stress scenario

Election-week spike: 100,000 visits × ~2 MB each ≈ 200 GB → still inside the
1 TB free tier. Static files cost no compute, so traffic cannot run up a bill
the way a server or database can.

## Optional extras

| Extra | Cost |
|---|---|
| Custom domain `.com.br` | ~R$40/year at registro.br |
| Route 53 hosted zone for that domain | US$0.50/month |
| ACM certificate for CloudFront | free |

## Guard rails

The AWS account must have (it already does; recreate them if the account
ever changes):

| Budget | Limit |
|---|---|
| Zero-spend budget | US$1/month → alert as soon as real charges begin |
| Monthly cost budget | US$20/month |

## What would make it expensive (and is therefore excluded)

| Thing | Approximate cost |
|---|---|
| RDS PostgreSQL `db.t4g.micro` + 20 GB | ~US$15/month |
| NAT Gateway (needed for a VPC job to reach the internet) | ~US$32/month + data |
| Public IPv4 on any EC2/RDS | ~US$3.60/month each |
| WAF on CloudFront | ~US$5/month + rules |
