# OpenShift Dev Spaces configuration

Shared Java 21 and Apache Camel workspace configuration for the personal
OpenShift Dev Spaces lab. The reusable parent is
[`devfiles/java-camel/devfile.yaml`](devfiles/java-camel/devfile.yaml).

## What the parent provides

- Container `tools`, using `quay.io/jnyilimbibi/devspaces-extended-udi:3.31`
  with project sources mounted.
- Java 21 selection and the existing JVM proxy and `/home/user` settings.
- Persistent XDG paths: `/home/user/.config` and `/home/user/.local/share`.
- Explicit tooling PATH with the selected JDK first and local system binaries
  ahead of system alternatives.
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

## AI injection and tooling PATH

The Dev Spaces 3.31 dashboard prepends `/injected-tools/bin` to a PATH explicitly
set on the target container. Without one, it supplies a generic editor PATH,
which can omit the selected JDK and other image-specific tools. See the
[dashboard PATH handling](https://github.com/eclipse-che/che-dashboard/blob/7.123.0/packages/dashboard-frontend/src/services/helpers/aiTools.ts).

Maven can remain available through `/usr/bin/mvn` and use `$JAVA_HOME/bin/java`
even when `java` is absent from terminal PATH. System alternatives can meanwhile
select Java 17 for `javac` and `jar` despite `JAVA_HOME` selecting Java 21.
Go and Node/npm also remain available through `/usr/bin`; loss of their other
image directories does not mean their commands disappeared.

The parent restores the tool-specific directories from extended UDI `3.31`
image config at digest
`sha256:101e0db59101f7b4edd40715dd6f8682b0c77092b426cda078f6c71e21f677f6`.
These retain image order, with trailing slashes normalized and duplicate and
empty entries removed. Standard directories come last in this order:

```text
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

This keeps `/usr/local/bin` ahead of `/usr/bin`, preserving the working Python
3.11 and Bubblewrap 0.11 selections; copying the image PATH verbatim would select
Python 3.9 and Bubblewrap 0.6 instead. The JDK directory remains
`/home/user/.java/current/bin`, so the existing JDK selectors also select the
matching compiler and tools. The dashboard supplies its own injected-tools
prefix. Do not use `$PATH` or `$(PATH)` self-references in the devfile's literal
environment value.

Interactive shell startup can restore some tools that remain absent from raw
container execution. JBang/Camel, the IDE's `code-oss` remote CLI, environment
modules, and optional .NET tools can add paths during shell or IDE initialization.
Keep that initialization intact; do not hardcode its installation-specific paths
or copy the entire terminal PATH into the devfile. Repeated terminal entries are
not the cause of the missing Java command.

There is a separate Dev Spaces 3.31 AI-injection limitation: the dashboard finds
target containers before resolving parents. A container defined only in the
parent may therefore receive neither the AI-tools mount nor its PATH setup,
even when its injector completes. An explicit PATH in this parent does not fix
that discovery limitation. On this release, projects using dashboard AI injection
should provide a complete standalone devfile with the tooling container directly
present. Reconsider inheritance after a newer release passes an actual workspace
smoke test. Parent inheritance remains valid Devfile syntax.

When upgrading the image, inspect its config, apply the same directory ordering
rules, and update the PATH and reference digest together. With `skopeo` and `jq`
installed, inspect the reference without pulling image layers:

```sh
skopeo inspect --config \
  docker://quay.io/jnyilimbibi/devspaces-extended-udi@sha256:101e0db59101f7b4edd40715dd6f8682b0c77092b426cda078f6c71e21f677f6 \
  | jq -r '.config.Env[] | select(startswith("PATH="))'
```

Check both noninteractive execution and the actual IDE terminal:

```sh
bash --noprofile --norc -c 'command -v java javac jar mvn jbang camel podman kubedock pip3 virtualenv go node npm oc kubectl'
java -version
javac -version
jar --version
mvn --version
python --version
bwrap --version
command -v code-oss                      # actual IDE terminal
```

For this image's Java 21 setup, Java, `javac`, and `jar` must select 21; Maven must
continue using Java 21; Python/Bubblewrap must retain their working 3.11/0.11
selections. Check JBang, Camel, Podman, Kubedock, Python utilities, Go, Node/npm,
OpenShift/container clients, and any selected injected AI tool too. After
OpenCode injection, verify `opencode --version`. Compare executable inventories
before and after PATH changes to detect missing commands. Lookup and version
checks do not exercise every tool's full functionality.

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
