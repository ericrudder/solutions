![alt text](https://github.com/d-nishi/solutions/blob/master/featured-image-gitlab.png)
# Continuous Delivery with Gitlab and Pulumi on Amazon EKS

Authors: [Nishi Davidson](https://pulumi.quip.com/eLPAEA81Pyv) and [Sean Gillespie](https://pulumi.quip.com/DbWAEA1i3Dl)
Date: 05/16/2019

## Prerequisites

* An account on [https://app.pulumi.com](https://app.pulumi.com/) with an organization. Sign-in using your GitLab credentials. Pulumi can also be run from anywhere and the Pulumi application and infrastructure code can be hosted anywhere. We will use the latter flow in our scenario.
* The latest CLI. Installation instructions are [here](https://pulumi.io/quickstart/install.html).
* A bare repository. Set the remote URL to be your GitLab project.

## Concepts in Pulumi

### Organization of Pulumi Projects and Pulumi Stacks

All users in the Pulumi service will start with the hierarchy of an organization. This can be a specific Github, Gitlab or Atlassian organization or your solo organization. Inside each organization, users create Pulumi projects and stacks.
Pulumi [Projects](https://pulumi.io/reference/project.html) and [stacks](https://pulumi.io/reference/stack.html) are intentionally flexible to accommodate diverse needs across teams, applications, and infrastructure scenarios. Just like Git repos that work with varying approaches Pulumi projects and stacks allow you to organize your code within them. Immediate options include:

* **Monolithic project/stack structure:** A single project defines the infrastructure and application resources for an entire vertical service.

![alt text](https://github.com/d-nishi/solutions/blob/master/monolith.png)

* **Micro-stacks project/stack structure: **A project broken into separately managed smaller projects, often across different dimensions. 

![alt text](https://github.com/d-nishi/solutions/blob/master/microstack.png)

Working with Inter-Stack Dependencies with the latter option is more suited in a production setup giving users more flexibility and boundaries between their teams. We will use this structure in our example below. For more information on Pulumi projects and stacks, please refer to our documentation [here](https://pulumi.io/reference/organizing-stacks-projects.html).

### Tagging Pulumi Stacks to create Environments:

* Pulumi Stacks have associated metadata in the form of key/value tags. 
* You can assign custom tags to stacks (when logged into the [web backend](https://pulumi.io/reference/state.html)) to customize how stacks are listed in the [Pulumi Cloud Console](https://app.pulumi.com/?__hstc=228626179.56681581cf02b2e77b51bd3037fb698a.1554407757799.1557330248351.1557339846440.52&__hssc=228626179.2.1557339846440&__hsfp=3057520729). 
    * In our example below we have two environments prod and dev. 
    * We group stacks by environment by assigning custom `environment` tags `prod` and `dev` to the respective stacks
    * In the Pulumi Cloud Console, you’ll be able to group stacks by `Tag: environment:dev` and `Tag: environment:prod`. 
    
Please read more about [how to manage stack tags here](https://pulumi.io/reference/stack.html#stack-tags).

![alt text](https://github.com/d-nishi/solutions/blob/master/microstack-environment.png)

Let's now work through our example with Gitlab Pipelines.

## Gitlab Pipeline by Environment - Example 

1. We created a Gitlab Group called **pulumi**
2. We created 3 Gitlab projects called **sample-iam**, **sample-eks** and **sample-k8sapp**
3. We have 2 pipelines: **environment:dev** and **environment:prod**
    1. In the two pipelines, we have a total of “six” pulumi stacks: 
        1. **pulumi/sample-IAM/dev** and **pulumi/sample-iam/prod**
        2. **pulumi/sample-eks/dev** and **pulumi/sample-eks/prod**
        3. **pulumi/sample-k8sapp/dev** and **pulumi/sample-k8sapp/prod**
    2. **pulumi/sample-iam/dev** stack will trigger the downstream stack **pulumi/sample-eks/dev** provided the cycle of **pulumi preview → pulumi deploy** completes without any failure. 
    3. **pulumi/sample-eks/dev** will trigger the downstream stack **pulumi/sample-k8sapp/dev** provided the cycle of **pulumi preview → pulumi deploy** completes without any failure.

![alt text](https://github.com/d-nishi/solutions/blob/master/microstack-environment-gitlab.png)

4. To use Pulumi within GitLab CI, there are a few environment variables you’ll need to set for each build.
    1. The first is `PULUMI_ACCESS_TOKEN`, which is required to authenticate with **pulumi.com** in order to perform the preview or update. You can create a new Pulumi access token specifically for your CI/CD job on your [Pulumi Account page](https://app.pulumi.com/account/tokens?__hstc=228626179.56681581cf02b2e77b51bd3037fb698a.1554407757799.1557357031878.1557359155284.54&__hssc=228626179.2.1557359155284&__hsfp=3057520729).
    2. Next, you will also need to set environment variables specific to your cloud resource provider. For example, if your stack is managing resources on AWS, `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.

## Create Pulumi stacks and push the files to your Gitlab project

If you run `pulumi` from any branch other than the `master` branch, you will hit an error that the `PULUMI_ACCESS_TOKEN` environment variable cannot be accessed. You can fix this by specifying a wildcard regex to allow specific branches to be able to access the secret environment variables. Please refer to the [GitLab documentation](https://gitlab.com/help/user/project/protected_branches.md) to understand this better.

First we set up “three” Pulumi stacks: **sample-iam**; **sample-eks** and **sample-k8sApp** with stack tag:`environment:dev` 

**Step 1:** Create the pulumi stack "sample-IAM" and set stack tag "key:value" = "environment:dev". Update the `index.ts` file with the relevant code block as shown below and run `pulumi up `

We then initialize a new stack tag “key:value" = "environment:prod" and run  `pulumi up` with the same `index.ts` file

```
$ pulumi new aws-typescript --dir pulumi/sample-iam/dev

$ cat index.ts

import * as aws from "@pulumi/aws";

/*
 * Single step deployment of three IAM Roles
 */

function createIAMRole(name: string): aws.iam.Role {
    // Create an IAM Role...
    return new aws.iam.Role(`${name}`, {
        assumeRolePolicy: `{
            "Version": "2012-10-17",
            "Statement":[
              {
                "Sid": "",
                "Effect": "Allow",
                "Principal": {
                  "AWS": "arn:aws:iam::153052954103:root"
                },
                "Action": "sts:AssumeRole"
              }
            ]
           }
        `,
          tags: {
              "clusterAccess": `${name}-usr`,
          },
        });
    }

    // Administer Automation role for use in pipelines, e.g. gitlab CI, Teamcity, etc.
    export const AutomationRole = createIAMRole("AutomationRole");
    export const AutomationRoleArn = AutomationRole.arn;
    
$ pulumi up

//Initialize new pulumi stack in the format pulumi stack init <org name>/<project>/<stack>

$ pulumi stack init pulumi/sample-iam/prod
$ pulumi stack tag set environment prod
$ pulumi up

```

**Step 2:** Create the pulumi stack **sample-eks** and set stack tag "key:value" = "environment:dev". Update the `index.ts` file with the relevant code block as shown below, download the additional npm packages for EKS and Kubernetes and run `pulumi up`

We then initialize a new stack tag “key:value" = "environment:prod" and run  `pulumi up` with the same `index.ts` file

```
$ pulumi new aws-typescript --dir pulumi/sample-eks/dev

$ cat index.ts**

import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";
import * as eks from "@pulumi/eks";
import * as k8s from "@pulumi/kubernetes";
import * as pulumi from "@pulumi/pulumi";

const env = pulumi.getStack();
const iamstack = new pulumi.StackReference(`pulumi/sample-iam/${env}`);

const AutomationRoleArn = iamstack.getOutput("AutomationRoleArn")

/*
 * Single step deployment of EKS cluster with the most important variables and simple function to create two namespaces
 */

const vpc = new awsx.Network("vpc");
const cluster = new eks.Cluster("eks-cluster", {
  vpcId             : vpc.vpcId,
  subnetIds         : vpc.subnetIds,
  instanceType      : "t2.medium",
  nodeRootVolumeSize: 200,
  desiredCapacity   : 1,
  maxSize           : 2,
  minSize           : 1,
  deployDashboard   : false,
  vpcCniOptions     : {
    warmIpTarget    : 4,
  },
  roleMappings      : [
    // Map IAM role arn "AutomationRoleArn" to the k8s user with name "automation-usr", e.g. gitlab CI
    {
      groups    : ["pulumi:automation-grp"],
      roleArn   : AutomationRoleArn,
      username  : "pulumi:automation-usr",
    },
  ],
});

export const clusterName = cluster.eksCluster.name;

/*
 * Single Step deployment of k8s RBAC configuration
 */

new k8s.rbac.v1.Role("AutomationRole", {
  metadata: {
    name: "AutomationRole",
    namespace: "automation",
  },
  rules: [{
    apiGroups: ["*"],
    resources: ["*"],
    verbs: ["*"],
  }]
}, {provider: cluster.provider});

new k8s.rbac.v1.RoleBinding("automation-binding", {
  metadata: {
    name: "automation-binding",
    namespace: "automation",
  },
  subjects: [{
     kind: "User",
     name: "pulumi:automation-usr",
     apiGroup: "rbac.authorization.k8s.io",
  }],
  roleRef: {
    kind: "Role",
    name: "AutomationRole",
    apiGroup: "rbac.authorization.k8s.io",
  },
}, {provider: cluster.provider});

export const kubeconfig = cluster.kubeconfig.apply(JSON.stringify)

$ npm install --save @pulumi/eks @pulumi/kubernetes
$ pulumi up

//Initialize new pulumi stack in the format pulumi stack init <org name>/<project>/<stack>

$ pulumi stack init pulumi/sample-eks/prod
$ pulumi stack tag set environment prod
$ pulumi up

```

**Step 3:** Create the pulumi stack "sample-EKS" and set stack tag "key:value" = "environment:dev". Update the `index.ts` file with the relevant code block as shown below, download the additional npm packages for EKS and Kubernetes and run `pulumi up`

We then initialize a new stack tag “key:value" = "environment:prod" and run  `pulumi up` with the same `index.ts` file

```
$ pulumi new aws-typescript --dir pulumi/sample-k8sapp/dev 

$ cat index.ts

import * as aws from "@pulumi/aws";
import * as docker from "@pulumi/docker";
import * as k8s from "@pulumi/kubernetes";
import * as pulumi from "@pulumi/pulumi";

const env = pulumi.getStack();
const eksCluster = new pulumi.StackReference(`pulumi/sample-eks/${env}`);

const kubeconfig = eksCluster.getOutput("kubeconfig");

const k8sProvider = new k8s.Provider("eks-cluster", {
    kubeconfig: kubeconfig,
 });

/*
 * Single step deployment of one docker container in ECR
 */

function getImageRegistry(repo: aws.ecr.Repository) {
    return repo.registryId.apply(async registryId => {
        if (!registryId) {
            throw new Error("Expected registry ID to be defined during push");
        }
        const credentials = await aws.ecr.getCredentials({ registryId: registryId });
        const decodedCredentials = Buffer.from(credentials.authorizationToken, "base64").toString();
        const [username, password] = decodedCredentials.split(":");
        if (!password || !username) {
            throw new Error("Invalid credentials");
        }
        return {
            server: credentials.proxyEndpoint,
            username: username,
            password: password,
        };
    });
}

const ecr1 = new aws.ecr.Repository("breathe");
const image1 = new docker.Image("breathe", {
    imageName: ecr1.repositoryUrl,
    build: {
        context: "./app",
        cacheFrom: true,
    },
    registry: getImageRegistry(ecr1),
});

// Declare the docker container based deployment
  const appLabels = { app: appName };
  const breathecontainer = new k8s.apps.v1beta1.Deployment(appName, {
      spec: {
          selector: { matchLabels: appLabels },
          replicas: 1,
          template: {
              metadata: { labels: appLabels },
              spec: { containers: [{ name: appName, image: image1.imageName }] }
          }
      },
    }, { provider: k8sProvider });**

$ npm install --save @pulumi/kubernetes @pulumi/docker
$ pulumi up

//Initialize new pulumi stack

$ pulumi stack init pulumi/sample-eks/prod
$ pulumi stack tag set environment prod
$ pulumi up

```

## Using Gitlab Pipelines with the “six” Pulumi stacks in environment:dev and environment:prod

GitLab pipelines are configured using `.gitlab-ci.yml` files in the root of each repository. GitLab Silver and above is capable of [running pipelines that cross project boundaries](https://docs.gitlab.com/ee/ci/multi_project_pipelines.html#passing-variables-to-a-downstream-pipeline), so we will be using that to construct our pipeline.

All three `.gitlab-ci.yml` files that we use are very similar in structure. The base one, sample`-iam`, looks like this:

```
image:
  name: pulumi/pulumi:v0.17.10
  entrypoint:
    - '/usr/bin/env'
    - 'PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin'

stages:
  - preview
  - update
  - downstream

Pulumi Preview:
  stage: preview
  script:
    - npm ci
    - pulumi stack select pulumi/sample-iam/$DEPLOY_ENVIRONMENT
    - pulumi preview

Pulumi Update:
  stage: update
  script:
    - npm ci
    - pulumi stack select pulumi/sample-iam/$DEPLOY_ENVIRONMENT
    - pulumi update --skip-preview

Update EKS:
  stage: downstream
  trigger: pulumi-gitlab/sample-eks
```

This file describes a three-stage pipeline for the `sample-iam` project:

1. First, we run a preview for the requested deployment environment, failing the pipeline if the preview fails
2. If the preview was successful, we run `pulumi update`, which deploys the IAM changes
3. Finally, we trigger the pipeline in `pulumi-gilab/sample-eks`, which triggers the next pipeline in our pipeline daisy chain illustrated in the above image.

Despite being powerful, conceptually this setup is quite simple and doesn't require much code to get right.

![alt text](https://github.com/d-nishi/solutions/blob/master/pipeline-image.png)

Upon a successful update, each tier's pipeline will trigger a pipeline for the tiers that depend on it. Pulumi's StackReference feature ensures that the dependent tiers receive new copies of the outputs exported from the IAM stack, so the deployment flows naturally through the pipeline!

This brings us to the end of our CD solution with Pulumi and Gitlab on Amazon EKS. For more examples, refer to our open source repository [here](https://github.com/pulumi/examples). Refer to my previous blog on Amazon EKS and k8s RBAC [here](https://blog.pulumi.com/simplify-kubernetes-rbac-in-amazon-eks-with-open-source-pulumi-packages).
