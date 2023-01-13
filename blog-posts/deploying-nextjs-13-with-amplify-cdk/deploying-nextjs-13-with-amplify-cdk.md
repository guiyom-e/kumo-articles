---
published: true
title: 'Deploying Next.js 13 with Amplify CDK'
cover_image: 'https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pm8b99id1yfngkax8oa3.png'
description: 'How to deploy a Next.js app on AWS Amplify with a CDK construct'
tags: aws, cdk, nextjs, amplify
series:
canonical_url:
---

> 📜 How to deploy a Next.js app step by step with an Amplify CDK construct, avoiding you all the pains.

> 🐝 TLDR: an example app has just been added in [`swarmion`](https://github.com/swarmion/swarmion/pull/433), so you can now bootstrap a ready-to-use Next.js project!

[Next.js](https://nextjs.org/) is a popular React framework to deploy web apps with advanced features like Server Side Rendering (SSR), Image optimization or Incremental Static Regeneration (ISR). In my last project, I chose this framework notably to ensure a [good SEO](https://nextjs.org/learn/seo/introduction-to-seo).

To deploy the app, I had the requirement to use AWS as a cloud-provider, with a limited budget, so neither Vercel (a safe choice, as it is the company which develops Next.js), nor a containerized solution like the managed AWS ECS Fargate service could satisfy my needs. Then, I started with a serverless hosting thanks to the [serverless-next](https://github.com/serverless-nextjs/serverless-next.js) project, which provides a serverless plugin or a CDK construct to deploy a Next.js stack. However, this project is no more well-maintained and new Next.js features (like middlewares, etc.) are not supported.

Here comes AWS Amplify which has just announced on November 17th to [support Next.js 12 and 13](https://aws.amazon.com/fr/about-aws/whats-new/2022/11/aws-amplify-hosting-support-next-js-12-13) 🎉! Amplify is an AWS service to build and host full-stack applications. It uses serverless services under the hood. I tested it with @mamadoudicko and @alexandreperni4 and we will explain you how to setup a production-ready app!

## Introducing Amplify's AWS CDK

AWS CDK, for AWS Cloud Development Kit, is a multi-languages framework for writing infrastructure as code and deploying it through AWS CloudFormation. Our examples will be written in Typescript, but other languages like Python, Java, C# or Go can be used as well.

To deploy with the AWS CDK, you would first need to declare a `cdk.json` file at the root of your frontend repository. This allows you to tell the AWS CDK where is the entry point of the CDK app, `hosting/bin.ts` here:

```json
{
  "app": "pnpm ts-node hosting/bin.ts"
}
```

You need to install these dev dependencies:

```bash
pnpm add -D aws-cdk-lib
pnpm add -D @aws-cdk/aws-amplify-alpha
pnpm add -D constructs
```

You will also need some configuration files to bootstrap the app:

- `.nvmrc` file to define your node version
- `tsconfig.json` TypeScript configuration file, with `target` version set to "es6" or more

In the `hosting` folder, you then need to add a `bin.ts` file to declare the `cdk` app:

```tsx
import * as cdk from 'aws-cdk-lib';

import { AmplifyStack } from './stack';

const app = new cdk.App();

new AmplifyStack(app, 'NextJsSampleStack', {
  description: 'Cloudformation stack containing the Amplify configuration',
});
```

Nothing fancy at this point, the interesting part is in the `./stack.ts` configuration file:

```tsx
import { App } from '@aws-cdk/aws-amplify-alpha';
import { aws_iam, CfnOutput, Stack, StackProps } from 'aws-cdk-lib';
import { BuildSpec } from 'aws-cdk-lib/aws-codebuild';
import { Construct } from 'constructs';

export class AmplifyStack extends Stack {
  constructor(scope: Construct, id: string, props: StackProps) {
    super(scope, id, props);

    // Define Amplify app
    const amplifyApp = new App(this, 'AmplifyAppResource', {
      appName: 'NextJS app',
      description: 'My NextJS APP deployed with Amplify',

      // ⬇️ configuration items to be defined ⬇️
      role,
      sourceCodeProvider,
      buildSpec,
      autoBranchCreation,
      autoBranchDeletion,
      environmentVariables,
      // ⬆️ end of configuration ⬆️
    });

    // Attach your main branch and define the branch settings (see below)
    const mainBranch = amplifyApp.addBranch('main', {
      autoBuild: false, // set to true to automatically build the app on new pushes
      stage: 'PRODUCTION',
    });

    new CfnOutput(this, 'appId', {
      value: amplifyApp.appId,
    });
  }
}
```

Let’s go through what we need for the different variables within the configuration object.

## 📖 Step-by-step guide

### Define a role to Amplify (`role`)

This is needed to add a custom role that will be assumed by the Amplify resource.

```tsx
import { ManagedPolicy, Role, ServicePrincipal } from 'aws-cdk-lib/aws-iam';

const role = new Role(this, 'AmplifyRoleWebApp', {
  assumedBy: new ServicePrincipal('amplify.amazonaws.com'),
  description: 'Custom role permitting resources creation from Amplify',
  managedPolicies: [ManagedPolicy.fromAwsManagedPolicyName('AdministratorAccess-Amplify')],
});
```

### Connection to your Github repository (`sourceCodeProvider`)

The `sourceCodeProvider` configuration allows Amplify to access the source code of your application. To connect a Github repository, you can use the declaration below:

```tsx
import { GitHubSourceCodeProvider } from '@aws-cdk/aws-amplify-alpha/lib/source-code-providers';
import { SecretValue } from 'aws-cdk-lib';

const sourceCodeProvider = new GitHubSourceCodeProvider({
  // GitHub token should be saved in a secure place, we recommend AWS Secret Manager:
  oauthToken: SecretValue.secretsManager('GITHUB_TOKEN_KEY'), // replace GITHUB_TOKEN_KEY by the name of the Secrets Manager resource storing your GitHub token
  owner: '<user name of the GitHub repository owner>',
  repository: '<name of the Github repository>',
});
```

To get a Github token, go to your Github account develop settings and generate a personal access token. Amplify will need quite high access rights to your repository as it will need to generate ssh keys to clone the repository.

To store the token in AWS, we recommend to use the [AWS Secret Manager](https://aws.amazon.com/secrets-manager) service. Be careful to choose a plain text secret (in the AWS console select "Store a new secret", "Other type of secret" and finally "Plaintext"). Choose a key name that matches the key used in the `GitHubSourceCodeProvider` construct (`GITHUB_TOKEN_KEY` in this example).

### Build settings (`buildSpec`)

Then, you need to define how the app will be built. Below is an example with `pnpm` package manager:

```tsx
import { BuildSpec } from 'aws-cdk-lib/aws-codebuild';

import { environmentVariables } from './environmentVariables';

export const buildSpec = BuildSpec.fromObjectToYaml({
  version: '1.0',
  applications: [
    {
      frontend: {
        phases: {
          preBuild: {
            commands: [
              // Install the correct Node version, defined in .nvmrc
              'nvm install',
              'nvm use',
              // Install pnpm
              'corepack enable',
              'corepack prepare pnpm@latest --activate',
              // Avoid memory issues with node
              'export NODE_OPTIONS=--max-old-space-size=8192',
              // Ensure node_modules are correctly included in the build artifacts
              'pnpm install',
            ],
          },
          build: {
            commands: [
              // Allow Next.js to access environment variables
              // See https://docs.aws.amazon.com/amplify/latest/userguide/ssr-environment-variables.html
              `env | grep -E '${Object.keys(environmentVariables).join('|')}' >> .env.production`,
              // Build Next.js app
              'pnpm next build --no-lint',
            ],
          },
        },
        artifacts: {
          baseDirectory: '.next',
          files: ['**/*'],
        },
      },
    },
  ],
});
```

This example is using `pnpm`, but you can choose to use any other package manager, such as `npm` or `yarn`. A few things to note here :

- Setting the `--max-old-space-size` node option is important to prevent out of memory (OOM) errors while building your application when it reaches a certain size.
- To let your server-side code access your environment variables, you have to include them in a .env.production file. This allows you to put secrets known only from your CI environment in the .env.production file created on the fly.
- The artifacts section is very important as it configures which files from the build step will be included in the artifacts and thus available at run time.

### Auto-deploy when remote branches satisfying a certain name pattern are created

Amplify allow to create a dedicated environment when you push to a branch with a specific name pattern.

```tsx
import { AutoBranchCreation } from '@aws-cdk/aws-amplify-alpha';

export const autoBranchCreation: AutoBranchCreation = {
  autoBuild: true,
  patterns: ['feature/*'],
  pullRequestPreview: true,
};
```

You might want to disable the feature in production. For this, simply use `undefined` instead of this configuration.

> 💡 Note that this environment can be automatically removed once the branch is deleted if the parameter `autoBranchDeletion` is set to `true`.

### Configuring environment variables

```tsx
export const environmentVariables = {
  // https://docs.aws.amazon.com/amplify/latest/userguide/build-settings.html#enable-diff-deploy
  AMPLIFY_DIFF_DEPLOY: 'false',
};
```

> 💡 Environment variables can be defined globally or in branch environments.

### Setting up a domain

You can add the domain configuration directly in the `stack.ts` file, to have access to the `amplifyApp` and `mainBranch` variables:

```tsx
const domain = amplifyApp.addDomain('your-domain.com', {
  autoSubdomainCreationPatterns: ['feature/*'],
  enableAutoSubdomain: true,
});
```

Don't forget to map it with a specific branch by adding the following line:

```tsx
domain.mapRoot(mainBranch);
```

> 💡 If the domain is already defined in AWS Route 53 service, it will be automatically linked to your stack: AWS will add the appropriate DNS records in the hosted zone. Otherwise, you will need to add the records manually in your domain name provider.

### Deployment

Almost there! Before being able to detect and deploy Next.js, Amplify needs to be configured with the platform type `WEB_COMPUTE`. It needs to be done after Amplify app deployment. As there is no option in Cloudformation for now, the best way to do it programmatically is to use a Custom Resource construct:

```ts
import { AwsCustomResource, AwsCustomResourcePolicy } from 'aws-cdk-lib/custom-resource';

// Set Amplify platform type to WEB_COMPUTE
new AwsCustomResource(this, 'AmplifySetPlatform', {
  onUpdate: {
    service: 'Amplify',
    action: 'updateApp',
    parameters: {
      appId: amplifyApp.appId,
      platform: 'WEB_COMPUTE',
    },
    physicalResourceId: PhysicalResourceId.of('AmplifyCustomResourceSetPlatform'),
  },
  policy: AwsCustomResourcePolicy.fromSdkCalls({
    resources: [amplifyApp.arn],
  }),
});
```

> 💡 This construct is equivalent to a call to AWS SDK, that can also be written with an `aws` CLI command:

```bash
aws amplify update-app --app-id <yourAppID> --platform  WEB_COMPUTE
```

Finally, to deploy your AWS CDK app, navigate to the folder containing the `cdk.json` file, then enter the following command:

```bash
pnpm cdk bootstrap && pnpm cdk deploy
```

When the deployment ends successfully, the id of your Amplify app will be printed in your command line interface.

> ⚠️ This step only deploys an Amplify stack, but it does not build nor deploy the Next.js app yet! To do so, you can either:
>
> - Set the `autoBuild` parameter to `true` in a branch setting to deploy on GitHub pushes (or other source provider)
> - Use a webhook as explained below.

### Adding a webhook

In many cases, a dedicated CI/CD platform is used to test and deploy an app. Then, the direct GitHub integration that triggers an Amplify build at each push on a particular branch is no more sufficient. So you may want to trigger a deployment directly in a CD step. Amplify allows to trigger a deployment with webhooks. This is not yet configurable through the CDK but you can use the Amplify CLI:

```bash
aws amplify create-webhook --app-id <yourAppID> --branch-name <yourBranchName>
```

[https://awscli.amazonaws.com/v2/documentation/api/latest/reference/amplify/create-webhook.html](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/amplify/create-webhook.html)

In your CD, add `curl [webhook url]`.

> 💡 Alternatively, you can use a Custom Resource construct to create your webhook:

```ts
// Create a webhook to use in your proper CI/CD
const webhookCustomResource = new AwsCustomResource(this, 'AmplifyWebhook', {
  onUpdate: {
    service: 'Amplify',
    action: 'createWebhook',
    parameters: {
      appId: amplifyApp.appId,
      branchName: 'main',
    },
    physicalResourceId: PhysicalResourceId.of('AmplifyCustomResourceWebhook'),
  },
  policy: AwsCustomResourcePolicy.fromSdkCalls({
    resources: [`${amplifyApp.arn}/webhooks/*`],
  }),
});

// Outputs the secret deployment webhook
webhookCustomResource.getResponseField('webhook.webhookUrl');
```

### Other interesting features

Other configurations are available: custom response headers, custom rewrites and redirects or even basic authentication to prevent access to any test environment!

## 📚 Caveats with a monorepo

In a monorepo, additional configuration is required:

- Add `appRoot` property in the build settings, with the frontend directory path.
- Ensure `node_modules` are correctly included in the build. In the context of a monorepo, most package managers will put the `node_modules` at the root of your repository. A nice workaround, with `pnpm`, is to set the virtual store directory inside the frontend directory when installing the project: `pnpm install --virtual-store-dir [your frontend directory]/node_modules/.pnpm`

See [swarmion](https://github.com/swarmion/swarmion/pull/433) full implementation example.

## ✨ Conclusion

If you plan to go on a monorepo architecture, you can already use [Swarmion](https://www.swarmion.dev) to create high scalable serverless application including Next.js latest version by simply running: `pnpm create swarmion-app` and choosing the Next.js option.

Swarmion will pre-configure all required settings so that your app will smoothly be deployed to amplify.

To complete, there are other Next.js deployment solutions which may be interesting to explore like:

- [Vercel](https://vercel.com/guides/deploying-nextjs-with-vercel) (Next.js core team): probably the most user friendly infrastructure solution, if you can afford it on your project.
- [serverless-nextjs](https://github.com/serverless-nextjs/serverless-next.js#features): very interesting initiative but seems no longer maintained unfortunately. The resources declared in this library are very similar to the one deployed by Amplify under the hood, which probably made it the cheapest solution around for a while.
- [OpenNext project](https://open-next.js.org/) which uses the [NextJsSite SST construct](https://docs.sst.dev/constructs/NextjsSite). A promising project, but still a work in progress. This one focuses on the connector between the Next.js framework and the infrastructure, but will not provide the infrastructure as code template.
- [JetBridge cdk-nextjs construct](https://github.com/jetbridge/cdk-nextjs), another proposal to deploy Next.js with a construct.

Thanks a lot to @mamadoudicko, @alexandreperni4 and [@Antoine Apollis](https://github.com/apollisa) who co-wrote this article and tested the setup process in this [repository](https://github.com/alexandrepernin/nextjs-amplify)! We also relied a lot on this [AWS blog article](https://aws.amazon.com/blogs/mobile/deploy-a-nextjs-13-application-to-amplify-with-the-aws-cdk/) to build this guide.
