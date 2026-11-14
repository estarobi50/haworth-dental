# haworth-dental

## Overview
End-to-end build of a single-page website for Haworth Dental (Haworth, NJ):
design system, hand-built responsive site, and secure static hosting on AWS in an
account owned by the practice. The site is served from a private S3 bucket through
CloudFront over HTTPS, with DNS in Route 53 and the domain registered at GoDaddy.

Status: the site is a proof of concept / mock-up. The AWS section is the
deployment plan and runbook for the practice-owned account.

## Architecture
User visits https://haworth.com
  → GoDaddy (registrar) nameservers delegate the domain to Route 53
  → Route 53 public hosted zone: Alias A / AAAA records (apex and www) → CloudFront
  → CloudFront distribution "haworth-dental" (HTTPS, ACM certificate)
      → Cache HIT  → served instantly from the edge location
      → Cache MISS → fetched from the origin through Origin Access Control (OAC)
  → Private S3 bucket "haworth-dental" returns index.html, error.html and assets/

## Technologies Used
- HTML / CSS / JS          — hand-built single-page site, no framework or build step
- Cormorant Garamond, Jost — self-hosted fonts (no third-party requests)
- Amazon S3                — private storage for the site files
- Amazon CloudFront        — CDN, HTTPS termination, edge caching, OAC to the bucket
- AWS Certificate Manager (ACM) — free SSL/TLS certificate, auto-renewing (us-east-1)
- Amazon Route 53          — DNS hosted zone and Alias records
- AWS IAM, CloudTrail, Budgets — MFA, named admin access, audit logging, billing alarm
- GoDaddy                  — domain registrar (nameservers delegated to Route 53)

## Repository Layout
  site/               — the deployable website (upload contents to the bucket root)
    index.html, privacy.html, terms.html, error.html, favicon.ico
    fonts/, img/
  aws/bucket-policy.json — S3 bucket policy allowing only the CloudFront distribution
  DESIGN.md           — design system: colors, type, spacing, components, motion

## Design Notes
- Direction: calm, upscale, boutique family dentistry — navy and white, serif
  headlines, hairline detail, one restrained gold accent
- Keeps the practice's existing logo style (blue flower mark + HAWORTH DENTAL wordmark)
- Single page by design; informational only, no patient data is collected
- Reveal-on-scroll motion, fully disabled under prefers-reduced-motion
- See DESIGN.md for tokens and component rules

## Deployment Steps

  1. Create the AWS account (in the practice's name)
       - Go to https://aws.amazon.com → Create an AWS account
       - Use a practice-owned business email and billing method
       - Sign in as root → Security credentials → Assign MFA device
       - Store recovery details with the practice
       - Create named administrative access for the consultant in IAM (no shared
         passwords) so it can be revoked at any time
       - CloudTrail → Create trail for audit logging
       - Billing and Cost Management → Budgets → monthly cost budget with an email alert

  2. Create the S3 bucket
       - Name: haworth-dental
       - Block all public access: ON
       - Static website hosting: OFF (CloudFront reaches the bucket through OAC)
       - Upload the contents of site/ to the bucket root
         (index.html, error.html, assets and folders)

  3. Request the SSL certificate in ACM
       - Region: us-east-1 (required for CloudFront)
       - Domains: haworth.com and www.haworth.com
       - Validation: DNS (use "Create records in Route 53" once the zone exists)

  4. Create the Route 53 hosted zone
       - Route 53 → Hosted zones → Create hosted zone → haworth.com (public)
       - Inventory every existing DNS record at GoDaddy first (website, email/MX,
         SPF/DKIM, verification TXT records)
       - Recreate all of them in the new zone before changing anything

  5. Create the CloudFront distribution
       - Origin: S3 bucket haworth-dental, Origin access: Origin access control
         settings → Create new OAC
       - Viewer protocol policy: Redirect HTTP to HTTPS
       - Alternate domain names: haworth.com, www.haworth.com
       - Custom SSL certificate: the ACM certificate from step 3
       - Default root object: index.html
       - Name the distribution haworth-dental
       - Copy the policy CloudFront offers (or use aws/bucket-policy.json with the
         account and distribution IDs filled in) into S3 → haworth-dental →
         Permissions → Bucket policy
       - Error pages: 403/404 → /error.html

  6. Point DNS at CloudFront
       - Route 53 → haworth.com → Create record
       - Record 1: apex (blank name), type A, Alias ON → CloudFront distribution
       - Record 2: apex, type AAAA, Alias ON → same distribution
       - Records 3 and 4: www, types A and AAAA, Alias ON → same distribution

  7. Cut over the nameservers
       - Route 53 → hosted zone → copy the four NS values
       - GoDaddy → My Products → Domains → haworth.com → DNS → Nameservers →
         Change → Enter my own nameservers → paste the four values
       - Do this only after step 4 is verified, at a low-traffic time
       - Domain stays registered at GoDaddy (lowest risk); transfer to Route 53
         later is optional

  8. Update the site (after any change)
       - S3 → haworth-dental → Upload the changed files (overwrite)
       - CloudFront → haworth-dental → Invalidations → Create invalidation → /*

## Key Learnings
- Keep the bucket private and use Origin Access Control instead of S3 static website
  hosting — the site is reachable only through CloudFront, never directly from S3
  (this is the opposite of the S3 website-endpoint approach in secure-s3-cloudfront-site,
  and the better default for a client site)
- ACM certificates for CloudFront must be created in us-east-1 regardless of where
  anything else lives
- With OAC the default root object only applies to the site root; error pages need
  explicit 403/404 custom error responses because S3 returns 403 for missing keys
- Copy every DNS record (especially MX, SPF, DKIM) into Route 53 before changing
  nameservers, or email breaks the moment the cutover happens
- Delegating nameservers to Route 53 keeps the domain at the registrar (no transfer
  delay or fee) while moving DNS to AWS
- Alias A/AAAA records work at the apex where a CNAME cannot, and are free for AWS targets
- CloudFront does not clear its cache on S3 upload — always run an invalidation (/*)
- Owning the AWS account in the practice's name (with delegated, revocable access for
  the consultant) means no vendor lock-in for the client
- Stock imagery in the mock-up must be replaced with licensed or real practice photos
  before launch
