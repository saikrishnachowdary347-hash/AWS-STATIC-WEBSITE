# 🌐 AWS Static Website — Premium Guide

Welcome — this polished README shows you how to host a static website on Amazon S3, secure it, and optionally serve it over HTTPS with CloudFront and a custom domain. Designed to be clear, actionable, and visually appealing with emojis for quick scanning. 🚀

---

✨ Highlights
- Quick CLI + Console steps
- Production best practices (CloudFront + OAC + ACM)
- SPA routing & cache strategies
- GitHub Actions deploy workflow
- Copy-paste policies & IAM least-privilege examples

---

Table of contents
- 🔎 Overview
- 🛠️ Prerequisites
- ⚡ Quick start (CLI)
- 🧭 Console steps
- 🔐 Bucket policy (public read) & CloudFront OAC example
- 📦 Uploading & Cache-Control
- 🔁 SPA fallback / routing
- 🌍 Custom domain + HTTPS (CloudFront + ACM + Route 53)
- ♻️ Cache invalidation
- 🤖 GitHub Actions (automated deploy)
- 🛡️ Security considerations
- 🐞 Troubleshooting
- 🔗 Useful links

---

🔎 Overview
Amazon S3 can serve static assets (HTML/CSS/JS/images). For production, front it with CloudFront for HTTPS, caching, and better security. This guide covers both minimal public S3 hosting and production-ready CloudFront + OAC setups. ✨

---

🛠️ Prerequisites
- AWS account with permissions for S3, CloudFront, ACM, Route 53 (if used)
- AWS CLI installed and configured (`aws configure`)
- Optional: Terraform / CloudFormation for IaC
- Your static site build (index.html + assets)

---

⚡ Quick start (AWS CLI)
Replace example.com with your bucket/domain. Commands are copy-ready.

1) Create bucket (replace region)
```bash
aws s3api create-bucket --bucket example.com --region us-east-1 \
  --create-bucket-configuration LocationConstraint=us-east-1
# Note: omit create-bucket-configuration for us-east-1
```

2) Disable Block Public Access (only for public website; do NOT do this for CloudFront-backed private buckets)
```bash
aws s3api put-public-access-block --bucket example.com --public-access-block-configuration \
  BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false
```

3) Public bucket policy (simple website)
Save as s3-bucket-policy.json and apply:
```json
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Sid":"PublicReadGetObject",
      "Effect":"Allow",
      "Principal":"*",
      "Action":"s3:GetObject",
      "Resource":"arn:aws:s3:::example.com/*"
    }
  ]
}
```
Apply:
```bash
aws s3api put-bucket-policy --bucket example.com --policy file://s3-bucket-policy.json
```

4) Enable static website hosting
```bash
aws s3 website s3://example.com/ --index-document index.html --error-document error.html
```

5) Upload your site
- Long-cached hashed assets, and non-cached index.html:
```bash
aws s3 sync ./public s3://example.com/ --exclude "index.html" --cache-control "max-age=31536000,public"
aws s3 cp ./public/index.html s3://example.com/index.html --cache-control "no-cache, no-store, must-revalidate"
```

6) Test URL
http://example.com.s3-website-<region>.amazonaws.com

---

🧭 Console steps (quick)
1. Create bucket in S3 console.  
2. (Public website) Disable "Block public access" for bucket.  
3. Enable "Static website hosting" → set Index & Error document.  
4. Add Bucket Policy (Permissions → Bucket policy).  
5. Upload files.  
6. Visit website endpoint.

---

🔐 Production: CloudFront + S3 (recommended)
Why CloudFront:
- Custom domain + HTTPS (ACM) ✅
- Edge caching & faster global delivery ✅
- Security: keep S3 private, allow only CloudFront (OAC) ✅

High-level:
1. Create S3 bucket (private). Upload files. Use REST endpoint (not website endpoint).  
2. Create Origin Access Control (OAC) and attach to CloudFront origin.  
3. Grant CloudFront OAC permission in S3 bucket policy (only allow CloudFront).  
4. Request ACM certificate in us-east-1 for CloudFront.  
5. Create CloudFront distribution with your domain names and attach the ACM cert.  
6. Use Route 53 alias A record to point to CloudFront.  
7. For SPA, set custom error responses (404 -> /index.html, 200) or use CloudFront Function to rewrite.  

Example S3 policy for CloudFront OAC (replace OAC/principal as given by AWS console):
```json
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Sid":"AllowCloudFrontServicePrincipal",
      "Effect":"Allow",
      "Principal":{"Service":"cloudfront.amazonaws.com"},
      "Action":"s3:GetObject",
      "Resource":"arn:aws:s3:::example.com/*",
      "Condition":{
         "StringEquals": {"AWS:SourceArn":"arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE"}
      }
    }
  ]
}
```
(When using OAC, the console will show the recommended policy; prefer OAC over OAI.)

---

📦 Uploading & Cache-Control (best practice)
- Immutable, hashed assets (e.g., app.ab12cd.js) → long cache (max-age=31536000, public).  
- index.html → no-cache to ensure clients fetch the newest manifest.  
- Use `aws s3 sync` for convenience; it will usually set Content-Type automatically.

Examples:
```bash
# hashed assets
aws s3 sync ./public s3://example.com/ --exclude "index.html" --cache-control "max-age=31536000,public"

# index
aws s3 cp ./public/index.html s3://example.com/index.html --cache-control "no-cache, no-store, must-revalidate"
```

---

🔁 SPA routing / fallback
Option A — S3 website hosting: set Error document = index.html.  
Option B — CloudFront (recommended): use "Custom Error Responses" mapping 404 → /index.html (response code 200) or rewrite with CloudFront Function / Lambda@Edge.

---

♻️ Cache invalidation
Invalidate CloudFront on deploy (only if needed):
```bash
aws cloudfront create-invalidation --distribution-id E1234567890 --paths "/*"
```
Tip: use hashed filenames to avoid frequent invalidations.

---

🤖 GitHub Actions — deploy example (recommended)
Create .github/workflows/deploy.yml with this snippet:

```yaml
name: Deploy to S3

on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install & build
        run: |
          npm ci
          npm run build # outputs to ./public

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Sync to S3
        env:
          BUCKET: ${{ secrets.S3_BUCKET }}
        run: |
          aws s3 sync ./public s3://$BUCKET --delete --exclude "index.html" --cache-control "max-age=31536000,public"
          aws s3 cp ./public/index.html s3://$BUCKET/index.html --cache-control "no-cache, no-store, must-revalidate"

      - name: Invalidate CloudFront (optional)
        if: ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }}
        run: |
          aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"
```

---

🛡️ Security considerations
- For public website: S3 Block Public Access must be disabled (but this is not ideal for production).  
- For production: keep S3 private and use CloudFront OAC to serve content.  
- Follow least privilege for CI deploy credentials (s3:PutObject, s3:DeleteObject, s3:GetObject, s3:ListBucket restricted to the bucket).  
- Use signed URLs if per-user access control is needed.

IAM deploy policy (least-privilege example)
```json
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Sid":"AllowS3Deploy",
      "Effect":"Allow",
      "Action":[
        "s3:PutObject",
        "s3:PutObjectAcl",
        "s3:DeleteObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource":[
        "arn:aws:s3:::example.com",
        "arn:aws:s3:::example.com/*"
      ]
    }
  ]
}
```

---

🐞 Troubleshooting (common)
- 403 Access Denied:
  - Public S3 website: ensure bucket policy allows public reads and Block Public Access is disabled.
  - CloudFront + OAC: ensure bucket policy allows CloudFront OAC or correct principal.
- SPA 404s: configure S3 error document or CloudFront custom error response (404 → /index.html → 200).
- Wrong Content-Type: upload with correct --content-type or use tools that set it automatically.
- Changes not visible: CloudFront caches — invalidate or use hashed assets.

---

🔗 Useful links
- S3 Static Website Hosting: https://docs.aws.amazon.com/AmazonS3/latest/dev/WebsiteHosting.html  
- CloudFront + S3 best practices: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistS3AndCustomOrigins.html  
- ACM + CloudFront: https://docs.aws.amazon.com/acm/latest/userguide/acm-services.html  
- Origin Access Control: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html

---

🎁 Want help?
I can:
- Create the GitHub Actions workflow for your repo
- Generate Terraform/CloudFormation for S3+CloudFront+ACM
- Tailor the README for your framework (React/Vue/Angular/Hugo)

Tell me which one and I’ll draft it with ready-to-run files. ✅
