# kubernetes-conjur-demo

This repo demonstrates an app retrieving secrets from Conjur or a Dynamic Access
Provider (DAP) follower running in Kubernetes or OpenShift.

**Note:** These demo scripts have been tested with the following products:
  - Dynamic Access Provider v11+ or Conjur OSS v1.5+.
    - Older versions of Conjur Enterprise v4 are not supported.
  - cyberark/conjur-authn-k8s-client v0.18+
  - cyberark/secretless-broker v1.0+

## Demo Workflow

This demo works with both Conjur OSS and DAP. You can tailor the specific
steps that the demo scripts perform using environment variable settings,
based on your specific needs (e.g. do you want the scripts to load policy
into Conjur master, or will you be doing that independently) and whether
you are using Conjur OSS or DAP.

The steps, or workflow, that the scripts perform can be categorized into
three phases (the `Security Admin Steps` phase being optional):

- Demo Preparation:
  - Check for required environment settings
  - Log into the OpenShift platform, if using OpenShift

- Security Admin Steps (Optional):
  - Create a Conjur CLI deployment for demo to use, if not already present
  - Load Conjur policies for the application into Conjur master
  - Initialize the Conjur master's certificate authority

- Demo Application Deployment:
  - Create a demo application namespace
  - Create a RoleBinding to allow the Conjur Kubernetes authenticator
    to access resources in the demo application namespace
  - Retrieve the Conjur CA certificate and store it in a ConfigMap
  - Build demo application containers and push them to a registry
  - Deploy a few instances of the [pet store](https://github.com/conjurdemos/pet-store-demo/)
    demo application (including its database) set up to run with:
    - [Secretless Broker](https://github.com/cyberark/secretless-broker) sidecar
      to manage the database connection
    - [Conjur Kubernetes Authenticator Client](https://github.com/cyberark/conjur-authn-k8s-client)
      sidecar to provide the Conjur access token, and Summon to inject the database
      credentials into the app environment
    - [Conjur Kubernetes Authenticator Client](https://github.com/cyberark/conjur-authn-k8s-client)
      init container to provide the Conjur access token, and Summon to inject the database
      credentials into the app environment
  - Verify that demo applications can be accessed and are retrieving
    secrets from Conjur, and print a summary

You may choose to skip the `Security Admin Steps` for example if you plan on
loading Conjur policy separately from these scripts.

## The Pet Store App

The pet store demo app is based on the `cyberark/demo-app` Docker image. It can
be deployed with a PostgreSQL or MySQL database and the DB credentials are stored
in Conjur.

## Requirements

This demo works with both Conjur OSS and DAP, but the requirements vary depending
on which you are using.

To run this demo, you must load policy. You may want to **set up a separate
Conjur cluster** purely for the purpose of running this demo since you may not want
to load demo policy in your production environment.

There are a couple of options available for deploying a Conjur cluster:

- You can deploy a demo Conjur DAP cluster, including a Conjur master node
  and several Conjur follower nodes using the
  [Kubernetes Conjur deploy scripts](https://github.com/cyberark/kubernetes-conjur-deploy).
  See the [demo guide to deploying a master cluster](https://github.com/cyberark/kubernetes-conjur-deploy/blob/master/CONTRIBUTING.md#deploying-conjur-master-and-followers-test-and-demo-only)
  for more information.
- You can deploy a Conjur OSS cluster using the  
  [Conjur OSS Helm Chart](https://github.com/cyberark/conjur-oss-helm-chart).

### Requirements for Conjur OSS

Supported platforms:
- Kubernetes v1.16+

- To run this demo with Conjur OSS, you must have deployed Conjur OSS to your
  Kubernetes cluster using the [helm chart](https://github.com/cyberark/conjur-oss-helm-chart).
- You must have credentials for a Conjur user that can load policy

### Requirements for Dynamic Access Provider

Supported platforms:
- Kubernetes v1.16+
- OpenShift 3.11

To run this demo with DAP, you must have deployed a DAP follower to your
Kubernetes cluster following the [documentation](https://docs.cyberark.com/Product-Doc/OnlineHelp/AAM-DAP/Latest/en/Content/Integrations/ConjurDeployFollowers.htm).

*Note: if you have been following the [DAP documentation](https://docs.conjur.org/Latest/en/Content/Integrations/Kubernetes_deployConjur.htm),
you may have completed this step while you were already logged into the Conjur
master. If not, you will need to do so now.*

## Usage instructions

To run this demo via the command line, ensure you are logged in to the correct
cluster. Make sure you have followed the instructions in the
[requirements](#requirements) section so that your Conjur environment is prepared.

Set the following variables in your local environment:

| Environment Variable | Definition | Mandatory | Default | Example |
|--|--|--|--|--|
| `AUTHENTICATOR_ID` | The Conjur Kubernetes authenticator ID to use in Conjur policy (refer to the [documentation on enabling Conjur authenticators](https://docs.cyberark.com/Product-Doc/OnlineHelp/AAM-DAP/Latest/en/Content/Integrations/Kubernetes_deployApplicationCluster.htm?tocpath=Integrations%7COpenShift%252C%20Kubernetes%7C_____5)). | Yes | - | `my-authn-id` |
| `CONFIGURE_CONJUR_MASTER` | Boolean to determine if security admin steps described above (initialize Conjur CA, configure Conjur policy) should be performed by the scripts. NOTE: This setting only applies when running the scripts with DAP. When running with Conjur OSS (i.e. when `CONJUR_OSS_HELM_INSTALLED` is set to `true`), then security admin steps are performed regardless of this setting. | No | `false` | `true` |
| `CONJUR_ACCOUNT` | The account your Conjur / DAP cluster is configured to use. | Yes | - | `myConjurAccount` |
| `CONJUR_ADMIN_PASSWORD` | The `admin` user password that was created when you created the account on your Conjur / DAP cluster. | Yes | - | |
| `CONJUR_AUTHN_LOGIN_RESOURCE` | Type of Kubernetes resource to use as Conjur [application identity](https://docs.cyberark.com/Product-Doc/OnlineHelp/AAM-DAP/Latest/en/Content/Integrations/Kubernetes_AppIdentity.htm). | No | `service_account` | `deployment` |
| `CONJUR_NAMESPACE_NAME` | The namespace to which Conjur was deployed. | Yes | - | `conjur-namespace` |
| `CONJUR_OSS_HELM_INSTALLED` | Set to `true` if you are using Conjur OSS. | No | `false` | `true` |
| `DOCKER_REGISTRY_URL` | Set to the Docker registry to use for your platform for pushing/pulling application images that get built by the script. Examples are `docker.io` for DockerHub or `us.gcr.io` for GKE. | Yes | - | `us.gcr.io` |
| `DOCKER_REGISTRY_PATH` | Set to the Docker organization to use for your platform for pushing/pulling application images that get built by the script. | Yes | - | `myorganization` |
| `PLATFORM` | Set this variable to `kubernetes` or `openshift`, depending on which type of cluster you will be running the demo in. | No | `kubernetes` | `openshift` |
| `TEST_APP_DATABASE` | The type of database to run with the pet store app. Supported values are `mysql`, `mssql`, and `postgres`. | Yes | - | `mysql` |
| `TEST_APP_NAMESPACE_NAME` | The Kubernetes namespace in which your test app will be deployed. The demo scripts create this namespace for you if necessary. | Yes | - | `demo-namespace` |
| `TEST_APP_LOADBALANCER_SVCS` | Boolean to determine whether to use LoadBalancer type service instead of NodePort services. When running MiniKube or Kubernetes-in-Docker, you may want to set this to `false`. | No | `true` | `false` |

The demo scripts determine whether to use the `kubectl` or `oc` CLI
based on your `PLATFORM` environment variable configuration.

**Note**: if you are using a private Docker registry, you will also need to set:
```
export DOCKER_USERNAME=<your-username>
export DOCKER_PASSWORD=<your-password>
export DOCKER_EMAIL=<your-email>
```

Once you have:
- Reviewed the [requirements](#requirements) to ensure your Conjur / DAP server is set up correctly
- Logged into your Kubernetes cluster via the local command line
- Set your local environment as defined above

Run `./start` from the root directory of this repository to execute the numbered
scripts and step through the process of deploying test apps.

## Contributing

We welcome contributions of all kinds to this repository. For instructions on how to get started and descriptions of our development workflows, please see our [contributing
guide][contrib].

[contrib]: https://github.com/cyberark/conjur/blob/master/CONTRIBUTING.md
