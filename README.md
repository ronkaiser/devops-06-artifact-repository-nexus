# Module 6 - Artifact Repository Manager with Nexus

**Demo Project:** Run Nexus on Droplet and Publish Artifact to Nexus

**Technologies used:** Nexus, DigitalOcean, Linux, Java, Gradle, Maven

**Project Description:**

- Install and configure Nexus from scratch on a cloud server
- Create new User on Nexus with relevant permissions
- Java Gradle Project: Build Jar & Upload to Nexus
- Java Maven Project: Build Jar & Upload to Nexus

*Module 6: Artifact Repository Manager with Nexus*

---

## Description

### What is an artifact repository?

Continuous integration produces **build artifacts**—packaged outputs that you promote toward different deployment environments. An **artifact repository** stores those artifacts in a **central place** so teams can retrieve consistent versions for deployment and for reuse as dependencies.

An **artifact** is your application (or library) built into one or more files. Common **formats** include **JAR**, **WAR**, **ZIP**, **TAR**, and others. Each format may need a repository that understands that layout; in practice you often maintain **one repository per artifact type or technology**.

### What is an artifact repository manager?

Instead of many disconnected stores—one per format or team—a **repository manager** provides **one system** that can host **many** repository types and formats. That scales well when a company builds Java, .NET, Docker images, and more.

### Public vs private repository managers

**Public** repository managers (for example Maven Central) hold libraries and frameworks you consume as dependencies. You can also publish some projects there publicly.

**Private** repository managers keep **company-internal** artifacts: you **upload** builds, **download** them for deployments, and scope access for internal usage. **Nexus Repository** is one of the most widely used options for private artifact management.

### Nexus as a repository manager

Nexus offers **open source** and **commercial** editions. You **host** your own repositories and can use **proxy** repositories as intermediaries to upstream public repos so developers fetch everything through Nexus.

Supported formats in Nexus include **Maven** (`maven2`), **NuGet**, **Docker**, and others—so Java JARs, .NET packages, and container images can live behind one manager.

### Features (why it fits CI/CD)

Repository managers must integrate with pipelines and operations. Nexus includes capabilities such as:

- **LDAP** integration for directory-driven authentication  
- **User tokens** for non-interactive authentication from tools and CI  
- **REST API** for automation (upload, download, list repositories and components, etc.)  
- **Backup and restore**  
- **Multi-format** support (ZIP, TAR, Docker, and more)  
- **Metadata tagging** for labeling artifacts  
- **Cleanup policies** to remove artifacts that match configurable rules  
- **Search** across projects and repositories  

These features matter because Nexus typically sits in your **CI/CD path**: builds publish here; deployments and other jobs consume artifacts from here.


## Prerequisites

- A [DigitalOcean](https://www.digitalocean.com/) account (the course uses signup credits where available).
- An **SSH client** on your computer (OpenSSH: `ssh`, `ssh-keygen`, `scp`).
- **Java 17**, **Gradle**, and **Maven** on your machine if you build the course examples locally. The samples target Java 17 (see `java` in [java-app/build.gradle](./java-app/build.gradle) and the compiler settings in [java-maven-app/pom.xml](./java-maven-app/pom.xml)).

**Example projects:**

- Gradle / Spring Boot: [java-app](./java-app/)
- Maven / Spring Boot: [java-maven-app](./java-maven-app/)

---

## Install and run Nexus on a cloud server

Goal: run Nexus on an Ubuntu **Droplet** you reach over SSH, then open the Nexus UI from your browser. The handout summarizes this in two parts; below expands those bullets into actionable steps. Exact package names and Nexus download URLs change over time—always confirm against [Sonatype Nexus Repository documentation](https://help.sonatype.com/).

### Part 1 — Droplet, SSH, Java, Nexus binary, `nexus` Linux user

#### 1. Create an Ubuntu server (Droplet)

Per the handout, size the Droplet at least **4 GB RAM**, **2 CPUs**, and **160 GB SSD** disk so Nexus has room for the application and artifact storage.

1. In DigitalOcean, choose **Create → Droplets**.
2. Select an **Ubuntu** image (LTS is a good default).
3. Choose the **minimum size above** (or larger for heavier use).
4. Under **Authentication**, use **SSH keys** as in prior modules.
5. Create the Droplet.

#### 2. Open SSH (port 22)

Allow **inbound TCP 22** so you can administer the server (DigitalOcean **Cloud Firewall** or equivalent). Outbound traffic is typically allowed by default for downloads and updates.

#### 3. Install Java on the server

The handout references **Java 8** for the Nexus host. Nexus 3.x **minimum Java version depends on the Nexus release** you install (newer Nexus versions often require **Java 11 or 17**). Before installing, check **Sonatype’s installation guide** for your exact Nexus version and install a matching **JDK** on the Droplet.

Example check after installation:

```bash
java -version
```

#### 4. Download and install Nexus

1. On the Droplet, download the **Nexus Repository OSS** distribution for Linux from Sonatype (see [download / installation](https://help.sonatype.com/) for current links).
2. Extract it to a conventional path such as `/opt` (exact layout may be `nexus-x.y.z` plus a `sonatype-work` data directory, depending on distribution).

Do **not** run Nexus as `root` long term.

#### 5. Create a `nexus` user and group (best practice)

Services should run as a dedicated account with **least privilege**—only what that service needs. Follow the **Sonatype installation guide** for your Nexus version: it shows how to create a **`nexus`** (or similarly named) Linux user and which directories that user should own under **`nexus-app`** and **`sonatype-work`** paths on your server.

### Part 2 — Ownership, start Nexus as `nexus`, firewall 8081, browser access

#### 6. Make `nexus` own the installation and data folders

```bash
sudo chown -R nexus:nexus /path/to/nexus-app
sudo chown -R nexus:nexus /path/to/sonatype-work
```

Use the real paths from your extraction step.

#### 7. Start Nexus as the `nexus` user

Use the startup mechanism documented for your version—often invoking `nexus`/`nexus.rc` scripts or a **systemd** unit so Nexus restarts after reboot. Example foreground-style check (paths illustrative):

```bash
sudo -u nexus /path/to/nexus/bin/nexus run
```

For production-style operation, prefer **systemd** or your distribution’s service layout per Sonatype docs.

#### 8. Open port 8081 on the firewall

Nexus serves the UI and repository endpoints on **8081** by default. Add an **inbound** rule for **TCP 8081** (limit **source** to your IP or office/VPN range when learning).

SSH on **22** alone does **not** expose the Nexus UI; without **8081** (or HTTPS on another port), browsers on your laptop cannot reach Nexus.

#### 9. Access Nexus from a browser

Visit:

```text
http://YOUR_DROPLET_PUBLIC_IP:8081/
```

Wait until the UI reports that Nexus is ready (first start can take several minutes).

---

## Login and default repositories

### Login

Nexus creates a default **`admin`** user for administration. On first install, the **initial password** is stored on the server in a file—see [Recovering the admin password](https://help.sonatype.com/) (location such as `admin.password` under the Nexus data directory). Sign in through the UI and complete the password change flow when prompted.

### Default repositories

A fresh Nexus ships with **default repositories** for common formats. You will use **Maven**-style repositories for the course Java projects.

### Repository types and formats

- **Hosted:** stores artifacts you publish (for example **company-internal** builds).  
- **Proxy:** sits in front of another repository (for example **Maven Central**) and caches artifacts.  
- **Group:** aggregates multiple repositories so clients use **one URL**.  

**Formats** depend on technology: **`maven2`** for Java JAR/WAR/POM; **NuGet** for .NET; **Docker** for images—each maps to repositories tuned for that layout.

---

## Create a Nexus user with deployment permissions

Goal: avoid deploying builds as **`admin`**. Create a **dedicated Nexus user** (or service account) with roles that allow **deploy** to your **hosted Maven snapshot** (and/or release) repository.

Typical steps in the Nexus UI:

1. Create a **role** (or use a built-in role) that includes permission to **add/update** components in the target **hosted** repository (for example `maven-snapshots`).  
2. Create a **user**, assign that role.  
3. Optionally use **user tokens** for automation (aligned with the handout’s mention of token-based auth).

You will use this user’s **username and password** (or token) in Gradle and Maven configuration below.

---

## Publish artifacts to Nexus (Gradle)

Goal: **build** the JAR and **publish** it to Nexus using your build tool’s built-in publishing support.

The course Gradle project lives in [java-app](./java-app/). Publishing is configured with **`maven-publish`** in [java-app/build.gradle](./java-app/build.gradle): publication URL, repository name, and credentials read from [java-app/gradle.properties](./java-app/gradle.properties).

1. Set **`repoUser`** and **`repoPassword`** to your Nexus deployment user (or use Gradle properties / environment variables—avoid committing secrets).  
2. Point the repository **`url`** at your hosted snapshot repository, for example:

   `http://YOUR_NEXUS_HOST:8081/repository/maven-snapshots/`

   Replace **`YOUR_NEXUS_HOST`** with your Droplet’s public IP or DNS name. **`allowInsecureProtocol`** is only appropriate for **HTTP** lab setups; production should use **HTTPS**.

3. Produce the bootable JAR and publish:

```bash
cd java-app
gradle bootJar publish
```

If you add the [Gradle wrapper](https://docs.gradle.org/current/userguide/gradle_wrapper.html) to the project, use `./gradlew bootJar publish` instead.

4. In Nexus **Browse**, confirm the component appears under the correct **Maven** repository.

---

## Publish artifacts to Nexus (Maven)

Goal: **`mvn deploy`** uploads the build to the snapshot repository declared in [java-maven-app/pom.xml](./java-maven-app/pom.xml) under **`distributionManagement`** (`nexus-snapshots`).

Maven resolves credentials from **`~/.m2/settings.xml`**, keyed by the **`<id>`** that must **match** the server id in **`distributionManagement`** (`nexus-snapshots`).

**Example `~/.m2/settings.xml` fragment:**

```xml
<settings>
  <servers>
    <server>
      <id>nexus-snapshots</id>
      <username>YOUR_NEXUS_USERNAME</username>
      <password>YOUR_NEXUS_PASSWORD</password>
    </server>
  </servers>
</settings>
```

Update the **`url`** in `pom.xml` if your Nexus host differs (`http://YOUR_NEXUS_HOST:8081/repository/maven-snapshots`).

Deploy:

```bash
cd java-maven-app
mvn clean deploy
```

Verify the artifact in the Nexus UI under the snapshot repository.

---

## Nexus REST API

You can automate Nexus with its **REST API**: query repositories, list components, upload or download artifacts, etc.—often with **`curl`** or **`wget`** and HTTP Basic auth using a Nexus user’s credentials.

Nexus Repository 3 exposes REST resources under **`/service/rest/`**. For example, to **list components** in a repository (sorted by version), use **HTTP Basic** auth and a **GET** request:

```bash
curl -u YOUR_NEXUS_USERNAME:YOUR_NEXUS_PASSWORD \
  -X GET \
  'http://YOUR_NEXUS_HOST:8081/service/rest/v1/components?repository=YOUR_REPO_NAME&sort=version'
```

Replace **`YOUR_REPO_NAME`** with the repository id from Nexus (for example **`maven-snapshots`**). Query parameters and available **`sort`** values are defined in the current API reference—confirm fields and pagination (`continuationToken`) in the docs when you script against this endpoint.

**Practice:** rely on **official API documentation** for endpoints and payloads; APIs evolve, so memorizing URLs is less useful than knowing how to read the docs (as called out in the handout).

---

## Components vs assets

In Nexus terminology:

- **Component:** the logical thing you manage (coordinates / name across a format)—an abstract unit you upload or browse.  
- **Asset:** the **physical file(s)** stored for that component (for example a JAR and its POM).

One **component** can include **one or more assets**.

---

## Cleanup policies and scheduled tasks

Nexus lets you define **cleanup policies** per repository (for example delete artifacts older than **30 days**, or not downloaded for more than **10 days**) and schedule **tasks** to apply those policies on a cadence. Use this to reclaim disk space while matching your retention policy.

---

## References

- README structure guidance: [Make a README](https://www.makeareadme.com/)
- Sonatype **Nexus Repository** documentation: [help.sonatype.com](https://help.sonatype.com/) (installation, repositories, security, REST API)
- DigitalOcean: [Droplets](https://docs.digitalocean.com/products/droplets/), [Cloud Firewalls](https://docs.digitalocean.com/networking/firewalls/)
