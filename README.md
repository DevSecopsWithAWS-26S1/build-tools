# RoboShop Build Tools and Dependency Repositories

This guide explains how RoboShop applications are prepared for deployment, where their dependencies come from, and how to route those downloads through a private repository.

It is written for people new to builds and package repositories. The setup guides in the sibling [`roboshop`](../roboshop/) directory show how to deploy and run each service; this guide focuses on preparing the software and obtaining its dependencies.

## 1. Build, package, and repository: the basic ideas

- A **build tool** prepares application code. It may compile code into an executable or JAR, or install the libraries an interpreted application needs.
- A **dependency** is software that an application uses but does not contain itself, such as a Java library or a Node.js package.
- A **package registry/repository** stores packages and serves them to tools such as npm, pip, Maven, and `dnf`.
- An **artifact** is the output prepared for deployment, for example a Go executable or Java JAR.

Most dependency tools download packages from public registries by default. A private repository manager (for example Nexus Repository or JFrog Artifactory) can proxy and cache public packages, host packages belonging to your organization, and provide one controlled endpoint for builds. A proxy downloads from an upstream registry when needed; a hosted repository stores packages your organization publishes; a group/virtual repository presents several repositories through one URL.

## 2. RoboShop build architectures

“Architecture” here means the language and preparation/build flow used by each application. It does not mean CPU architecture such as `x86_64` or `arm64`.

| Service(s) | Language and runtime | Dependency/build definition | Preparation in the setup guides | Result |
|---|---|---|---|---|
| Catalogue, User, Cart | JavaScript on Node.js | `package.json` | `npm install` | Installs packages into `node_modules`; Node.js runs `server.js`. No separate compile step. |
| Shipping | Java on the JVM | `pom.xml` | `mvn clean package` | Maven compiles and packages `target/shipping-1.0.jar`; Java runs the JAR. |
| Payment | Python | `requirements.txt` | `pip3 install -r requirements.txt` | Installs Python packages; uWSGI runs the app using `payment.ini`. No separate application compile step. |
| Dispatch | Go | `main.go`; the supplied archive does not include `go.mod` | `go mod init dispatch`, `go mod tidy`, then `go build` | Compiles a `dispatch` executable. |
| Frontend | Static HTML, CSS, JavaScript, and images | No package manifest in the supplied archive | Extract files into Nginx's web root | Nginx serves the files; there is no application build step. |

The database and messaging components (MongoDB, Redis, MySQL, RabbitMQ) are installed as server packages in these deployment instructions. They are not built from application source in this repository.

### Repository layout

Each application source folder is located directly under the repository root:

```text
catalogue/
cart/
dispatch/
payment/
shipping/
user/
build-tools/
```

### Node.js services: Catalogue, User, and Cart

`package.json` names the Node.js packages the application needs. Run npm from the directory containing that file:

```shell
npm install
node server.js
```

In the deployment guides, systemd starts `server.js` after the packages have been installed. The supplied archives have `package.json` files but do not include npm lockfiles, so `npm install` can select newer compatible package versions over time. For repeatable production builds, keep a reviewed `package-lock.json` with the source and use `npm ci`.

### Java service: Shipping

Maven reads `pom.xml`, which declares the project, Java dependencies, and packaging configuration. From the directory containing `pom.xml`:

```shell
mvn clean package
```

Maven resolves the parent POM and dependencies, compiles the service, and writes a JAR under `target/`. This RoboShop guide renames the JAR to `shipping.jar` for systemd to run.

### Python service: Payment

`requirements.txt` lists Python packages. From the directory containing it:

```shell
pip3 install -r requirements.txt
```

The app is started through uWSGI, configured by `payment.ini`. The supplied requirements are not pinned to exact versions. For reproducible deployments, pin and review versions (and consider a generated lock/constraints file) in the source repository.

### Go service: Dispatch

The supplied archive has Go source but no `go.mod`. The deployment guide creates a module and resolves imports before compiling:

```shell
go mod init dispatch
go mod tidy
go build
```

`go mod tidy` discovers imported modules and records them in `go.mod` and `go.sum`; `go build` produces the executable. Because those module files are generated during deployment in the current guide, a fresh build may resolve different versions later. For repeatability, generate and review `go.mod` and `go.sum` with the source, then build from those committed files.

### Frontend and server packages

The frontend is static content, so it is copied into Nginx's document root rather than compiled. Nginx and the other operating-system components are installed with `dnf`; the enabled OS repositories and module streams provide those packages.

## 3. Where the source and dependencies are downloaded from

There are two separate downloads in a deployment:

1. **RoboShop application archives**: the service ZIP files are downloaded from the `downloads/` directory of the `DevSecopsWithAWS-26S1/ibm-roboshop-setup` GitHub repository (the guides use `raw.githubusercontent.com` URLs).
2. **Third-party dependencies and server packages**: each package tool contacts its configured registry/repository. These are not downloaded from the RoboShop GitHub repository.

Default public dependency sources when no private registry is configured:

| Installer | What it downloads | Default public source |
|---|---|---|
| npm | Node.js packages declared in `package.json` | npm registry: `https://registry.npmjs.org/` |
| pip | Python packages declared in `requirements.txt` | Python Package Index (PyPI): `https://pypi.org/simple` |
| Maven | Java dependencies and parent POMs declared/referenced by `pom.xml` | Maven Central: `https://repo.maven.apache.org/maven2/` |
| Go | Go modules found from source imports | Go module proxy: `https://proxy.golang.org`; integrity checks use `https://sum.golang.org` by default |
| `dnf` | Node.js, Java/Maven, Python, Go, Nginx, databases, and other RPM packages | Enabled Rocky Linux/RHEL repositories and module streams, plus any explicitly configured third-party RPM repositories |

In the current guides, MongoDB is configured from `repo.mongodb.org`. RabbitMQ and Erlang are configured from `yum1.novemberain.com` and `yum2.novemberain.com`. Other `dnf install` commands use whatever OS repositories are enabled on the server. An installed package does not necessarily come from the same source as the application's libraries.

The setup docs configure MongoDB and RabbitMQ RPM repositories with `gpgcheck=0` for this lab. Do not copy that setting into a production repository configuration without a security review; prefer signed packages and signature verification.

## 4. Using a private repository

### Recommended setup

Ask your platform/repository administrator to provide a TLS-protected repository manager and these endpoints:

1. A **proxy** for each public ecosystem you use (npm, PyPI, Maven Central, Go modules, and/or RPM repositories).
2. A **hosted** repository for private packages your team publishes.
3. A **group/virtual** repository that combines the needed proxy and hosted repositories, when supported.
4. A read-only identity for build/download jobs, and a separate publish identity for releases.

Then configure each tool to use the group/virtual endpoint instead of the public URL. Use your organization's real URLs and credential-management process in place of the examples below. Never commit passwords or tokens to source control.

### npm

Create a project `.npmrc` beside `package.json`:

```ini
registry=https://packages.example.com/repository/npm-group/
//packages.example.com/repository/npm-group/:_authToken=${NPM_TOKEN}
always-auth=true
```

Provide `NPM_TOKEN` to the build process through its secret store. Use a read-only token for installs. If private packages are scoped, the repository manager can serve both public and private packages through the configured registry.

### Python / pip

Set the private PyPI-compatible group as the package index. For example, configure these environment variables in the build environment:

```text
PIP_INDEX_URL=https://packages.example.com/repository/pypi-group/simple
PIP_EXTRA_INDEX_URL=
```

If authentication is required, use the repository manager's approved pip credential configuration or a short-lived secret provided by CI; do not put credentials in a checked-in requirements file. Prefer a single trusted group index rather than adding an untrusted public index as an extra index, which can create package-confusion risks.

### Maven

Put Maven repository credentials in the user's `~/.m2/settings.xml` (or a protected CI settings file), not in `pom.xml`. A mirror directs Maven Central and other Maven repository requests through the private group:

```xml
<settings>
  <servers>
    <server>
      <id>private-maven</id>
      <username>${env.MAVEN_REPO_USER}</username>
      <password>${env.MAVEN_REPO_PASSWORD}</password>
    </server>
  </servers>
  <mirrors>
    <mirror>
      <id>private-maven</id>
      <mirrorOf>*</mirrorOf>
      <url>https://packages.example.com/repository/maven-group/</url>
    </mirror>
  </mirrors>
</settings>
```

Set `MAVEN_REPO_USER` and `MAVEN_REPO_PASSWORD` through a secret manager or CI secret variables. Maven uses the same ID to associate the mirror with its credentials. The example routes all Maven repositories through the private group; make sure that group contains every approved upstream and internal repository the build needs.

### Go

If the repository manager exposes a Go module proxy that serves both approved public modules and your private modules, configure Go to use that proxy:

```text
GOPROXY=https://packages.example.com/repository/go-group/
GONOPROXY=none
```

For private modules hosted under your own domain, also set a matching pattern, for example:

```text
GOPRIVATE=git.example.com/company/*
```

`GONOPROXY=none` prevents Go from bypassing the private proxy for paths matched by `GOPRIVATE`. Do not add a `,direct` fallback unless your organization has explicitly approved direct downloads for private module paths. If private modules should instead be fetched directly from your Git server, omit `GONOPROXY=none` and configure Git credentials with an approved credential helper or CI secrets; public modules can still use the configured public/private proxy. Confirm which flow your repository manager supports. `GOPRIVATE` also tells Go not to send matching paths to the public checksum database. Do not disable Go's integrity checks globally to work around repository configuration problems.

### `dnf` RPM packages

The repository administrator should mirror the approved Rocky/RHEL and third-party RPM sources and configure signing keys. On each server, add a repo file such as `/etc/yum.repos.d/company.repo`:

```ini
[company-rpms]
name=Company approved RPM repository
baseurl=https://packages.example.com/repository/rpm-group/
enabled=1
gpgcheck=1
gpgkey=https://packages.example.com/keys/RPM-GPG-KEY-company
```

Use the actual RPM repository URL and signing key supplied by your administrator. Configure authentication using a protected method supported by your repository manager; do not put secrets in a world-readable repo file. After the repo is configured, refresh metadata and inspect package origin:

```shell
dnf clean all
dnf makecache
dnf repolist
dnf info nodejs
```

If a vendor requires its own repository or signing key, have the repository administrator mirror and verify that source rather than silently replacing it with an unrelated RPM.

### Verify the configuration

- Check the package tool's effective registry/repository configuration before a build.
- Run a normal dependency install and confirm its logs show the private endpoint.
- Confirm that private packages can be downloaded and public dependencies are available through the approved proxy.
- Keep credentials out of logs, command-line arguments, source files, and build artifacts.
- Ensure CI agents and deployment servers can resolve and reach the repository over TLS.
- Grant only read access to build jobs; restrict package publishing to release identities.

## 5. References in the deployment repository

- Overall service architecture and build summary: [`roboshop/readme.MD`](../roboshop/readme.MD)
- Application setup guides: [`roboshop/`](../roboshop/)
- Application archives: [`roboshop/downloads/`](../roboshop/downloads/)
