# AWS-STATIC-WEBSITE
# Host a Static Website on Amazon S3 — Full Guide

This README explains how to host a static website on Amazon S3, secure it, and (optionally) serve it over HTTPS with CloudFront and a custom domain. It includes step-by-step console and AWS CLI instructions, sample bucket policy JSON, SPA fallback setup, caching, invalidation, and a sample GitHub Actions workflow to automate deployment.

Table of contents
- Overview
- Prerequisites
- Quick start (CLI)
- Console steps (if you prefer the AWS Console)
- Bucket policy example (public read)
- Uploading content and cache-control
- Single Page App (SPA) fallback / routing
- Custom domain + HTTPS using CloudFront + ACM + Route 53 (recommended for production)
- Cache invalidation
- Automating with GitHub Actions
- Security considerations
- Troubleshooting / common errors
- Useful references

---

Overview
--------
Amazon S3 can serve static websites (HTML, CSS, JS, images). For production with a custom domain and HTTPS, use CloudFront with an ACM certificate. This guide covers both minimal setups (S3 website endpoint) and production-ready setups (CloudFront + OAC + ACM + Route 53).

🧱🔥😵Prerequisites
-------------
- AWS account with permissions to manage S3, CloudFront, ACM, and Route 53 (if using custom domain).
- AWS CLI installed and configured (aws configure).
- Optional: Terraform/CloudFormation if you prefer IaC.
- Your static website files (index.html, assets, etc.).

Quick start (AWS CLI)
---------------------
Replace `example.com` with your bucket name (recommended to match your domain for custom domains).

1. Create the bucket
   - If using a website endpoint and custom domain matching the bucket name:
     - Bucket name must equal your domain name (example.com or www.example.com).
   - Create bucket:
     ```bash
     aws s3api create-bucket --bucket example.com --region us-east-1 \
       --create-bucket-configuration LocationConstraint=us-east-1
     ```
     Note: In `us-east-1` you may omit the create-bucket-configuration parameter.

2. Disable "Block Public Access" for the bucket
   - S3 console: Bucket > Permissions > Block public access > uncheck / disable
   - Or CLI:
     ```bash
     aws s3api put-public-access-block --bucket example.com --public-access-block-configuration \
       BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false
     ```

3. Attach a public read bucket policy (see policy example below)
   ```bash
   aws s3api put-bucket-policy --bucket example.com --policy file://s3-bucket-policy.json
   ```

4. Enable static website hosting
   ```bash
   aws s3 website s3://example.com/ --index-document index.html --error-document error.html
   ```

5. Upload your site
   - Basic sync (S3 will auto-detect content-type):
     ```bash
     aws s3 sync ./public s3://example.com/ --delete
     ```
   - Example with Cache-Control for assets (long lived caching for hashed assets):
     ```bash
     aws s3 sync ./public s3://example.com/ --exclude "index.html" --cache-control "max-age=31536000,public"
     aws s3 cp ./public/index.html s3://example.com/index.html --cache-control "no-cache, no-store, must-revalidate"
     ```
     Adjust as needed. Using hashed filenames + long cache lifetime is recommended.

6. Test
   - S3 website endpoint:
     ```
     http://example.com.s3-website-<region>.amazonaws.com
     ```
   - Or via your domain if DNS already points to S3 website endpoint.

Console steps (summary)
-----------------------
1. Create bucket (S3 console).
2. Turn off "Block public access" for the bucket to allow public policies.
3. Enable "Static website hosting" under Properties -> Static website hosting. Enter Index document and Error document.
4. Add the bucket policy (Permissions -> Bucket policy).
5. Upload your files (Upload or use AWS CLI).
6. Visit the website endpoint.

Bucket policy example (public read)
-----------------------------------
Save this as `s3-bucket-policy.json` and apply it. This policy allows public read access to objects only (recommended over giving public bucket ACLs).

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::example.com/*"
    }
  ]
}
```

Notes:
- Replace `example.com` with your bucket name.
- Ensure S3 Block Public Access is disabled; otherwise this policy will be blocked.
- Granting public access to objects is required for simple S3 website hosting. For production with CloudFront, prefer restricting S3 to only allow CloudFront (see CloudFront section).

Uploading content and cache-control
-----------------------------------
- For SPA index.html use `no-cache` so you can push updates without stale pages:
  ```bash
  aws s3 cp ./public/index.html s3://example.com/index.html --cache-control "no-cache, no-store, must-revalidate"
  ```
- For static hashed assets (js/ css with content hash), use long cache:
  ```bash
  aws s3 sync ./public s3://example.com/ --exclude "index.html" --cache-control "max-age=31536000,public"
  ```
- To set content-type correctly, `aws s3 sync` usually detects it automatically. If needed, use `--content-type` on upload.

Single Page App (SPA) fallback / routing
---------------------------------------
SPAs need the index document returned for client routes. Two approaches:

1. If using S3 website hosting endpoint (not CloudFront): set the Error document to `index.html` to serve index for unknown paths.
   - In console: Static website hosting -> Error document = index.html
2. If using CloudFront + S3 REST API origin secured via Origin Access Control (OAC): create a Lambda@Edge or CloudFront Function to rewrite 404s to index.html, or configure custom error response in CloudFront (e.g., HTTP 404 => respond with /index.html with 200).

Custom domain + HTTPS (recommended): CloudFront + ACM + Route 53
----------------------------------------------------------------
Why CloudFront:
- HTTPS on custom domain (S3 website endpoint can't use ACM certs directly)
- Better caching, performance, and security
- Edge functions (CloudFront Functions or Lambda@Edge) for SPA routing

High-level steps
1. Create S3 bucket (the bucket does NOT need to be public). Upload site files. Use the bucket as origin but use the REST endpoint (not the website endpoint) so CloudFront can retrieve private objects securely.
2. Create an Origin Access Control (OAC) or Origin Access Identity (OAI) to restrict direct public access. Grant CloudFront permission to get objects from the bucket.
   - Newer recommended approach: Origin Access Control (OAC).
3. Create an ACM certificate in us-east-1 (N. Virginia) for CloudFront if you’ll use the CloudFront distribution with a global cert.
4. Create a CloudFront distribution:
   - Origin: S3 bucket (REST endpoint)
   - Restrict bucket access via OAC and update bucket policy to allow only CloudFront's Origin Access.
   - Behavior: Default root object = index.html (optional)
   - For SPA: configure custom error responses (404 => /index.html, response code 200)
   - Attach ACM certificate and set Alternate Domain Names (CNAMEs) to your domain (e.g., example.com, www.example.com).
5. Set DNS (Route 53)
   - Create an A (alias) record pointing your domain to the CloudFront distribution domain.
6. Invalidate CloudFront after deploys when necessary (see invalidation section).

Example bucket policy to allow only CloudFront (OAI/OAC)
- When using OAI or OAC you will add a policy that grants s3:GetObject to the CloudFront principal or your OAC principal. The exact policy depends on which method you choose — the console will provide the statement during CloudFront setup.

ACM certificate
--------------
- Request certificate in us-east-1 (N. Virginia) for CloudFront.
- Validate by DNS (recommended) or email.

Cache invalidation
------------------
- To serve updated files immediately via CloudFront, submit an invalidation:
  ```bash
  aws cloudfront create-invalidation --distribution-id E1234567890 --paths "/*"
  ```
- Avoid frequent full invalidations to reduce cost. Use immutable (hashed) asset filenames to avoid invalidation for assets.

Automating deployments with GitHub Actions
------------------------------------------
A simple workflow to build and sync to S3 using AWS CLI. Configure GitHub Secrets: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION, S3_BUCKET.

Example `.github/workflows/deploy.yml` snippet:

```yaml
name: Deploy to S3

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install and build
        run: |
          npm ci
          npm run build # outputs to ./public or ./build

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Sync to S3
        run: |
          aws s3 sync ./public s3://${{ secrets.S3_BUCKET }} --delete \
            --exclude "index.html" --cache-control "max-age=31536000,public"
          aws s3 cp ./public/index.html s3://${{ secrets.S3_BUCKET }}/index.html --cache-control "no-cache, no-store, must-revalidate"
```

If you use CloudFront, add an invalidation step after the sync:
```yaml
      - name: Invalidate CloudFront
        env:
          DISTRIBUTION_ID: ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }}
        run: |
          aws cloudfront create-invalidation --distribution-id $DISTRIBUTION_ID --paths "/*"
```

Security considerations
-----------------------
- For simple public websites using S3 website hosting: keep an eye on S3 Block Public Access (must be disabled for public website).
- For production: prefer CloudFront + OAC and keep S3 objects private; only allow CloudFront to access the bucket.
- Use least privilege for deploy IAM credentials. Create an IAM user with limited S3 permissions (s3:PutObject, s3:DeleteObject, s3:GetObject) limited to the bucket.
- Use signed URLs if content needs to be private per user.

Troubleshooting / common errors
-------------------------------
- 403 Access Denied
  - If using direct S3 website hosting: ensure bucket policy allows public read AND Block Public Access is disabled.
  - If using CloudFront with OAC/OAI: ensure bucket policy allows CloudFront origin access identity or access via OAC.
- 404 Not Found for SPA routes
  - If using S3 website hosting endpoint, set Error document to index.html.
  - If using CloudFront, configure "Custom Error Responses" mapping 404 => /index.html with HTTP response code 200 or use a function to rewrite requests.
- Wrong Content-Type
  - Ensure uploads set correct content-type. `aws s3 sync` typically sets it automatically. If it doesn't, set manually via `aws s3 cp --content-type`.
- DNS still pointing to old site
  - TTL may be cached. Use Route 53 alias records to CloudFront to avoid CNAME issues.
- Changes not showing
  - CloudFront caches; create invalidation or rely on cache-control and file hashing.

Sample IAM deploy policy (least-privilege for syncing a single bucket)
---------------------------------------------------------------------
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3Deploy",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl",
        "s3:DeleteObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::example.com",
        "arn:aws:s3:::example.com/*"
      ]
    }
  ]
}
```

Useful notes & best practices
----------------------------
- Use hashed asset filenames (e.g., app.f3a1b2.js) with long cache times to minimize invalidations.
- Use `index.html` with `no-cache` to ensure clients get the latest HTML that references hashed assets.
- Prefer CloudFront OAC for new distributions (OAI is older).
- Request ACM certificates in us-east-1 for CloudFront.
- Use Route 53 alias records to CloudFront for cheaper DNS and automatic root domain support.

Useful links
------------
- S3 Static Website Hosting: https://docs.aws.amazon.com/AmazonS3/latest/dev/WebsiteHosting.html
- CloudFront + S3 origin best practices: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistS3AndCustomOrigins.html
- ACM + CloudFront: https://docs.aws.amazon.com/acm/latest/userguide/acm-services.html
- Origin Access Control: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html

---

If you'd like, I can:
- Create a ready-to-use GitHub Actions workflow file for your repository with secrets to set,
- Generate CloudFormation / Terraform templates to provision the S3 + CloudFront + ACM stack,
- Or tailor the README to your specific framework (React, Vue, Angular, Hugo) including exact build steps.

What I did: I prepared this complete README covering both simple and production-ready deployment patterns. Next: tell me whether you want the GitHub Actions file, Terraform/CloudFormation, or a customized README for a specific framework and domain name, and I will generate it for you.
