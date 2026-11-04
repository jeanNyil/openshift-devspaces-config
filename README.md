# OpenShift Dev Spaces configuration

Shared Java 21 and Apache Camel workspace configuration for the personal
OpenShift Dev Spaces lab. The reusable parent is
[`devfiles/java-camel/devfile.yaml`](devfiles/java-camel/devfile.yaml).

## What the parent provides

- Container `tools`, using `quay.io/jnyilimbibi/devspaces-extended-udi:3.31`
  with project sources mounted.
- Java 21 selection and the existing JVM proxy and `/home/user` settings.
- Persistent XDG paths: `/home/user/.config` and `/home/user/.local/share`.
- Memory request and limit of `8Gi`, CPU request of `100m`, and CPU limit of
  `4000m`.
- PostStart commands `install-camel-cli-and-k8s-plugin` and `sync-agent-home`,
  in that order. Their command IDs and container name are stable interfaces
  that child devfiles can reference.

The Camel command installs the CLI and Kubernetes plugin through the custom
image's JBang tooling. The agent-home command clones the private
`jeanNyil/agent-home` repository on first start, pulls with `--ff-only` on later
starts, and runs its bootstrap to link global configuration, rules and skills
into the user home. Clone or pull errors fail the sync command.

## Prerequisites

This parent targets the existing Dev Spaces 3.31 lab and its custom extended UDI.
It is personal configuration: users need access to the private agent-home
repository through the workspace's GitHub credentials. Public access to this
parent does not grant access to agent-home.

The cluster must persist `/home/user` for the configuration and authentication
state to survive restarts. The tools image must provide JBang at
`/home/tooling/.local/bin/jbang`, Git, and the other tools used by the commands.
The workspace needs network access to GitHub and the Camel/JBang download
endpoints.

OpenCode binary injection and its MaaS API key are managed separately by the
cluster AI registry and Dev Spaces dashboard. This parent supplies the XDG
settings and agent-home sync; it does not install the binary or provision keys.
No credentials or private agent configuration are stored in this repository.

## Use the parent

Keep a `devfile.yaml` in each project and replace `<parent-commit-sha>` with a
published commit from this repository:

```yaml
schemaVersion: 2.3.0
metadata:
  name: my-project
parent:
  uri: https://raw.githubusercontent.com/jeanNyil/openshift-devspaces-config/<parent-commit-sha>/devfiles/java-camel/devfile.yaml
```

Pin the full commit SHA so each project adopts parent changes deliberately.
The URL must be readable without authentication before workspace startup.
See the [Devfile parent reference documentation](https://devfile.io/docs/2.3.0/referring-to-a-parent-devfile).

Project-specific commands belong in the child's top-level `commands` list.
Give them unique IDs and use `exec.component: tools` to run them in the inherited
container. For project-relative commands, use `workingDir: ${PROJECT_SOURCE}`.
Do not duplicate the inherited components or startup bindings.

## Override resources and environment variables in a child

The parent's defaults are 8Gi memory, a 4-CPU limit, and Java 21. This example
overrides the inherited container under `parent` with smaller resources and
selects Java 17:

```yaml
schemaVersion: 2.3.0
metadata:
  name: smaller-project
parent:
  uri: https://raw.githubusercontent.com/jeanNyil/openshift-devspaces-config/<parent-commit-sha>/devfiles/java-camel/devfile.yaml
  components:
    - name: tools
      container:
        memoryRequest: 4Gi
        memoryLimit: 4Gi
        cpuRequest: 100m
        cpuLimit: 2000m
        env:
          - name: USE_JAVA21
            value: "false"
```

Put overrides under `parent.components`; defining another top-level `tools`
component would duplicate the inherited component. Environment entries merge
by name, so this replaces the inherited `USE_JAVA21: "true"` value. Other
inherited environment settings, including `JAVA_TOOL_OPTIONS` and the XDG paths,
and both postStart bindings are retained. See the
[Devfile API reference](https://devfile.io/docs/2.3.0/devfile-schema).

The current `3.31` image checks `USE_JAVA8`, `USE_JAVA11`, and `USE_JAVA21`; it
does not consume `USE_JAVA17`. Java 17 is the fallback when none of those
selectors is `"true"`. With no Java 8 or Java 11 selector enabled, overriding
`USE_JAVA21` to `"false"` selects Java 17. After starting or restarting a workspace
with this child, verify the selected JDK with `java -version`.
