---
slug: integrate-with-harness-cd
id: 3hl608ktdrht
type: challenge
title: integrate-with-harness-cd
teaser: Deliver software faster, with visibility and control.
notes:
- type: text
  contents: |
    Until now we saw how do CI from your laptops. As part of this challenge we will explore how to integrate your local CI with Harness CD(Free Tier) to perform Continuous Deployments(CD).
    ## Objectives

    In this challenge, this is what you'll learn:
    - [x] Register and Create an account https://harness.io
    - [x] Configure Harness CD to use your GitHub repository
    - [x] Configure Harness CD to user your Container Registry like DockerHub, Quay.io etc.,
    - [x] Integrate Drone CI extension with Harness CD

    Time: ~30 mins
tabs:
- title: Terminal
  type: terminal
  hostname: kubernetes-vm
- title: Editor
  type: code
  hostname: kubernetes-vm
  path: /root/repos/drone-ci-101
- title: Harness CD
  type: website
  path: /
  url: https://app.harness.io/ng
  new_window: true
difficulty: basic
timelimit: 1800
---

👋 Introduction
===============

__TODO__: Needs update

Eliminate scripting and manual deployments with declarative GitOps and powerful, easy-to-use pipelines. Empower your teams to deliver new features, faster – with AI/ML for automated canary and blue/green deployments, advanced verification, and intelligent rollback. Check all the boxes with enterprise-grade security, governance, and granular control powered by the Open Policy Agent.

Pre-Requisites
===============

Before we get started with the exercises of this chapter, you need to have the following,

- A GitHub Account and  [Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) with __repo__ access enabled.
- An account with [Docker Hub](https://hub.docker.com) or [Quay.io](https://quay.io).
- Forked the project <https://github.com/harness-apps/drone-ci-harness-cd-demo> under your GitHub account.

Register with Harness
=====================

__TODO__: Needs update

Harness provides __Free Tier__ with all its platforms that can be used to perform PoCs, tests etc.,

Let us get started by registering for a free account, click the Harness CD tab to get an account created for yourself.

Create Project
==============

On your Harness CD account navigate to `Home --> Projects` and click "Create Project" to create a new [project](https://docs.harness.io/article/hv2758ro4e#organizations_and_projects).

![Harness Projects](../assets/harness_cd_projects_home.png)

You can give any name you want, but rest of the instructions we will refer to the project as `drone-ci-extension-cd`.  Click "Save and Continue" to leave the next screen to defaults.

![Harness New Project](../assets/harness_cd_new_project.png)

On the __Harness Modules__ screen choose "Continuous Delivery" and "Start Free Plan".

__NOTE:__ Click `x` to cancel pipeline creation as we will be creating one as part of the upcoming sections.

If all went well we should land on the __Deployments__ dashboard,

![Harness New Project](../assets/harness_cd_home.png)

🔧 Install Harness Delegate
===========================

The Harness Delegate is a service you run in your local network or VPC to connect all of your artifact, infrastructure, collaboration, verification and other providers with the Harness Manager. You can read more about various types of Delegates that Harness supports [here](https://docs.harness.io/article/h9tkwmkrm7-delegate-installation).

Navigate to `__Project Setup__ --> __Delegate__`,

![Create Delegate](../assets/harness_cd_new_delegate.png)

Click the __Create Delegate__ to create a new delegate. On the delegate type screen choose __Kubernetes__ and click __Continue__.

![K8s Delegate](../assets/harness_cd_k8s_delegate.png)

Give a name to the delegate like __my-harness-delegate__, select the delegate size to be __Small__ and installer type to be __Kubernetes__. Leave rest to defaults and click __Continue__.

![Kubernetes Delegate Options](../assets/harness_cd_k8s_delegate_options.png)

On the next screen click the __Copy to Clipboard__ to copy the Kubernetes manifest to clipboard.

Go to the Editor tab and create new file called `harness-delegate.yml` and paste the copied content on to the file. Click __Save__ to save the file.

Get back to the Harness Window and click __Continue__ to finish the wizard.

On the __Terminal__ run the following command,

```shell
kubectl apply -f "$TUTORIAL_HOME/harness-delegate.yml"
```

Wait for the harness delegate statefulset __my-harness-delegate__ to be up and running,

```shell
kubectl rollout status -n harness-delegate-ng  statefulset my-harness-delegate --timeout=180s
```

> __NOTE__: It will take few minutes the delegate to be ready and connected to your Harness Account.

If all went well and successful you should the delegate ready on your Harness account,

![Delegates Ready](../assets/harness_cd_delegate_ready.png)

㊙️ Secrets
==========

Harness includes built-in Secrets Management to store your encrypted secrets, such as access keys, and use them in your Harness account.  Harness integrates with all popular Secrets Managers. For more information check the [Secret Management](https://docs.harness.io/article/hv2758ro4e#secrets_management) online.

As part of the upcoming exercises you need to connect to Docker Hub, GitHub and Kubernetes Clusters. To connect to them transparently you need to save your Harness account with the respective credentials. For this challenge we will store the credentials as __Encrypted__ __Text__ secrets.

Before we go further into creation of secrets ensure you have the following details with you,

- Docker Registry Username and Password
- GitHub Personal Access Toke
- Kubeconfig of the lab cluster

Navigate to `__Project Setup__ --> __Secrets__`.

🐳 Docker Registry
------------------

![Project Secrets](../assets/harness_cd_project_secrets.png)

Click on the ![Add Text Secret](../assets/add_harness_cd_project_text_secret.png) to start adding new secret,

Fill the details of the secret as shown and click save to save the secret.

[!Docker Registry Secret](../assets/harness_cd_project_docker_reg_secret.png)

> __NOTE:__
>
> Though we can use encryption for usernames as well but for this challenge we will store the username(s) as plain text.

🐙 GitHub PAT Connector
-----------------------


As did with previous section click on the ![Add Text Secret](../assets/add_harness_cd_project_text_secret.png) to start adding new secret,

Fill the details of the secret as shown and click __Save__ to save the secret.

[!GitHub PAT](../assets/harness_cd_project_github_pat_secret.png)

🐳 Kubernetes Secret
--------------------

The Kubernetes Connector that we will configure in upcoming section requires  one for each of the following,

- Client Key
- Client Certificate

### Kubeconfig ###

On the __Terminal__ tab navigate to the tutorial home folder,

```shell
cd "$TUTORIAL_HOME"
```

As prompted let us set some environment variables,

```shell
direnv allow .
```

From your lab environment get the kubeconfig,

```shell
cat "$KUBECONFIG" | grep -iA1 server
```

As you notice the Kube API server is set to `127.0.0.1` which is fine as long as we connect to Kubernetes cluster from within the lab __Terminal__. To make it accessible form outside world from our Harness CD environment we need to update it as shown,

```shell
sed "s|127.0.0.1|kubernetes-vm.${_SANDBOX_ID}.instruqt.io|" "$KUBECONFIG" > "$TUTORIAL_HOME/.kubeconfig.external"
chmod 0700 "$KUBECONFIG" > "$TUTORIAL_HOME/.kubeconfig.external"
```

### Client Certificate ###

Open the `$TUTORIAL_HOME/.kubeconfig.external` using the editor tab and copy the __value__ of __client-certificate-data__ (should be approximately line #18),

![Copy Client Certificate](../assets/harness_cd_project_k8s_client_cert_secret_value.png)

As did earlier add the __Client Certificate__ by clicking ![Add Text Secret](../assets/add_harness_cd_project_text_secret.png),

Fill the details of the secret as shown and click __Save__ to save the secret.

[!Kubernetes Client Cert](../assets/harness_cd_project_k8s_client_cert_secret.png)

### Client Key ###

On the opened __Editor__ tab copy the __value__ of __client-key-data__ (should be approximately line #19) from the file `$TUTORIAL_HOME/.kubeconfig.external`.

![Copy Client Certificate](../assets/harness_cd_project_k8s_client_key_secret_value.png)

As did earlier add the __Client Key__ by clicking ![Add Text Secret](../assets/add_harness_cd_project_text_secret.png),

Fill the details of the secret as shown and click __Save__ to save the secret.

[!Kubernetes Client Key](../assets/harness_cd_project_k8s_client_key_secret.png)

With this your project's __Secrets__ dashboard should look like,

![All Secrets](../assets/harness_cd_project_all_secrets.png)

🔌 Connectors
=============

Connectors contain the information necessary to integrate and work with 3rd party tools. Harness uses Connectors at Pipeline runtime to authenticate and perform operations with a 3rd party tool.

As part of this challenge we will configure the following three connectors,

- [Docker Registry Connector](https://docs.harness.io/article/u9bsd77g5a-docker-registry-connector-settings-reference) to make our pipelines listen to new image pushes.
- [GitHub Connector](https://docs.harness.io/article/jd77qvieuw-add-a-git-hub-connector) to pull Helm manifest sources to deploy the application to Kubernetes Cluster.
- [Kubernetes Connector](https://docs.harness.io/article/1gaud2efd4-add-a-kubernetes-cluster-connector) to connect to our lab Kubernetes cluster and deploy the Helm charts of the *hello world* application.

🐳 Docker Registry Connector
----------------------------

🐙 GitHub Connector
-------------------

🐳 Kubernetes Connector
-----------------------

🏁 Finish
=========

To learn further with Harness CD, please visit our official documentation [page](https://docs.harness.io/category/pfzgb4tg05-howto-cd).

To complete this challenge, press __Check__.