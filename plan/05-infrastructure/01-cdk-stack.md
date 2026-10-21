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
the assertion tests (Step 6b) and `cdk synth` as checks — a stack change that
breaks a security invariant fails CI before anyone can deploy it. The deploy role can publish files and invalidate the
cache — nothing more.

---

## Step 1 — Create the folder and `package.json`

From the repository root:

```bash
mkdir -p infra/bin infra/lib infra/test
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
    "lint": "biome check .",
    "test": "vitest run --coverage"
  },
  "dependencies": {
    "aws-cdk-lib": "2.269.0",
    "constructs": "10.8.1"
  },
  "devDependencies": {
    "@biomejs/biome": "2.5.12",
    "@types/node": "24.13.4",
    "@vitest/coverage-v8": "5.0.5",
    "aws-cdk": "2.1141.0",
    "typescript": "7.0.2",
    "vitest": "5.0.5"
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
  "include": ["bin", "lib", "test", "vitest.config.ts"]
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

## Step 4 — `lib/config.ts` and `bin/app.ts` (the account is pinned here)

The stack's configuration lives in `lib/config.ts`, separate from
`bin/app.ts`, so the assertion tests (Step 6b) build the stack from **the very
same values** that get deployed. `bin/app.ts` cannot be imported by a test: it
creates the CDK `App` as a side effect.

`infra/lib/config.ts`:

```ts
import type { SiteStackProps } from "./site-stack.ts";

/**
 * Pinned on purpose. A developer machine often has several AWS profiles. CDK
 * refuses to deploy when the resolved credentials do not belong to the pinned
 * account, which turns a wrong profile into an error message instead of
 * resources in some other account.
 */
export const ACCOUNT = "436812234154";

/** CloudFront certificates (for a future custom domain) must live here. */
export const REGION = "us-east-1";

export const STACK_NAME = "ProjetoAplicado2";

export const siteStackProps: SiteStackProps = {
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
};
```

`infra/bin/app.ts`:

```ts
#!/usr/bin/env node
import { App, Tags } from "aws-cdk-lib";
import { STACK_NAME, siteStackProps } from "../lib/config.ts";
import { SiteStack } from "../lib/site-stack.ts";

const app = new App();

new SiteStack(app, STACK_NAME, siteStackProps);

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
import { fileURLToPath } from "node:url";
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
        filePath: fileURLToPath(new URL("spa-rewrite.js", import.meta.url)),
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

## Step 6b — Assertion tests (`infra/test/`)

Decision D15 and `01-foundation/05-testing-strategy.md` § Infrastructure:
the security invariants of this stack are **pinned by tests**. They run in the
CI job `Infrastructure (CDK)` (and therefore before every site deploy, which
reuses CI). Coverage is **reported, not gated**: the code is declarative and
the assertions are what protect it.

| Invariant | Test |
|---|---|
| Bucket: Block Public Access fully on, SSL enforced, `RETAIN`, no lifecycle rule, no website hosting | `site-stack.test.ts` → "bucket" |
| Bucket policy allows reads only to this distribution | `site-stack.test.ts` → "bucket" |
| Distribution: Origin Access Control (no legacy OAI), redirect to HTTPS, optimized cache policy, no error-page fallback, all edge locations, HTTP/2+3 | `site-stack.test.ts` → "distribution" |
| Viewer-request function attached, and its deployed code **is** `lib/spa-rewrite.js` | `site-stack.test.ts` → "distribution" |
| Response headers policy with the **exact** CSP string and the other security headers | `site-stack.test.ts` → "response headers" |
| Deploy role trust: `aud = sts.amazonaws.com` and **exactly** the two subjects, no wildcard, no extra condition | `site-stack.test.ts` → "deploy role" |
| Deploy role permissions: **exactly** the six documented actions, on this bucket / this distribution only | `site-stack.test.ts` → "deploy role" |
| OIDC provider imported: no provider resource (and no Lambda) in the template | `site-stack.test.ts` → "stack placement" |
| Stack pinned to `436812234154` / `us-east-1` | `site-stack.test.ts` → "stack placement" |
| CloudFront Function rewrite table | `spa-rewrite.test.ts` |

Expected values (account, subjects, CSP, actions) are written out
**literally** in the tests. Importing them from the code under test would make
every assertion pass by construction.

### `infra/vitest.config.ts`

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    include: ["test/**/*.test.ts"],
    environment: "node",
    // Synthesising the stack takes a few seconds on a cold CI runner.
    testTimeout: 30_000,
    coverage: {
      provider: "v8",
      // bin/app.ts is a side-effect entry point and is never imported by tests.
      include: ["lib/**/*.ts"],
      reporter: ["text", "text-summary"],
      // No thresholds: reported only (decision D15). lib/spa-rewrite.js runs
      // inside node:vm in its test, which v8 coverage does not instrument.
    },
  },
});
```

### `infra/test/site-stack.test.ts`

```ts
import { readFileSync } from "node:fs";
import { App } from "aws-cdk-lib";
import { Match, Template } from "aws-cdk-lib/assertions";
import { beforeAll, describe, expect, it } from "vitest";
import { STACK_NAME, siteStackProps } from "../lib/config.ts";
import { SiteStack } from "../lib/site-stack.ts";

// ---- Expected values, written out literally on purpose -------------------------------

const ACCOUNT = "436812234154";
const REGION = "us-east-1";
const OIDC_PROVIDER_ARN = `arn:aws:iam::${ACCOUNT}:oidc-provider/token.actions.githubusercontent.com`;
const SUBJECTS = [
  "repo:Miguel060/projetoAplicado2ifc:ref:refs/heads/main",
  "repo:Miguel060@77062840/projetoAplicado2ifc@1366189075:ref:refs/heads/main",
];
const CSP =
  "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; " +
  "font-src 'self'; connect-src 'self'; object-src 'none'; base-uri 'self'; form-action 'self'; " +
  "frame-ancestors 'none'; upgrade-insecure-requests";
/** AWS managed cache policy "CachingOptimized". */
const CACHING_OPTIMIZED_ID = "658327ea-f89d-4fab-a63d-7e88639e58f6";
const DEPLOY_ACTIONS = [
  "s3:ListBucket",
  "s3:GetObject",
  "s3:PutObject",
  "s3:DeleteObject",
  "cloudfront:CreateInvalidation",
  "cloudfront:GetInvalidation",
];

// ---- Helpers ----------------------------------------------------------------------------

type CfnResource = ReturnType<Template["findResources"]>[string];

interface PolicyStatement {
  Effect: string;
  Action: string | string[];
  Resource?: unknown;
  Principal?: unknown;
  Condition?: unknown;
}

/** The feature flags the CDK CLI applies from cdk.json, so the tested template equals the deployed one. */
function cdkJsonContext(): Record<string, unknown> {
  const raw = readFileSync(new URL("../cdk.json", import.meta.url), "utf8");
  return (JSON.parse(raw) as { context: Record<string, unknown> }).context;
}

let stack: SiteStack;
let template: Template;

function onlyResource(type: string): { key: string; resource: CfnResource } {
  const entries = Object.entries(template.findResources(type));
  expect(entries, `exactly one ${type}`).toHaveLength(1);
  const [entry] = entries;
  if (entry === undefined) throw new Error(`no ${type} in the template`);
  return { key: entry[0], resource: entry[1] };
}

function statementsOf(resource: CfnResource): PolicyStatement[] {
  return resource["Properties"]["PolicyDocument"]["Statement"] as PolicyStatement[];
}

beforeAll(() => {
  const app = new App({ context: cdkJsonContext() });
  stack = new SiteStack(app, STACK_NAME, siteStackProps);
  template = Template.fromStack(stack);
});

// ---- Tests ------------------------------------------------------------------------------

describe("stack placement", () => {
  it("is pinned to the project account and region", () => {
    expect(siteStackProps.env).toEqual({ account: ACCOUNT, region: REGION });
    expect(stack.account).toBe(ACCOUNT);
    expect(stack.region).toBe(REGION);
    expect(STACK_NAME).toBe("ProjetoAplicado2");
  });

  it("imports the GitHub OIDC provider instead of creating one, and creates no compute", () => {
    template.resourceCountIs("Custom::AWSCDKOpenIdConnectProvider", 0);
    template.resourceCountIs("AWS::IAM::OIDCProvider", 0);
    template.resourceCountIs("AWS::Lambda::Function", 0);
  });

  it("exports the four outputs the repository variables are copied from", () => {
    for (const name of ["BucketName", "DistributionId", "SiteUrl", "DeployRoleArn"]) {
      template.hasOutput(name, {});
    }
  });
});

describe("bucket", () => {
  it("is private, retained, and has no lifecycle rule or website hosting", () => {
    onlyResource("AWS::S3::Bucket");
    template.hasResource("AWS::S3::Bucket", {
      DeletionPolicy: "Retain",
      UpdateReplacePolicy: "Retain",
      Properties: {
        PublicAccessBlockConfiguration: {
          BlockPublicAcls: true,
          BlockPublicPolicy: true,
          IgnorePublicAcls: true,
          RestrictPublicBuckets: true,
        },
        LifecycleConfiguration: Match.absent(),
        WebsiteConfiguration: Match.absent(),
      },
    });
  });

  it("denies every request that is not made over TLS", () => {
    template.hasResourceProperties("AWS::S3::BucketPolicy", {
      PolicyDocument: {
        Statement: Match.arrayWith([
          Match.objectLike({
            Effect: "Deny",
            Action: "s3:*",
            Condition: { Bool: { "aws:SecureTransport": "false" } },
          }),
        ]),
      },
    });
  });

  it("allows reads only to this CloudFront distribution", () => {
    const { resource: policy } = onlyResource("AWS::S3::BucketPolicy");
    const { key: distributionKey } = onlyResource("AWS::CloudFront::Distribution");
    const allows = statementsOf(policy).filter((s) => s.Effect === "Allow");

    expect(allows.length).toBeGreaterThan(0);
    for (const statement of allows) {
      expect(statement.Principal).toEqual({ Service: "cloudfront.amazonaws.com" });
      const condition = JSON.stringify(statement.Condition);
      expect(condition).toContain("AWS:SourceArn");
      expect(condition).toContain(`{"Ref":"${distributionKey}"}`);
    }
    const actions = new Set(allows.flatMap((s) => [s.Action].flat()));
    expect(actions).toEqual(new Set(["s3:GetObject", "s3:ListBucket"]));
  });
});

describe("distribution", () => {
  it("reaches the bucket only through Origin Access Control", () => {
    onlyResource("AWS::CloudFront::OriginAccessControl");
    template.resourceCountIs("AWS::CloudFront::CloudFrontOriginAccessIdentity", 0);
    template.hasResourceProperties("AWS::CloudFront::OriginAccessControl", {
      OriginAccessControlConfig: {
        OriginAccessControlOriginType: "s3",
        SigningBehavior: "always",
        SigningProtocol: "sigv4",
      },
    });
    template.hasResourceProperties("AWS::CloudFront::Distribution", {
      DistributionConfig: {
        Origins: [Match.objectLike({ OriginAccessControlId: Match.anyValue() })],
      },
    });
  });

  it("redirects to HTTPS, honours upload-time caching and has no error-page fallback", () => {
    template.hasResourceProperties("AWS::CloudFront::Distribution", {
      DistributionConfig: {
        DefaultRootObject: "index.html",
        DefaultCacheBehavior: {
          ViewerProtocolPolicy: "redirect-to-https",
          CachePolicyId: CACHING_OPTIMIZED_ID,
          Compress: true,
        },
        // errorResponses would turn a missing data file into 200 + HTML (Step 5).
        CustomErrorResponses: Match.absent(),
        CacheBehaviors: Match.absent(),
        PriceClass: "PriceClass_All",
        HttpVersion: "http2and3",
        IPV6Enabled: true,
      },
    });
  });

  it("runs lib/spa-rewrite.js, unchanged, on every viewer request", () => {
    const { key: functionKey } = onlyResource("AWS::CloudFront::Function");
    template.hasResourceProperties("AWS::CloudFront::Function", {
      FunctionCode: readFileSync(new URL("../lib/spa-rewrite.js", import.meta.url), "utf8"),
      FunctionConfig: { Runtime: "cloudfront-js-2.0" },
    });
    template.hasResourceProperties("AWS::CloudFront::Distribution", {
      DistributionConfig: {
        DefaultCacheBehavior: {
          FunctionAssociations: [
            {
              EventType: "viewer-request",
              FunctionARN: { "Fn::GetAtt": [functionKey, "FunctionARN"] },
            },
          ],
        },
      },
    });
  });
});

describe("response headers", () => {
  it("sends the exact Content-Security-Policy and the other security headers", () => {
    onlyResource("AWS::CloudFront::ResponseHeadersPolicy");
    template.hasResourceProperties("AWS::CloudFront::ResponseHeadersPolicy", {
      ResponseHeadersPolicyConfig: {
        SecurityHeadersConfig: {
          ContentSecurityPolicy: { ContentSecurityPolicy: CSP, Override: true },
          StrictTransportSecurity: {
            AccessControlMaxAgeSec: 63_072_000, // 730 days
            IncludeSubdomains: true,
            Override: true,
          },
          ContentTypeOptions: { Override: true },
          FrameOptions: { FrameOption: "DENY", Override: true },
          ReferrerPolicy: { ReferrerPolicy: "strict-origin-when-cross-origin", Override: true },
        },
      },
    });
  });

  it("is attached to the default behaviour", () => {
    const { key: policyKey } = onlyResource("AWS::CloudFront::ResponseHeadersPolicy");
    template.hasResourceProperties("AWS::CloudFront::Distribution", {
      DistributionConfig: { DefaultCacheBehavior: { ResponseHeadersPolicyId: { Ref: policyKey } } },
    });
  });
});

describe("deploy role", () => {
  it("is the only role, and trusts exactly main of this repository through the imported provider", () => {
    onlyResource("AWS::IAM::Role");
    template.hasResourceProperties("AWS::IAM::Role", {
      RoleName: "ProjetoAplicado2Deploy",
      MaxSessionDuration: 3600,
      AssumeRolePolicyDocument: {
        Statement: [
          {
            Effect: "Allow",
            Action: "sts:AssumeRoleWithWebIdentity",
            Principal: { Federated: OIDC_PROVIDER_ARN },
            // exact: no StringLike, no extra key, the two subjects and nothing else
            Condition: Match.exact({
              StringEquals: {
                "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
                "token.actions.githubusercontent.com:sub": SUBJECTS,
              },
            }),
          },
        ],
      },
    });
  });

  it("has no wildcard anywhere in its trust policy", () => {
    const { resource: role } = onlyResource("AWS::IAM::Role");
    expect(JSON.stringify(role["Properties"]["AssumeRolePolicyDocument"])).not.toMatch(/[*?]/);
  });

  it("can do exactly the documented actions, on exactly this bucket and this distribution", () => {
    const { key: roleKey, resource: role } = onlyResource("AWS::IAM::Role");
    const { key: bucketKey } = onlyResource("AWS::S3::Bucket");
    const { key: distributionKey } = onlyResource("AWS::CloudFront::Distribution");
    const { resource: policy } = onlyResource("AWS::IAM::Policy");

    expect(role["Properties"]["ManagedPolicyArns"]).toBeUndefined();
    expect(role["Properties"]["Policies"]).toBeUndefined();
    expect(policy["Properties"]["Roles"]).toEqual([{ Ref: roleKey }]);

    const grants = statementsOf(policy).flatMap((statement) => {
      expect(statement.Effect).toBe("Allow");
      expect(statement.Condition).toBeUndefined();
      return [statement.Action].flat().flatMap((action) =>
        [statement.Resource].flat().map((resource) => ({ action, resource })),
      );
    });

    expect(new Set(grants.map((g) => g.action))).toEqual(new Set(DEPLOY_ACTIONS));

    const bucketArn = { "Fn::GetAtt": [bucketKey, "Arn"] };
    for (const { action, resource } of grants) {
      expect(resource, action).not.toBe("*");
      if (action === "s3:ListBucket") {
        expect(resource, action).toEqual(bucketArn);
      } else if (action.startsWith("s3:")) {
        expect(resource, action).toEqual({ "Fn::Join": ["", [bucketArn, "/*"]] });
      } else {
        const text = JSON.stringify(resource);
        expect(text, action).toContain(`arn:aws:cloudfront::${ACCOUNT}:distribution/`);
        expect(text, action).toContain(`{"Ref":"${distributionKey}"}`);
      }
    }
  });
});
```

Notes on the less obvious lines:

- `new App({ context: cdkJsonContext() })` — the CLI reads `cdk.json`, a bare
  `new App()` in a test does not. Without the same feature flags
  (`minimizePolicies`, …) the tested template would differ from the deployed one.
- The permission test expands every statement into `(action, resource)` pairs,
  so it holds whether or not CDK merges statements, and fails on any extra
  action, any `*` resource, any condition, or any managed policy.
- If a CDK upgrade renames an internal detail (e.g. the logical shape of a
  joined ARN), fix the expectation in the same PR as the upgrade — never
  loosen it to `Match.anyValue()`.

### `infra/test/spa-rewrite.test.ts`

```ts
import { readFileSync } from "node:fs";
import { createContext, runInContext } from "node:vm";
import { describe, expect, it } from "vitest";

interface CfValue {
  value: string;
}

interface CfRequest {
  method: string;
  uri: string;
  querystring: Record<string, CfValue>;
  headers: Record<string, CfValue>;
  cookies: Record<string, CfValue>;
}

interface CfEvent {
  version: "1.0";
  context: { eventType: "viewer-request" };
  viewer: { ip: string };
  request: CfRequest;
}

type Handler = (event: CfEvent) => CfRequest;

/**
 * Evaluates the exact file CloudFront runs (the stack test proves the deployed
 * code equals this file) in an empty context, like the CloudFront runtime:
 * no require, no process, nothing leaked from the test.
 */
function loadHandler(): Handler {
  const source = readFileSync(new URL("../lib/spa-rewrite.js", import.meta.url), "utf8");
  const context = createContext({});
  runInContext(source, context, { filename: "spa-rewrite.js" });
  const handler: unknown = context["handler"];
  if (typeof handler !== "function") {
    throw new Error("spa-rewrite.js must declare a top-level function named handler");
  }
  return handler as Handler;
}

const handler = loadHandler();

function viewerRequest(uri: string, querystring: Record<string, CfValue> = {}): CfEvent {
  return {
    version: "1.0",
    context: { eventType: "viewer-request" },
    viewer: { ip: "192.0.2.1" },
    request: { method: "GET", uri, querystring, headers: {}, cookies: {} },
  };
}

describe("spa-rewrite (CloudFront Function)", () => {
  it.each([
    "/",
    "/candidatos",
    "/candidatos/",
    "/candidatos/a-b-123",
    "/sobre",
  ])("serves the app shell for the route %s", (uri) => {
    expect(handler(viewerRequest(uri)).uri).toBe("/index.html");
  });

  it.each([
    "/index.html",
    "/data/latest.json",
    "/data/20990101T000000Z/index.json",
    "/data/20990101T000000Z/candidates/a-b-123.json",
    "/data/20990101T000000Z/photos/240000000001.jpg",
    "/data/20990101T000000Z/dataset.sqlite",
    "/assets/x.js",
    "/assets/index-3f2a9c.css",
    "/favicon.svg",
    "/robots.txt",
  ])("leaves the file %s untouched, so a missing file stays a 404", (uri) => {
    expect(handler(viewerRequest(uri)).uri).toBe(uri);
  });

  it("keeps the query string, so search filters survive a reload", () => {
    const querystring = { q: { value: "maria" }, cargo: { value: "federal" } };
    const result = handler(viewerRequest("/candidatos", querystring));
    expect(result.uri).toBe("/index.html");
    expect(result.querystring).toEqual(querystring);
  });

  it("returns the request, never a response, so CloudFront continues to the origin", () => {
    const result = handler(viewerRequest("/sobre"));
    expect(result.method).toBe("GET");
    expect(result).not.toHaveProperty("statusCode");
  });
});
```

### Run them

```bash
cd infra
npm test        # vitest run --coverage: all green, coverage table printed (no threshold)
```

From the repository root: `make test-infra`.

## Step 7 — Check it locally (no credentials needed)

Exactly what the CI job `Infrastructure (CDK)` runs:

```bash
cd infra
npm run lint
npm run typecheck
npm test                       # assertion tests (Step 6b); coverage reported
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

1. Edit `infra/lib/site-stack.ts` in a branch. If the change deliberately
   alters an invariant (a new header, a new permission), update the matching
   assertion in `infra/test/site-stack.test.ts` **in the same PR**, and say
   why in the PR description. CI runs the tests and `cdk synth`.
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

## Done when

- [ ] `npm run lint`, `npm run typecheck` and `npm test` pass in `infra/`; the
      coverage table is printed (no threshold).
- [ ] Every row of the invariants table in Step 6b has a passing test, and
      `spa-rewrite.test.ts` covers the full rewrite table.
- [ ] `npx cdk synth --quiet` is clean and the trust policy shows exactly the two subjects.
- [ ] The CI job `Infrastructure (CDK)` is green on the PR that adds `infra/`.
- [ ] The stack is deployed (Step 8), its outputs recorded (Step 9) and the
      read-only checks of Step 10 pass.
