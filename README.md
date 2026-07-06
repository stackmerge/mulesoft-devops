# MuleSoft DevOps — CloudHub 2.0 CI/CD with Multi-Environment Deployment & Automated POM Version Update

---

## Overview

This repository is a hands-on demonstration project for MuleSoft DevOps practitioners learning how to implement a **fully automated CI/CD pipeline** using **GitHub Actions** for a MuleSoft 4 application targeting **Anypoint CloudHub 2.0**.

The core focus of this chapter is:

1. **Multi-Environment Deployments** — Automatically deploy to separate CloudHub 2.0 environments (`dev`, `test`, `prod`) based on which Git branch is pushed.
2. **Automated POM Version Update** — Automatically increment the Maven `<version>` in `pom.xml` on every successful `dev` build, then commit and push the change back to the repository using `[skip ci]` to prevent infinite loops.
3. **Anypoint Exchange Publishing** — Publish the built Mule application artifact to Anypoint Exchange as part of the `dev` pipeline.

---

## What This Repository Demonstrates

| Feature | Details |
|---|---|
| CI/CD Tool | GitHub Actions |
| Deployment Target | Anypoint CloudHub 2.0 |
| Environments | `dev`, `test`, `prod` |
| Branch-to-Env Mapping | `dev` branch → DEV, `test` branch → TEST, `main` branch → PROD |
| POM Version Strategy | Automated patch-version increment on every `dev` push |
| Authentication Method | Connected App (Client Credentials) |
| Exchange Publishing | Automated on `dev` branch only |
| Mule Runtime | 4.7.0 |
| Java Version (Build) | JDK 17 (Temurin distribution) |
| Maven Plugin | `mule-maven-plugin` v4.2.0 |

---

## Application — API Description

The Mule application exposes a simple REST API that retrieves user data from the public [Reqres API](https://reqres.in/) and returns it in a transformed JSON format. It is intentionally simple to serve as a learning vehicle for the CI/CD and deployment concepts.

### Endpoint

```
GET http://<host>:8081/getUser/{id}
```

| Component | Value |
|---|---|
| Protocol | HTTP |
| Host | `0.0.0.0` (all interfaces) |
| Port | `8081` |
| Path | `/getUser/{id}` |
| Method | `GET` |
| Path Parameter | `id` — the user ID to retrieve |

### Flow: `getUserFlow`

```
HTTP Listener (GET /getUser/{id})
        |
        v
HTTP Request (GET https://reqres.in/api/users/{id})   [outbound HTTPS]
        |
        v
DataWeave 2.0 Transform
        |
        v
JSON Response → { "UserDetails": { ... } }
```

**Step-by-step description:**

1. **HTTP Listener** (`GET /getUser/{id}`)  
   Listens on `0.0.0.0:8081`. Accepts any `GET` request matching `/getUser/{id}` where `{id}` is a URI path parameter.

2. **HTTP Request** (`getUser`)  
   Makes an outbound HTTPS GET call to:
   ```
   https://reqres.in/api/users/{id}
   ```
   - The `id` value is taken from the URI path parameter of the incoming request via the expression: `"https://reqres.in/api/users/" ++ attributes.uriParams.id default 2`
   - Defaults to user `2` if no ID is provided
   - The response `data` field is stored in `targetValue: #[payload.data]`
   - Uses an insecure TLS trust store (`insecure="true"`) — acceptable for demos only

3. **DataWeave Transform** (`Response`)  
   Transforms the Reqres API response using DataWeave 2.0:
   ```dataweave
   %dw 2.0
   output json
   ---
   {
       UserDetails: payload.data
   }
   ```

**Sample Request:**
```http
GET /getUser/2 HTTP/1.1
Host: localhost:8081
```

**Sample Response:**
```json
{
  "UserDetails": {
    "id": 2,
    "email": "janet.weaver@reqres.in",
    "first_name": "Janet",
    "last_name": "Weaver",
    "avatar": "https://reqres.in/img/faces/2-image.jpg"
  }
}
```

---

## Project Structure

```
ch2-cicd-multi-envs-pom-version-update/
|
+-- .github/
|   +-- workflows/
|       +-- maven.yml                   # GitHub Actions CI/CD pipeline (all 3 environments)
|
+-- exchange-docs/
|   +-- home.md                         # Anypoint Exchange asset home page (placeholder)
|
+-- src/
|   +-- main/
|   |   +-- mule/
|   |   |   +-- mulesoft-devops.xml     # Mule 4 application: flows, HTTP config, transforms
|   |   +-- resources/
|   |       +-- log4j2.xml              # Log4j2 logging configuration (production runtime)
|   +-- test/
|       +-- resources/
|           +-- log4j2-test.xml         # Log4j2 logging configuration (test scope only)
|
+-- .gitignore                          # Excludes: target/, .mule/, IDE files, etc.
+-- mule-artifact.json                  # Mule artifact metadata: minMuleVersion, Java versions
+-- pom.xml                             # Maven POM: project identity + CloudHub 2.0 deploy config
+-- settings.xml                        # Maven server credentials for Exchange and CH2
+-- README.md                           # This documentation file
```

---

## CI/CD Pipeline Architecture

The pipeline is defined in `.github/workflows/maven.yml` and is named **"CloudHub 2.0 deployment workflow"**.

### Branch Strategy and Environment Mapping

```
Developer pushes code
        |
        v
GitHub Actions triggered
        |
        +-- (branch == dev)  --> Job: deployToDev
        |                             1. Update POM version (patch++)
        |                             2. Commit & push pom.xml [skip ci]
        |                             3. Publish to Anypoint Exchange
        |                             4. Deploy to CloudHub 2.0 DEV
        |
        +-- (branch == test) --> Job: deployToTest
        |                             1. Deploy to CloudHub 2.0 TEST
        |
        +-- (branch == main) --> Job: deployToProd
                                      1. Deploy to CloudHub 2.0 PROD
```

**Workflow trigger configuration:**
```yaml
on:
  push:
    branches: [ "main", "dev", "test" ]
  pull_request:
    branches: [ "main", "dev", "test" ]
```

The workflow triggers on both `push` and `pull_request` events. Each job has an `if` condition to ensure it only runs for the appropriate branch.

---

### Job 1: `deployToDev` — triggered on `dev` branch

**Condition:** `if: ${{ github.ref == 'refs/heads/dev' }}`  
**Runner:** `ubuntu-latest`

This is the most complete job. After checking out the code and setting up JDK 17, it performs four distinct operations:

#### Step 1 — Update POM Version

```yaml
- name: Updating POM Version
  run: |
    git config user.email "email ID"
    git config user.name "Username"
    git config pull.rebase false
    git checkout dev
    mvn build-helper:parse-version versions:set \
      -DnewVersion=\${parsedVersion.majorVersion}.\${parsedVersion.minorVersion}.\${parsedVersion.nextIncrementalVersion} \
      versions:commit -s settings.xml --file pom.xml
    git add pom.xml
    git commit -m "[skip ci] version updated."
    git push origin dev
```

This step:
- Configures a git identity so the automated commit can be authored
- Checks out the `dev` branch
- Runs the Maven version update (see [Automated POM Version Update](#automated-pom-version-update))
- Commits the updated `pom.xml` with `[skip ci]` message
- Pushes the commit back to `dev` without triggering another CI run

#### Step 2 — Publish to Anypoint Exchange

```yaml
- name: Publish to Exchange
  run: mvn deploy --file pom.xml -s settings.xml
```

Publishes the built `.jar` artifact to Anypoint Exchange under the organization configured in `<distributionManagement>` in `pom.xml`.

#### Step 3 — Deploy to CloudHub 2.0 DEV

```yaml
- name: Deploy to CH2 dev
  run: mvn deploy -DmuleDeploy -Dmule.env=dev --file pom.xml -s settings.xml
```

Deploys the application to CloudHub 2.0 DEV environment. The `-DmuleDeploy` flag activates the CloudHub 2.0 deployment configuration in the `mule-maven-plugin`. The `-Dmule.env=dev` property is used to name the application and configure the runtime environment.

---

### Job 2: `deployToTest` — triggered on `test` branch

**Condition:** `if: ${{ github.ref == 'refs/heads/test' }}`  
**Runner:** `ubuntu-latest`

This job only deploys to TEST. No version update, no Exchange publish.

```yaml
- name: Deploy to CH2 test
  run: mvn deploy -DmuleDeploy -Dmule.env=test --file pom.xml -s settings.xml
```

The deployed application name will be: `mulesoft-devops-v1-test`

---

### Job 3: `deployToProd` — triggered on `main` branch

**Condition:** `if: ${{ github.ref == 'refs/heads/main' }}`  
**Runner:** `ubuntu-latest`

This job only deploys to PROD. No version update, no Exchange publish.

```yaml
- name: Deploy to CH2 prod
  run: mvn deploy -DmuleDeploy -Dmule.env=prod --file pom.xml -s settings.xml
```

The deployed application name will be: `mulesoft-devops-v1-prod`

---

### Environment Variables in All Jobs

All three jobs share the same environment variable block, which maps GitHub Secrets to environment variables:

```yaml
env:
  CONNECTEDAPP_CLIENT_ID: ${{ secrets.CONNECTEDAPP_CLIENT_ID }}
  CONNECTEDAPP_CLIENT_SECRET: ${{ secrets.CONNECTEDAPP_CLIENT_SECRET }}
```

These variables are used by `settings.xml` to authenticate against Anypoint Platform.

---

## Automated POM Version Update

One of the key learning points in this chapter is the **automated patch version increment** on every push to the `dev` branch.

### The Maven Commands

The version update uses a chain of two Maven plugins:

```bash
mvn build-helper:parse-version versions:set \
  -DnewVersion=${parsedVersion.majorVersion}.${parsedVersion.minorVersion}.${parsedVersion.nextIncrementalVersion} \
  versions:commit -s settings.xml --file pom.xml
```

**Plugin 1: `build-helper:parse-version`**

Parses the current `<version>` in `pom.xml` and exposes the components as Maven properties:

| Property | Example Value | Description |
|---|---|---|
| `parsedVersion.majorVersion` | `1` | Major version number |
| `parsedVersion.minorVersion` | `0` | Minor version number |
| `parsedVersion.incrementalVersion` | `8` | Current patch version |
| `parsedVersion.nextIncrementalVersion` | `9` | Patch version + 1 |

**Plugin 2: `versions:set`**

Sets the new `<version>` in `pom.xml` using the parsed components, assembling them as:
```
{majorVersion}.{minorVersion}.{nextIncrementalVersion}
```

**Plugin 3: `versions:commit`**

Removes the backup `pom.xml.versionsBackup` file that `versions:set` creates.

### Version Progression Example

| Event | Version |
|---|---|
| Initial commit | `1.0.0` |
| 1st `dev` push | `1.0.1` |
| 2nd `dev` push | `1.0.2` |
| ... | ... |
| 9th `dev` push | `1.0.9` ← current |

### Preventing Infinite CI Loops

After updating the version, the pipeline commits and pushes `pom.xml` back to `dev`:

```bash
git commit -m "[skip ci] version updated."
git push origin dev
```

The `[skip ci]` text in the commit message is a special directive recognized by GitHub Actions. When GitHub sees a commit message containing `[skip ci]`, it **does not trigger** a new workflow run, preventing an infinite loop of version bumps.

---

## CloudHub 2.0 Deployment Configuration

The CloudHub 2.0 deployment is configured in `pom.xml` inside the `mule-maven-plugin` configuration block:

```xml
<plugin>
    <groupId>org.mule.tools.maven</groupId>
    <artifactId>mule-maven-plugin</artifactId>
    <version>4.2.0</version>
    <extensions>true</extensions>
    <configuration>
        <cloudhub2Deployment>
            <uri>https://anypoint.mulesoft.com</uri>
            <provider>MC</provider>
            <environment>${mule.env}</environment>
            <target>Cloudhub-US-East-2</target>
            <muleVersion>4.7.0</muleVersion>
            <server>connectedapp</server>
            <applicationName>${project.artifactId}-v1-${mule.env}</applicationName>
            <replicas>1</replicas>
            <vCores>0.1</vCores>
            <businessGroupId>${project.groupId}</businessGroupId>
            <skipDeploymentVerification>true</skipDeploymentVerification>
            <properties>
                <mule.env>${mule.env}</mule.env>
                <anypoint.platform.config.analytics.agent.enabled>true</anypoint.platform.config.analytics.agent.enabled>
                <anypoint.platform.visualizer.layer>Experience</anypoint.platform.visualizer.layer>
            </properties>
            <secureProperties>
                <mule.key>${env.ENCRYPTION_KEY}</mule.key>
                <anypoint.platform.client_id>${env.ENV_CLIENT_ID}</anypoint.platform.client_id>
                <anypoint.platform.client_secret>${env.ENV_CLIENT_SECRET}</anypoint.platform.client_secret>
            </secureProperties>
            <deploymentSettings>
                <generateDefaultPublicUrl>true</generateDefaultPublicUrl>
                <updateStrategy>recreate</updateStrategy>
            </deploymentSettings>
        </cloudhub2Deployment>
    </configuration>
</plugin>
```

### Deployment Parameters Reference

| Parameter | Value | Description |
|---|---|---|
| `uri` | `https://anypoint.mulesoft.com` | Anypoint Platform base URL |
| `provider` | `MC` | MuleSoft Cloud (CloudHub 2.0 managed) |
| `environment` | `${mule.env}` | Injected at pipeline runtime: `dev`, `test`, or `prod` |
| `target` | `Cloudhub-US-East-2` | CloudHub 2.0 deployment target (US East region) |
| `muleVersion` | `4.7.0` | Mule runtime version |
| `server` | `connectedapp` | Server ID in settings.xml for auth |
| `applicationName` | `mulesoft-devops-v1-{env}` | Unique app name per environment |
| `replicas` | `1` | Number of application replicas |
| `vCores` | `0.1` | vCore allocation (smallest shared size) |
| `businessGroupId` | `${project.groupId}` | Anypoint org/business group ID |
| `skipDeploymentVerification` | `true` | Skip deployment health wait |
| `generateDefaultPublicUrl` | `true` | Auto-create a public URL |
| `updateStrategy` | `recreate` | Stop old replicas before new deploy |

---

| muleVersion | 4.7.0 | Mule runtime version to deploy on |
| server | connectedapp | Server ID in settings.xml for auth |
| applicationName | mulesoft-devops-v1-{env} | Unique app name per environment |
| replicas | 1 | Number of application replicas |
| vCores | 0.1 | vCore allocation (smallest shared size) |
| businessGroupId | ${project.groupId} | Anypoint org/business group ID |
| skipDeploymentVerification | true | Skip deployment health wait |
| generateDefaultPublicUrl | true | Auto-create a public URL |
| updateStrategy | recreate | Stop old replicas before new deploy |

---

## Local Development Setup



Update pom.xml groupId to your Anypoint org ID.

Test the API locally after running in Anypoint Studio on port 8081:


---

## Triggering Deployments

| Action | Branch | Result |
|---|---|---|
| Push to dev | dev | Increment POM version, publish to Exchange, deploy to DEV |
| Merge PR to dev | dev | Same as above |
| Push to test | test | Deploy to CloudHub 2.0 TEST environment |
| Push to main | main | Deploy to CloudHub 2.0 PROD environment |

### Maven Commands for Manual Deployment



---

## Security and Secure Properties

Secure (encrypted) runtime properties passed to CloudHub 2.0 (not visible in plain text in UI):

| Secure Property | Source Env Variable | Purpose |
|---|---|---|
| mule.key | ENCRYPTION_KEY | Encryption key for Mule Secure Properties module |
| anypoint.platform.client_id | ENV_CLIENT_ID | Anypoint environment client ID for autodiscovery |
| anypoint.platform.client_secret | ENV_CLIENT_SECRET | Anypoint environment client secret |

Best practices:
1. Use GitHub Secrets for all credentials - never commit to version control
2. Use environment-specific secrets (DEV_CLIENT_ID, PROD_CLIENT_ID) for isolation
3. Rotate Connected App credentials periodically
4. Grant minimal required permissions to Connected Apps

---

## Logging Configuration

**Production:** src/main/resources/log4j2.xml
Log4j2 configuration for the production Mule runtime with standard async appenders.

**Test:** src/test/resources/log4j2-test.xml
Separate test-scoped Log4j2 configuration with more verbose logging for debugging.

---

## Key Concepts Covered

| Concept | Where Demonstrated |
|---|---|
| GitHub Actions workflow definition | .github/workflows/maven.yml |
| Branch-based environment promotion | if conditionals in each job |
| CloudHub 2.0 deployment | pom.xml cloudhub2Deployment config block |
| Multi-environment application naming | applicationName with mule.env property |
| Connected App auth in CI/CD | settings.xml server configuration |
| Automated semantic version increment | build-helper:parse-version + versions:set |
| Preventing infinite CI loops | [skip ci] commit message convention |
| Anypoint Exchange publishing | mvn deploy with distributionManagement |
| Secure runtime property injection | secureProperties in cloudhub2Deployment |
| API Visualizer classification | anypoint.platform.visualizer.layer property |
| Anypoint Analytics enablement | anypoint.platform.config.analytics.agent.enabled |
| Maven dependency caching | cache: maven in actions/setup-java@v4 |

---

## References

- [MuleSoft Documentation](https://docs.mulesoft.com/general/)
- [Mule Maven Plugin - CloudHub 2.0](https://docs.mulesoft.com/mule-runtime/latest/deploy-to-cloudhub-2)
- [GitHub Actions Workflow Syntax](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions)
- [Anypoint Platform - Connected Apps](https://docs.mulesoft.com/access-management/connected-apps-overview)
- [Anypoint Exchange - Publishing Assets](https://docs.mulesoft.com/exchange/to-publish-assets-maven)
- [build-helper:parse-version Plugin](https://www.mojohaus.org/build-helper-maven-plugin/parse-version-mojo.html)
- [versions:set Plugin](https://www.mojohaus.org/versions/versions-maven-plugin/set-mojo.html)
- [Reqres API - External Test API](https://reqres.in/)
- [Original Repository: ash-citiustech/mulesoft-devops](https://github.com/ash-citiustech/mulesoft-devops)

---

*README generated for the MuleSoft Academy CI/CD Series - Chapter 2*
