---
id: redhat-openshift
title: "Red Hat OpenShift"
description: "Deploy Camunda Platform 8 Self-Managed on Red Hat OpenShift"
---

Camunda Platform 8 can be deployed using Helm on Red Hat OpenShift with proper configurations. The primarily difference from [general Helm deployment guide](../deployment.md) is related to the Security Context Constraints (SCCs) you have in your cluster.

## Compatibility

We test against the following OpenShift versions and guarantee compatibility with:

| OpenShift version | Supported          |
| ----------------- | ------------------ |
| 4.10.x            | :white_check_mark: |

Any version not explicitly marked in the table above is not tested, and we cannot guarantee compatibility.

## Pitfalls to avoid

For general deployment pitfalls, visit the [deployment troubleshooting guide](../../troubleshooting.md).

### Security Context Constraints (SCCs)

Much like how roles control the permissions of users, Security Context Constraints (SCCs) are a way to control the permissions of the applications deployed, both at the pod and container level. It's generally recommended deploying your application with the most restricted SCCs possible. If you're not familiar with security context constraints, refer to the [OpenShift documentation](https://docs.openshift.com/container-platform/latest/authentication/managing-security-context-constraints.html).

#### Permissive SCCs

Out of the box, if you deploy Camunda Platform 8 (and related infrastructure) with a permissive SCCs, there is nothing particular for you to configure. Here, a permissive SCCs refers to one where the strategy for `RunAsUser` is defined as `RunAsAny` (including root).

#### Non-root SCCs

If you wish to deploy Camunda Platform 8 but restrict the applications from running as root (e.g. the `nonroot` built-in SCCs), you will need to configure each pod and container to run as a non-root user. For example, when deploying Zeebe using a stateful set, you would add the following, replacing `1000` with the user ID you wish to use:

```yaml
spec:
  template:
    spec:
      securityContext:
        runAsUser: 1000
      containers:
        securityContext:
          runAsUser: 1000
```

:::note
As the container user in OpenShift is always part of the root group, it's not necessary to define a `fsGroup` for any Camunda Platform 8 applications pod security context.
:::

This is necessary for all Camunda Platform 8 applications, as well as related ones (e.g. Keycloak, PostgreSQL, etc.). This is notably crucial for stateful applications which will write to persistent volumes, but it's also generally a good idea to set.

#### Restrictive SCCs

The following is the most restrictive SCCs you can use to deploy Camunda Platform 8. Note that this is, in OpenShift 4.10, equivalent to the built-in `restricted` SCCs (which is the default SCCs).

```yaml
Allow Privileged: false
Default Add Capabilities: <none>
Required Drop Capabilities: KILL,MKNOD,SYS_CHROOT,SETUID,SETGID
Allowed Capabilities: <none>
Allowed Seccomp Profiles: <none>
Allowed Volume Types: configMap,downwardAPI,emptyDir,persistentVolumeClaim,projected,secret
Allow Host Network: false
Allow Host Ports: false
Allow Host PID: false
Allow Host IPC: false
Read Only Root Filesystem: false
Run As User Strategy: MustRunAsRange
SELinux Context Strategy: MustRunAs
FSGroup Strategy: MustRunAs
Supplemental Groups Strategy: RunAsAny
```

When using this, you must take care not to specify _any_ `runAsUser` or `fsGroup` in either the pod or container security context. Instead, let OpenShift assign arbitrary IDs.

:::note
If you are providing the ID ranges yourself, you can configure the `runAsUser` and `fsGroup` values yourself as well.
:::

The Camunda Platform Helm chart can be deployed to OpenShift with a few modifications, primarily revolving around your desired security context constraints. You can find out more about this in the next section.

## Deployment

As discussed in the previous section, you need to configure the pod and container security contexts based on your desired security context constraints (SCCs).

The `Elasticsearch`, `Keycloak`, and `PostgreSQL` charts all specify default non-root users for security purposes. To deploy these charts through the Camunda Platform Helm chart, these default values must be removed. Unfortunately, due to a [longstanding bug in Helm](https://github.com/helm/helm/issues/9136) affecting all Helm versions from 3.2.0 and greater, this makes the installation of the chart (when deploying any of these sub-charts) more complex.

Note that this is only an issue if you are deploying `Elasticsearch`, `Keycloak` (via `Identity`), or `PostgreSQL` (via `Keycloak`). If you are not deploying these, or not via the `camunda-platform` chart, or you are using [permissive SCCs](#permissive-sccs), this issue does not affect your deployment.

:::note
This also affects installations done through the OpenShift console, as it still uses Helm under the hood.
:::

### Permissive SCCs

To use permissive SCCs, install the charts as they are. Follow the [general Helm deployment guide](../deployment.md).

### Restrictive SCCs

To use more restrictive SCCs, configure the following minimum set of values for the various applications. The recommendations outlined in the sections are relevant here as well. As the Camunda Platform 8 applications do not define a pod or security context, follow these recommendations, or simply omit defining any.

If you are deploying with SCCs where `RunAsUser` is `MustRunAsRange` (e.g. the default `restricted` SCCs), and are deploying at least one of `Elasticsearch`, `Keycloak`, or `PostgreSQL`, it's necessary to unset the default security context of these charts. If this does not apply to you, you can stop here.

Now this depends on which Helm version you use: `3.1.3 and lower`, or `3.2.0 and greater` (i.e. one affected by [Helm's nested sub-charts](https://github.com/helm/helm/issues/9136)). Find out your Helm version by running the following:

```shell
helm version --short

v3.8.1+g5cb9af4
```

#### Helm 3.1.3 or lower

If you're running on Helm 3.0.0 up to 3.1.3, you need to add these values to your `values.yaml` file, or save them to a new file locally, e.g. `openshift.yaml`:

:::note
These values are also available in the [Camunda Platform Helm chart repository](https://github.com/camunda/camunda-platform-helm/blob/main/openshift/values.yaml).
:::

```yaml
# omit this section if elasticsearch.enabled is false
elasticsearch:
  securityContext:
    runAsUser: null
  sysctlInitContainer:
    enabled: false
  podSecurityContext:
    fsGroup: null
    runAsUser: null

# omit this section if identity.enabled is false
identity:
  # omit this section if identity.keycloak.enabled is false
  keycloak:
    containerSecurityContext:
      runAsUser: null
    podSecurityContext:
      fsGroup: null
      runAsUser: null
    postgresql:
      # omit this section if identity.keycloak.postgresql.primary.enabled is false
      primary:
        containerSecurityContext:
          runAsUser: null
        podSecurityContext:
          fsGroup: null
          runAsUser: null
      # omit this section if identity.keycloak.postgresql.readReplicas.enabled is false
      readReplicas:
        containerSecurityContext:
          runAsUser: null
        podSecurityContext:
          fsGroup: null
          runAsUser: null
      # omit this section if identity.keycloak.postgresql.metrics.enabled is false
      metrics:
        containerSecurityContext:
          runAsUser: null
        podSecurityContext:
          fsGroup: null
          runAsUser: null
```

When installing the chart, run the following:

```shell
helm install <RELEASE_NAME> camunda/camunda-platform --skip-crds -f values.yaml -f openshift.yaml
```

#### Helm 3.2.0 and greater

If you must deploy using Helm 3.2.0 or greater, you have two options. One is to use a SCCs which defines the `RunAsUser` strategy to be at least `RunAsAny`. If that's not possible, then you need to make use of [a post-renderer](https://helm.sh/docs/topics/advanced/#post-rendering). This workaround is also described in detail in the [Helm chart repository](https://github.com/camunda/camunda-platform-helm/tree/main/openshift#using-a-post-renderer).

:::warning
If using a post-renderer, you **must** use the post-renderer whenever you are upgrading your release, not only during the initial installation. If you do not, the default values will be used again, which will prevent some services from starting.
:::

While you can use your preferred `post-renderer`, [we provide one](https://github.com/camunda/camunda-platform-helm/blob/main/openshift/patch.sh) which requires only `bash` and `sed` to be available locally:

```bash
#!/bin/bash -eu
# Expected usage is as an Helm post renderer.
# Example usage:
#   helm install my-release camunda/camunda-platform --post-renderer ./patch.sh
#
# This script is a Helm chart post-renderer for users on Helm 3.2.0 and greater. It allows removing default
# values set in sub-charts/dependencies, something which should be possible but is currently not working.
# See this issue for more: https://github.com/helm/helm/issues/9136
#
# The result of patching the rendered Helm templates is printed out to STDOUT. Any other logging from the
# script is thus sent to STDERR.
#
# Note to contributors: this post-renderer is used in the integration tests, so make sure that it can be used
# from any working directory.

set -o pipefail

# Perform two passes: once for single quotes, once for double quotes, as it's not specified that string values are
# always output with single or double quotes
sed -e "s/'@@null@@'/null/g" -e 's/"@@null@@"/null/g'
```

You also need to use a custom values file, where instead of using `null` as a value to unset default values, you use a special marker value which will be removed by the post-renderer.

Copy these values to your values file or save them as a separate file, e.g. `openshift.yaml`:

:::note
These values are also available in the [Camunda Platform Helm chart repository](https://github.com/camunda/camunda-platform-helm/blob/main/openshift/values-patch.yaml).
:::

```yaml
# omit this section if elasticsearch.enabled is false
elasticsearch:
  securityContext:
    runAsUser: "@@null@@"
  sysctlInitContainer:
    enabled: false
  podSecurityContext:
    fsGroup: "@@null@@"
    runAsUser: "@@null@@"

# omit this section if identity.enabled is false
identity:
  # omit this section if identity.keycloak.enabled is false
  keycloak:
    containerSecurityContext:
      runAsUser: "@@null@@"
    podSecurityContext:
      fsGroup: "@@null@@"
      runAsUser: "@@null@@"
    postgresql:
      # omit this section if identity.keycloak.postgresql.primary.enabled is false
      primary:
        containerSecurityContext:
          runAsUser: "@@null@@"
        podSecurityContext:
          fsGroup: "@@null@@"
          runAsUser: "@@null@@"
      # omit this section if identity.keycloak.postgresql.readReplicas.enabled is false
      readReplicas:
        containerSecurityContext:
          runAsUser: "@@null@@"
        podSecurityContext:
          fsGroup: "@@null@@"
          runAsUser: "@@null@@"
      # omit this section if identity.keycloak.postgresql.metrics.enabled is false
      metrics:
        containerSecurityContext:
          runAsUser: "@@null@@"
        podSecurityContext:
          fsGroup: "@@null@@"
          runAsUser: "@@null@@"
```

Now, when installing the chart, you can do so by running the following:

```shell
helm install <RELEASE_NAME> camunda/camunda-platform --skip-crds \
    -f values.yaml -f openshift.yaml --post-renderer ./patch.sh
```
