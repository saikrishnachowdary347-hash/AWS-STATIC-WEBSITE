# Static Website Hosting on AWS — Quick README

This repository contains a tiny static site (index.html) and a short README showing how to host it on AWS using S3 (and optional CloudFront for HTTPS + CDN).

Prerequisites
- An AWS account
- AWS CLI installed and configured (`aws configure`)
- (Optional) Route53 if you want a custom domain

Quick steps (S3 static website)

1. Create an S3 bucket (bucket name must be globally unique)
   - Example:
     aws s3 mb s3://your-bucket-name --region us-east-1

2. Enable static website hosting on the bucket
   - Console: Properties → Static website hosting → Enable
   - CLI:
     aws s3 website s3://your-bucket-name --index-document index.html --error-document error.html

3. Make the objects public (for simple static hosting)
   - Disable Block Public Access for the bucket (console or CLI).
   - Add a bucket policy (replace `your-bucket-name` and region):
     ```json
     {
       "Version":"2012-10-17",
       "Statement":[{
         "Effect":"Allow",
         "Principal": "*",
         "Action":"s3:GetObject",
         "Resource":"arn:aws:s3:::your-bucket-name/*"
       }]
     }
     ```
     aws s3api put-bucket-policy --bucket your-bucket-name --policy file://bucket-policy.json

4. Upload site files
   - From the directory that contains index.html:
     aws s3 sync . s3://your-bucket-name --acl public-read --delete
   - Or to upload a single file:
     aws s3 cp index.html s3://your-bucket-name/index.html --acl public-read

5. Visit the website
   - S3 website endpoint:
     http://your-bucket-name.s3-website-<region>.amazonaws.com
   - If you use CloudFront (recommended for HTTPS), use the CloudFront domain or your custom domain.

Optional: Use CloudFront for HTTPS + CDN (recommended)
- Create a CloudFront distribution with your S3 bucket as the origin.
- Configure origin access (Origin Access Identity / Origin Access Control) to restrict S3 so only CloudFront can access it, then remove public access from the bucket.
- Point your domain (Route53) to the CloudFront distribution (Alias record).
- Invalidate cache when you deploy changes:
  aws cloudfront create-invalidation --distribution-id YOUR_ID --paths "/*"

Common troubleshooting
- 403 Forbidden: Check Block Public Access, bucket policy, and object ACLs. If using CloudFront with OAI/OAC, ensure bucket is not public and CloudFront has access.
- Content-type wrong: Use `aws s3 cp` or `sync` (they generally set the correct `Content-Type`) or set metadata on upload.
- Caching: CloudFront may cache; create invalidation or change filenames to bust cache.

Security note
- For production, prefer CloudFront + OAC/OAI and keep the S3 bucket private. Use HTTPS from CloudFront and provision an ACM certificate for your domain.

That's it — upload the provided index.html and follow the steps above to have a simple emoji-driven static site on AWS.






✨#Skills Learned

AWS S3 · Static Website Hosting · IAM · Cloud Deployment

🙌# Acknowledgment

Special thanks to Rakesh Taninki for helping me complete this project. 🚀 onstrating how to host and deploy a static website using AWS S3 by configuring bucket policies, enabling static website hosting, and making it publicly accessible.
