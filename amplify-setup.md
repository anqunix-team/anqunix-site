# AWS Amplify deployment setup

The GitHub Actions workflow (`.github/workflows/deploy.yml`) builds the site and pushes `dist/` to Amplify
using a manual deployment. Amplify is NOT connected to the repo, so it does not build anything itself.

## 1. Create the Amplify app (one time)
```
aws amplify create-app --name anqunix-site --region <region>
aws amplify create-branch --app-id <appId> --branch-name main
```
Note the `appId`.

## 2. GitHub OIDC role in AWS (one time)
1. IAM > Identity providers > add `token.actions.githubusercontent.com`, audience `sts.amazonaws.com`
   (skip if it already exists in the account).
2. Create a role with this trust policy (replace placeholders):
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::<account-id>:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
      "StringLike": { "token.actions.githubusercontent.com:sub": "repo:anqunix-team/<repo>:environment:production" }
    }
  }]
}
```
3. Attach this permissions policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["amplify:CreateDeployment", "amplify:StartDeployment", "amplify:GetJob"],
    "Resource": "arn:aws:amplify:<region>:<account-id>:apps/<appId>/branches/main/*"
  }]
}
```

## 3. GitHub repo settings
- Settings > Environments > create `production`.
- Settings > Secrets and variables > Actions:
  - Secret `AWS_ROLE_ARN` = the role ARN
  - Variable `AWS_REGION` = e.g. `us-east-1`
  - Variable `AMPLIFY_APP_ID` = the app id

## 4. Custom domain (anqunix.com)
Amplify console > Hosting > Custom domains > add `anqunix.com`. Amplify gives DNS records (a CNAME for www,
an ANAME/CNAME or Route 53 alias for the apex, and an ACM validation CNAME). Add them in Hostinger DNS and
**remove the four GitHub Pages `185.199.x.153` A records** and the old `www` CNAME to `anqunix-team.github.io`.
Keep the MX, SPF, DKIM and TXT records. Hostinger does not support apex CNAMEs, so either use Amplify's
suggested apex forwarding or move DNS to Route 53 for the apex.
