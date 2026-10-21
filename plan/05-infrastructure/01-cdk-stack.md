# 05.01 — CDK stack (S3 + CloudFront + deploy role)

**Owner:** the AWS account owner. **When:** milestone 2 (by 2026-11-09).
**Depends on:** `01-foundation/01-prerequisites.md` (tools, an AWS CLI profile for account `436812234154`).
**Unblocks:** `05-infrastructure/02-github-oidc-and-variables.md`, every workflow in `06-ci-cd/`.

One CDK app, one stack, `ProjetoAplicado2`, in account `436812234154` /
`us-east-1`. It creates exactly five things:

```
S3 bucket (private, Block Public Access, OAC only, RETAIN)
   ▲  read + list (Origin Access Control)
CloudFront distribution (HTTPS, HTTP/2+3, PRICE_CLASS_ALL)
   ├── viewer-request CloudFront Function  "spa-rewrite"   (extension-less path → /index.html)
   └── Response Headers Policy             "security headers" (CSP, HSTS, nosniff, DENY, referrer)
IAM role ProjetoAplicado2Deploy  ◄── GitHub OIDC (imported, account-level provider)
```

Infrastructure is deployed **manually** by the AWS account owner (the same
model as the PacketLaboratory project in this account). CI never changes infrastructure; it only runs
`cdk synth` as a check. The deploy role can publish files and invalidate the
cache — nothing more.

---

## Step 1 — Create the folder and `package.json`

From the repository root:

```bash
mkdir -p infra/bin infra/lib
```

`infra/package.json` — exact pins, no `^`:

```json
{
  "name": "projeto-aplicado-2-infra",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "description": "AWS infrastructure: private S3 bucket behind CloudFront, plus the GitHub deploy role.",
  "engines": {
    "node": ">=24.0.0"
  },
  "scripts": {
    "synth": "cdk synth",
    "diff": "cdk diff",
    "deploy": "cdk deploy",
    "typecheck": "tsc --noEmit",
    "lint": "biome check ."
  },
  "dependencies": {
    "aws-cdk-lib": "2.269.0",
    "constructs": "10.8.1"
  },
  "devDependencies": {
    "@biomejs/biome": "2.5.12",
    "@types/node": "24.13.4",
    "aws-cdk": "2.1141.0",
    "typescript": "7.0.2"
  }
}
```

Then generate the lockfile (commit it):

```bash
cd infra && npm install && cd ..
```

`biome check .` picks up the root `biome.json` (Biome looks for the
configuration in parent directories).

## Step 2 — `tsconfig.json`

The stack runs **without a build step**: Node 24 strips the types and
executes the `.ts` files directly. That only works for "erasable" TypeScript
(no `enum`, no `namespace`, no constructor parameter properties) and with
explicit `.ts` import specifiers — both enforced below.

`infra/tsconfig.json`:

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022"],
    "types": ["node"],
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "verbatimModuleSyntax": true,
    "erasableSyntaxOnly": true,
    "isolatedModules": true,
    "allowImportingTsExtensions": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "noEmit": true
  },
  "include": ["bin", "lib"]
}
```

## Step 3 — `cdk.json`

```json
{
  "app": "node --experimental-strip-types bin/app.ts",
  "output": "cdk.out",
  "context": {
    "@aws-cdk/aws-iam:minimizePolicies": true,
    "@aws-cdk/core:checkSecretUsage": true,
    "@aws-cdk/aws-s3:createDefaultLoggingPolicy": true,
    "@aws-cdk/core:target-partitions": ["aws"]
  }
}
```

`cdk.out/` is already in the root `.gitignore`.

## Step 4 — `bin/app.ts` (the account is pinned here)

```ts
#!/usr/bin/env node
import { App, Tags } from "aws-cdk-lib";
import { SiteStack } from "../lib/site-stack.ts";

/**
 * Pinned on purpose. A developer machine often has several AWS profiles. CDK
 * refuses to deploy when the resolved credentials do not belong to the pinned
 * account, which turns a wrong profile into an error message instead of
 * resources in some other account.
 */
const ACCOUNT = "436812234154";

/** CloudFront certificates (for a future custom domain) must live here. */
const REGION = "us-east-1";

const app = new App();

new SiteStack(app, "ProjetoAplicado2", {
  env: { account: ACCOUNT, region: REGION },
  description: "Projeto Aplicado II (IFC): non-partisan candidate panel for Videira/SC. S3 + CloudFront, no servers.",
  repository: "Miguel060/projetoAplicado2ifc",
  branch: "main",
  // From `gh api users/Miguel060 --jq .id` and `gh api repos/Miguel060/projetoAplicado2ifc --jq .id`.
  // Needed for the immutable-identifier form of the OIDC subject (see githubSubjects).
  ownerId: "77062840",
  repositoryId: "1366189075",
  // Account-level provider created by the PacketLaboratory stack. IMPORTED, never created
  // here: an account can hold only one provider per URL. See 05-infrastructure/02.
  githubOidcProviderArn: `arn:aws:iam::${ACCOUNT}:oidc-provider/token.actions.githubusercontent.com`,
});

// Lets Cost Explorer separate this project from PacketLaboratory once the tag is
// activated in Billing → Cost allocation tags.
Tags.of(app).add("project", "projetoAplicado2ifc");
```

## Step 5 — `lib/spa-rewrite.js` (CloudFront Function, runtime `cloudfront-js-2.0`)

Why it exists: the app has real paths (`/candidatos`, `/candidatos/fulano-1234`, `/sobre`)
that are not files in S3. The usual "map 403/404 to `/index.html`" trick is
distribution-wide, so a missing **data** file would come back as `200` +
HTML and the app would crash on `JSON.parse`. This function rewrites **only
paths whose last segment has no dot**; anything that looks like a file
(`.js`, `.css`, `.json`, `.jpg`, `.sqlite`, `.txt`, `.svg`) passes through
untouched and keeps its real status code.

```js
// CloudFront Functions, runtime cloudfront-js-2.0. Runs on every viewer request,
// before the cache lookup, so every SPA route shares the cached /index.html.
//
//   /                               -> /index.html   (landing page)
//   /candidatos                     -> /index.html   (search)
//   /candidatos/fulano-de-tal-1234  -> /index.html   (profile)
//   /sobre                          -> /index.html   (about)
//   /assets/index-3f2a.js          -> unchanged (file)
//   /data/latest.json              -> unchanged (file; a missing file stays a 404)
//
// Contract with the ETL: candidate slugs never contain a dot (they match ^[a-z0-9-]+$).
//
// biome-ignore lint/correctness/noUnusedVariables: CloudFront invokes `handler` by name.
function handler(event) {
  const request = event.request;
  const uri = request.uri;
  const lastSegment = uri.slice(uri.lastIndexOf("/") + 1);

  if (lastSegment.indexOf(".") === -1) {
    request.uri = "/index.html";
  }

  return request;
}
```

The query string is a separate field (`request.querystring`) and is not
touched, so `/candidatos?q=maria&cargo=federal` keeps its filters.

## Step 6 — `lib/site-stack.ts`

```ts
import { join } from "node:path";
import {
  CfnOutput,
  Duration,
  RemovalPolicy,
  Stack,
  type StackProps,
  aws_cloudfront as cloudfront,
  aws_cloudfront_origins as origins,
  aws_iam as iam,
  aws_s3 as s3,
} from "aws-cdk-lib";
import type { Construct } from "constructs";

export interface SiteStackProps extends StackProps {
  /** `owner/repo` allowed to assume the deploy role. */
  readonly repository: string;
  /** Only runs on this branch may deploy (push, schedule and manual dispatch from it). */
  readonly branch: string;
  /** Numeric ids GitHub embeds in the immutable-identifier form of the OIDC subject. */
  readonly ownerId: string;
  readonly repositoryId: string;
  /** ARN of the account-level GitHub OIDC provider, which this stack imports. */
  readonly githubOidcProviderArn: string;
}

/**
 * Everything the browser may load comes from our own origin. The app has no
 * inline scripts (Vite emits external module files), no third-party fonts and
 * no analytics. 'unsafe-inline' for styles only: Radix injects <style>
 * elements at runtime, which is low risk with script-src locked to 'self'.
 */
const CONTENT_SECURITY_POLICY = [
  "default-src 'self'",
  "script-src 'self'",
  "style-src 'self' 'unsafe-inline'",
  "img-src 'self' data:",
  "font-src 'self'",
  "connect-src 'self'",
  "object-src 'none'",
  "base-uri 'self'",
  "form-action 'self'",
  "frame-ancestors 'none'",
  "upgrade-insecure-requests",
].join("; ");

/**
 * GitHub emits the OIDC `sub` claim in one of two shapes, depending on a
 * setting that is easy to miss:
 *
 *   repo:Miguel060/projetoAplicado2ifc:ref:refs/heads/main
 *   repo:Miguel060@77062840/projetoAplicado2ifc@1366189075:ref:refs/heads/main
 *
 * The second ("immutable identifiers") carries numeric ids so that renaming an
 * account or repository cannot point this trust policy at somebody else's
 * repository. Both are accepted, exactly — no wildcard: `Miguel060*` would
 * also match an attacker's `Miguel060-evil`.
 */
function githubSubjects(props: SiteStackProps): string[] {
  const ref = `ref:refs/heads/${props.branch}`;
  const [owner, name] = props.repository.split("/");
  return [
    `repo:${props.repository}:${ref}`,
    `repo:${owner}@${props.ownerId}/${name}@${props.repositoryId}:${ref}`,
  ];
}

export class SiteStack extends Stack {
  constructor(scope: Construct, id: string, props: SiteStackProps) {
    super(scope, id, props);

    /*
     * Never public. CloudFront reaches it through Origin Access Control, so the
     * only way to an object is through the distribution and its headers.
     * RETAIN: `cdk destroy` must not take the published data with it.
     */
    const bucket = new s3.Bucket(this, "SiteBucket", {
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      encryption: s3.BucketEncryption.S3_MANAGED,
      enforceSSL: true,
      objectOwnership: s3.ObjectOwnership.BUCKET_OWNER_ENFORCED,
      removalPolicy: RemovalPolicy.RETAIN,
    });

    const spaRewrite = new cloudfront.Function(this, "SpaRewrite", {
      runtime: cloudfront.FunctionRuntime.JS_2_0,
      code: cloudfront.FunctionCode.fromFile({
        filePath: join(import.meta.dirname, "spa-rewrite.js"),
      }),
      comment: "Serve /index.html for extension-less paths (SPA routes)",
    });

    const securityHeaders = new cloudfront.ResponseHeadersPolicy(this, "SecurityHeaders", {
      comment: "Projeto Aplicado II security headers",
      securityHeadersBehavior: {
        contentSecurityPolicy: { contentSecurityPolicy: CONTENT_SECURITY_POLICY, override: true },
        strictTransportSecurity: {
          accessControlMaxAge: Duration.days(730),
          includeSubdomains: true,
          override: true,
        },
        contentTypeOptions: { override: true },
        frameOptions: { frameOption: cloudfront.HeadersFrameOption.DENY, override: true },
        referrerPolicy: {
          referrerPolicy: cloudfront.HeadersReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN,
          override: true,
        },
      },
      customHeadersBehavior: {
        customHeaders: [
          {
            header: "Permissions-Policy",
            value: "camera=(), microphone=(), geolocation=(), payment=(), usb=()",
            override: true,
          },
        ],
      },
    });

    const distribution = new cloudfront.Distribution(this, "SiteDistribution", {
      comment: "Projeto Aplicado II - painel de candidatos (Videira/SC)",
      defaultRootObject: "index.html",
      defaultBehavior: {
        /*
         * READ + LIST: with s3:ListBucket, S3 answers a missing key with 404
         * instead of 403, so "file not found" is reported honestly. Listing is
         * not reachable from outside: the cache policy forwards no query
         * strings, and "/" is rewritten to /index.html by the function.
         */
        origin: origins.S3BucketOrigin.withOriginAccessControl(bucket, {
          originAccessLevels: [cloudfront.AccessLevel.READ, cloudfront.AccessLevel.LIST],
        }),
        viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
        allowedMethods: cloudfront.AllowedMethods.ALLOW_GET_HEAD,
        // Honours the Cache-Control set at upload time (immutable / 60 s / no-cache).
        cachePolicy: cloudfront.CachePolicy.CACHING_OPTIMIZED,
        responseHeadersPolicy: securityHeaders,
        compress: true,
        functionAssociations: [
          { function: spaRewrite, eventType: cloudfront.FunctionEventType.VIEWER_REQUEST },
        ],
      },
      // All edge locations, including South America: the audience is in Videira/SC.
      priceClass: cloudfront.PriceClass.PRICE_CLASS_ALL,
      httpVersion: cloudfront.HttpVersion.HTTP2_AND_3,
      enableIpv6: true,
    });

    /*
     * The provider is account-wide and already exists (created by the
     * PacketLaboratory stack). Importing it adds no resource and changes
     * nothing in that stack.
     */
    const githubOidc = iam.OpenIdConnectProvider.fromOpenIdConnectProviderArn(
      this,
      "GitHubOidc",
      props.githubOidcProviderArn,
    );

    const deployRole = new iam.Role(this, "DeployRole", {
      roleName: "ProjetoAplicado2Deploy",
      description: "Assumed by GitHub Actions (main branch only) to publish the site and data snapshots.",
      maxSessionDuration: Duration.hours(1),
      assumedBy: new iam.WebIdentityPrincipal(githubOidc.openIdConnectProviderArn, {
        StringEquals: {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": githubSubjects(props),
        },
      }),
    });

    // Exactly what publishing needs. Written out instead of bucket.grantReadWrite(),
    // which would also grant bucket-level reads, multipart aborts and more.
    deployRole.addToPolicy(
      new iam.PolicyStatement({
        sid: "ListTheBucket",
        actions: ["s3:ListBucket"],
        resources: [bucket.bucketArn],
      }),
    );
    deployRole.addToPolicy(
      new iam.PolicyStatement({
        sid: "ReadWriteObjects",
        actions: ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
        resources: [bucket.arnForObjects("*")],
      }),
    );
    deployRole.addToPolicy(
      new iam.PolicyStatement({
        sid: "InvalidateTheDistribution",
        actions: ["cloudfront:CreateInvalidation", "cloudfront:GetInvalidation"],
        resources: [`arn:aws:cloudfront::${this.account}:distribution/${distribution.distributionId}`],
      }),
    );

    new CfnOutput(this, "BucketName", { value: bucket.bucketName });
    new CfnOutput(this, "DistributionId", { value: distribution.distributionId });
    new CfnOutput(this, "SiteUrl", { value: `https://${distribution.distributionDomainName}` });
    new CfnOutput(this, "DeployRoleArn", { value: deployRole.roleArn });
  }
}
```

Design notes that are easy to undo by accident:

| Choice | Why it must stay |
|---|---|
| No `errorResponses` on the distribution | They would turn missing data files into `200 text/html` (see Step 5). |
| `AccessLevel.LIST` on the origin | Without it, missing objects are `403`, indistinguishable from a permissions error. |
| `CACHING_OPTIMIZED` | It forwards no query strings and honours the upload-time `Cache-Control`. The immutable/60 s/no-cache scheme in `01-foundation/04-json-contract.md` depends on it. |
| No bucket lifecycle rule | Old data versions are pruned by `refresh-data.yml` (keep 5). A lifecycle rule would delete the live version if the weekly cron ever stops. |
| `RETAIN` on the bucket | `cdk destroy` keeps the content. Delete the bucket by hand if the project is ever retired. |
| Imported OIDC provider | Creating a second provider for the same URL fails (`EntityAlreadyExists`). |

## Step 7 — Check it locally (no credentials needed)

```bash
cd infra
npm run lint
npm run typecheck
npx cdk synth --quiet          # writes cdk.out/, prints nothing on success
```

Look at the synthesised trust policy once; it must list exactly the two
subjects and no wildcard:

```bash
grep -A6 '"token.actions.githubusercontent.com:sub"' cdk.out/ProjetoAplicado2.template.json
```

## Step 8 — Deploy (AWS account owner, manually)

The account is already bootstrapped (`CDKToolkit` in `us-east-1`, verified
2026-10-21) — **do not run `cdk bootstrap`**.

> Always pass `--profile <your-profile>` explicitly; the project uses a
> profile named `personal`, and the commands below are written with it. Never
> rely on a default profile or a global `AWS_PROFILE`.

From the repository root:

```bash
# 0. Preflight: must print exactly 436812234154. Anything else: STOP.
aws sts get-caller-identity --profile personal --query Account --output text

# 1. See what will be created (first time: everything).
cd infra
npx cdk diff --profile personal

# 2. Deploy. CDK shows the IAM changes and asks for confirmation: read the trust policy.
npx cdk deploy --profile personal
```

Equivalent from the repo root: `make infra-diff` and `make infra-deploy`
(both run the preflight first — see `01-foundation/02-repository-layout.md`).

Deploy with a **least-privilege IAM user** (never root access keys; MFA on
the root user). That user needs `sts:AssumeRole` on the CDK bootstrap roles
(`cdk-hnb659fds-*`). If CDK prints that it could not assume
`cdk-hnb659fds-deploy-role-…` and is "proceeding anyway" with the current
credentials, that permission is missing — fix the IAM user rather than
ignoring the warning.

The first deploy takes 5–10 minutes (CloudFront propagation).

## Step 9 — Record the outputs

`cdk deploy` prints four outputs. Copy them here and send them to the
repository admin for the repository variables (`05-infrastructure/02-github-oidc-and-variables.md`):

| Output | Repository variable | Value |
|---|---|---|
| `ProjetoAplicado2.BucketName` | `SITE_BUCKET` | _fill in after deploy_ |
| `ProjetoAplicado2.DistributionId` | `SITE_DISTRIBUTION_ID` | _fill in after deploy_ |
| `ProjetoAplicado2.SiteUrl` | `SITE_URL` (no trailing slash) | _fill in after deploy_ |
| `ProjetoAplicado2.DeployRoleArn` | `AWS_DEPLOY_ROLE_ARN` | `arn:aws:iam::436812234154:role/ProjetoAplicado2Deploy` |

To print them again later:

```bash
aws cloudformation describe-stacks --stack-name ProjetoAplicado2 --profile personal \
  --query 'Stacks[0].Outputs[].[OutputKey,OutputValue]' --output table
```

## Step 10 — Verify the infrastructure (read-only)

Before the first site deploy the bucket is empty, so `/` resolves to a missing
`/index.html`:

```bash
SITE_URL=https://dxxxxxxxxxxxx.cloudfront.net   # from Step 9

curl -sI "$SITE_URL/" | sed -n '1p'                       # HTTP/2 404 (empty bucket, LIST granted)
curl -sI "$SITE_URL/" | grep -iE 'content-security-policy|strict-transport-security|x-frame-options|x-content-type-options|referrer-policy|permissions-policy'
                                                          # all six headers present

# The bucket itself is not reachable directly:
BUCKET=<SITE_BUCKET>
curl -s -o /dev/null -w '%{http_code}\n' "https://$BUCKET.s3.amazonaws.com/index.html"   # 403
aws s3api get-public-access-block --bucket "$BUCKET" --profile personal                  # all four true
```

Optional — exercise the function without deploying anything:

```bash
FN=$(aws cloudfront list-functions --profile personal \
  --query "FunctionList.Items[?contains(Name, 'SpaRewrite')].Name | [0]" --output text)
ETAG=$(aws cloudfront describe-function --name "$FN" --stage LIVE --profile personal --query ETag --output text)
printf '%s' '{"version":"1.0","context":{"eventType":"viewer-request"},"viewer":{"ip":"1.2.3.4"},"request":{"method":"GET","uri":"/candidatos/fulano-1234","querystring":{},"headers":{},"cookies":{}}}' > cf-event.json
aws cloudfront test-function --name "$FN" --if-match "$ETAG" --stage LIVE \
  --event-object fileb://cf-event.json --profile personal \
  --query 'TestResult.FunctionOutput' --output text      # contains "uri":"/index.html"
```

## Changing the stack later

1. Edit `infra/lib/site-stack.ts` in a branch; CI runs `cdk synth`.
2. After merge, the AWS account owner runs `make infra-diff` and `make infra-deploy`.
3. If an output changed (it should not — names are stable), update the
   repository variables.

## Phase 2: custom domain (additive)

1. Request an ACM certificate for the domain in `us-east-1` (DNS validation).
2. Add `domainNames` and `certificate` to the distribution and
   `minimumProtocolVersion: TLS_V1_2_2021`.
3. Point the domain's DNS at the distribution (CNAME, or an alias record if
   the zone is in Route 53).
4. Update `SITE_URL`. Nothing already deployed is rebuilt.
