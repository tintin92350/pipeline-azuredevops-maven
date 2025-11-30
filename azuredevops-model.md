# CI/CD pipeline strategy for Azure DevOps with Maven and Nexus

A practical implementation of Maven Release Plugin with Azure DevOps 2022 pipelines requires orchestrating three core components: **automated version management** through Maven's prepare/perform lifecycle, **multi-stage deployments** with environment-specific gates, and **artifact promotion** via Nexus Repository. This guide presents a streamlined approach that balances enterprise requirements with simplicity, avoiding the over-engineering that plagues many Java CI/CD implementations.

The strategy centers on a modified GitFlow model where **develop** runs SNAPSHOT builds, **release branches** stage through UAT, and **main** represents production-ready code. Maven Release Plugin handles version transitions automatically, while Azure DevOps environment approvals control promotion gates.

---

## Maven Release Plugin configuration for CI/CD

The maven-release-plugin operates through two distinct phases that must execute in sequence. The **prepare** phase transforms `1.2.0-SNAPSHOT` to `1.2.0`, commits the change, creates a Git tag (`v1.2.0`), then bumps to `1.3.0-SNAPSHOT` and commits again. The **perform** phase checks out the tagged version and executes deployment to Nexus.

### Essential pom.xml configuration

```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.company</groupId>
    <artifactId>my-service</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    
    <!-- SCM Configuration for Azure DevOps Git -->
    <scm>
        <connection>scm:git:https://dev.azure.com/org/project/_git/repo</connection>
        <developerConnection>scm:git:https://org@dev.azure.com/org/project/_git/repo</developerConnection>
        <url>https://dev.azure.com/org/project/_git/repo</url>
        <tag>HEAD</tag>
    </scm>
    
    <properties>
        <project.scm.id>azure-devops</project.scm.id>
    </properties>
    
    <distributionManagement>
        <repository>
            <id>nexus-releases</id>
            <url>https://nexus.company.com/repository/maven-releases/</url>
        </repository>
        <snapshotRepository>
            <id>nexus-snapshots</id>
            <url>https://nexus.company.com/repository/maven-snapshots/</url>
        </snapshotRepository>
    </distributionManagement>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-release-plugin</artifactId>
                <version>3.2.0</version>
                <configuration>
                    <tagNameFormat>v@{project.version}</tagNameFormat>
                    <autoVersionSubmodules>true</autoVersionSubmodules>
                    <projectVersionPolicyId>SemVerVersionPolicy</projectVersionPolicyId>
                    <pushChanges>true</pushChanges>
                    <localCheckout>true</localCheckout>
                    <goals>deploy</goals>
                    <arguments>${arguments}</arguments>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

The `<project.scm.id>` property links to server credentials in settings.xml, enabling authentication without exposing secrets in the POM. Using **version 3.2.0** of the release plugin ensures automatic tag removal during rollback operations—a critical improvement over earlier versions.

### CI/CD execution commands

For non-interactive pipeline execution, always use batch mode with explicit version control:

```bash
# Full release (prepare + perform in single command)
mvn --batch-mode release:clean release:prepare release:perform \
    -DreleaseVersion=1.2.0 \
    -DdevelopmentVersion=1.3.0-SNAPSHOT \
    -Darguments="-DskipTests"

# Dry run for validation
mvn -B -DdryRun=true release:prepare
```

The `--batch-mode` flag (-B) prevents interactive prompts that would stall CI pipelines. Skip tests during the release phases since testing should occur in dedicated pipeline stages before release initiation.

---

## Azure DevOps multi-stage pipeline architecture

The pipeline implements a **build-once, deploy-many** pattern where artifacts compile once and promote through environments. Each environment has distinct approval requirements configured through Azure DevOps Environment resources.

### Complete pipeline structure

```yaml
trigger:
  branches:
    include:
      - main
      - develop
      - release/*
      - hotfix/*

variables:
  - group: 'nexus-credentials'
  - name: MAVEN_CACHE_FOLDER
    value: $(Pipeline.Workspace)/.m2/repository
  - name: MAVEN_OPTS
    value: '-Dmaven.repo.local=$(MAVEN_CACHE_FOLDER)'

pool:
  vmImage: 'ubuntu-latest'

stages:
# ========================================
# BUILD AND TEST
# ========================================
- stage: Build
  displayName: 'Build and Test'
  jobs:
  - job: BuildJob
    steps:
    - task: Cache@2
      displayName: 'Cache Maven dependencies'
      inputs:
        key: 'maven | "$(Agent.OS)" | **/pom.xml'
        path: $(MAVEN_CACHE_FOLDER)

    - task: JavaToolInstaller@0
      displayName: 'Configure Java 11'
      inputs:
        versionSpec: '11'
        jdkArchitectureOption: 'x64'
        jdkSourceOption: 'PreInstalled'

    - task: DownloadSecureFile@1
      name: settingsXml
      inputs:
        secureFile: 'maven-settings.xml'

    - task: Maven@4
      displayName: 'Maven Build'
      inputs:
        mavenPomFile: 'pom.xml'
        goals: 'clean package'
        options: '-s $(settingsXml.secureFilePath) $(MAVEN_OPTS)'
        javaHomeOption: 'JDKVersion'
        jdkVersionOption: '1.11'
        publishJUnitResults: true
        testResultsFiles: '**/surefire-reports/TEST-*.xml'

    - task: PublishPipelineArtifact@1
      displayName: 'Publish Artifact'
      inputs:
        targetPath: '$(System.DefaultWorkingDirectory)/target/*.jar'
        artifactName: 'java-app'

# ========================================
# DEPLOY TO DEV (automatic for develop branch)
# ========================================
- stage: DeployDev
  displayName: 'Deploy to Development'
  dependsOn: Build
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/develop'))
  jobs:
  - deployment: DeployDevJob
    environment: 'development'
    strategy:
      runOnce:
        deploy:
          steps:
          - download: current
            artifact: 'java-app'
          - script: echo "Deploying SNAPSHOT to dev environment"
            displayName: 'Deploy Application'

# ========================================
# DEPLOY TO UAT (release/hotfix branches, manual approval)
# ========================================
- stage: DeployUAT
  displayName: 'Deploy to UAT'
  dependsOn: Build
  condition: |
    and(
      succeeded(),
      or(
        startsWith(variables['Build.SourceBranch'], 'refs/heads/release/'),
        startsWith(variables['Build.SourceBranch'], 'refs/heads/hotfix/')
      )
    )
  jobs:
  - deployment: DeployUATJob
    environment: 'uat'  # Configure approval in Environment settings
    strategy:
      runOnce:
        deploy:
          steps:
          - download: current
            artifact: 'java-app'
          - script: echo "Deploying to UAT for validation"
            displayName: 'Deploy Application'

# ========================================
# RELEASE TO NEXUS (main branch only)
# ========================================
- stage: Release
  displayName: 'Maven Release to Nexus'
  dependsOn: Build
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: ReleaseJob
    environment: 'production'  # Requires senior approval
    strategy:
      runOnce:
        deploy:
          steps:
          - checkout: self
            persistCredentials: true
            
          - script: |
              git config --global user.email "azure-pipelines@company.com"
              git config --global user.name "Azure Pipelines"
            displayName: 'Configure Git Identity'

          - task: DownloadSecureFile@1
            name: settingsXml
            inputs:
              secureFile: 'maven-settings.xml'

          - task: Maven@4
            displayName: 'Maven Release'
            inputs:
              mavenPomFile: 'pom.xml'
              goals: '--batch-mode release:clean release:prepare release:perform'
              options: '-s $(settingsXml.secureFilePath) -Darguments="-DskipTests"'
              javaHomeOption: 'JDKVersion'
              jdkVersionOption: '1.11'
```

### Environment approval configuration

Azure DevOps environments provide the approval gates. Navigate to **Pipelines → Environments** and configure:

| Environment | Approval Type | Approvers |
|-------------|--------------|-----------|
| development | None (automatic) | — |
| uat | Manual approval | QA Team |
| production | Manual approval + Business Hours | Release Manager, Tech Lead |

The `deployment` job type (not `job`) is required to leverage environment approvals. Regular jobs cannot enforce gates.

---

## Branching strategy integrated with Maven versioning

The branching model maps directly to Maven version states and deployment environments. This eliminates confusion about which branch deploys where and what version format each carries.

### Branch-environment-version matrix

| Branch | Environment | Maven Version | Artifact Type |
|--------|-------------|---------------|---------------|
| `feat/*` | Local/Dev | `X.Y.Z-SNAPSHOT` | Not deployed |
| `develop` | Development | `X.Y.Z-SNAPSHOT` | Snapshot |
| `release/X.Y.Z` | UAT | `X.Y.Z-SNAPSHOT` → `X.Y.Z` | Snapshot → Release |
| `hotfix/X.Y.Z` | UAT | `X.Y.Z-SNAPSHOT` → `X.Y.Z` | Snapshot → Release |
| `main` | Production | `X.Y.Z` (tagged) | Release |

### Workflow execution steps

**Feature Development:**
1. Create `feat/JIRA-123-description` from `develop`
2. Version remains `1.3.0-SNAPSHOT` (inherited)
3. Merge to `develop` via Pull Request
4. CI deploys SNAPSHOT to dev environment

**Release Preparation:**
1. Create `release/1.2.0` from `develop` when features complete
2. Version stays `1.2.0-SNAPSHOT` during UAT testing
3. Bug fixes occur directly on release branch
4. After UAT approval: merge to `main`, back-merge to `develop`
5. Maven Release Plugin on `main` creates tag `v1.2.0`

**Hotfix Workflow (with mandatory UAT):**
1. Create `hotfix/1.2.1` from `main` (the only branch forking from main)
2. Version: `1.2.1-SNAPSHOT` during development
3. Deploy to UAT environment for validation
4. After approval: merge to `main` and `develop`
5. Release plugin creates tag `v1.2.1`

### Merge flow rules

The critical principle: **code flows down to develop, releases flow up to main**. Never merge main directly back into develop—use release or hotfix branches for back-merging fixes.

```
feat/* ─────┐
            ├──► develop ──► release/* ──► main
feat/* ─────┘         ▲           │          │
                      │           ▼          │
                      └───────────┴──────────┘
                        (back-merge fixes)    (hotfix back-merge)
```

---

## Nexus Repository configuration and promotion patterns

Nexus must maintain strict separation between mutable SNAPSHOT artifacts and immutable releases. This separation enables different retention policies and ensures production deployments are reproducible.

### Repository structure

Configure four repository types in Nexus:

- **maven-releases** (hosted): Immutable release artifacts, deployment policy "Disable Redeploy"
- **maven-snapshots** (hosted): Mutable development builds, allows redeploy
- **maven-central** (proxy): Caches Maven Central artifacts
- **maven-public** (group): Aggregates all above for unified consumption

### settings.xml for Azure DevOps pipelines

Store this file in Azure DevOps **Pipelines → Library → Secure files**:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings>
    <servers>
        <server>
            <id>nexus-releases</id>
            <username>${env.NEXUS_USERNAME}</username>
            <password>${env.NEXUS_PASSWORD}</password>
        </server>
        <server>
            <id>nexus-snapshots</id>
            <username>${env.NEXUS_USERNAME}</username>
            <password>${env.NEXUS_PASSWORD}</password>
        </server>
        <server>
            <id>azure-devops</id>
            <username>${env.GIT_USERNAME}</username>
            <password>${env.GIT_PAT}</password>
        </server>
    </servers>
    <mirrors>
        <mirror>
            <id>nexus</id>
            <mirrorOf>*</mirrorOf>
            <url>https://nexus.company.com/repository/maven-public/</url>
        </mirror>
    </mirrors>
</settings>
```

Environment variables inject secrets at runtime. In the pipeline, set these from Variable Groups linked to Azure Key Vault or stored as secret variables.

### Artifact cleanup policies

Configure scheduled tasks in Nexus to prevent disk exhaustion:

**Snapshot cleanup (run daily):**
- Minimum snapshot count to keep: **1**
- Snapshot retention: **14 days**
- Remove if released: **true**
- Grace period after release: **7 days**

**Compact blob store (run weekly):** Reclaims disk space from deleted artifacts.

Releases require no automated cleanup—maintain historical versions for rollback capability. Archive to cold storage if disk becomes a concern.

---

## Credential management and security configuration

Pipeline security requires careful separation of secrets from code. Azure DevOps provides multiple mechanisms for secure credential injection.

### Azure DevOps configuration

1. **Create Variable Group** (`nexus-credentials`):
   - `NEXUS_USERNAME`: Service account username
   - `NEXUS_PASSWORD`: (marked secret) Service account password
   - `GIT_PAT`: (marked secret) Personal Access Token for Git push

2. **Store settings.xml as Secure File**:
   - Upload to Library → Secure files
   - Reference via `DownloadSecureFile@1` task

3. **Grant Git permissions**:
   - **Project Settings → Repositories → Security**
   - Grant `Project Collection Build Service` the `Contribute` and `Create Tag` permissions
   - For release commits: Consider "Bypass policies when pushing" for the CI service account

### Nexus service account configuration

Create a dedicated `ci-deployer` role with minimal privileges:

```
Privileges:
- nx-repository-view-maven2-maven-releases-add
- nx-repository-view-maven2-maven-releases-edit
- nx-repository-view-maven2-maven-snapshots-*
- nx-repository-view-maven2-maven-public-browse
- nx-repository-view-maven2-maven-public-read
```

Never use admin accounts for automated deployments. Rotate the service account password quarterly.

---

## Common pitfalls and their solutions

**Git authentication failures during release:perform**
The release plugin pushes commits and tags. Ensure `persistCredentials: true` on the checkout step and that the Git identity is configured before running Maven commands.

**"Repository does not allow updating assets"**
Nexus rejects redeployment to release repositories by design. If you need to fix a released version, increment the version number. Never enable redeployment on release repositories.

**Tests running twice**
Tests execute during both prepare and perform phases by default. Skip during release with `-Darguments="-DskipTests"` and run comprehensive tests in a dedicated Build stage before release.

**Branch policies blocking release commits**
Azure DevOps branch policies may prevent the release plugin from pushing commits to main. Either exclude the CI service account from policies or run releases from a branch that allows direct pushes.

---

## Conclusion

This pipeline strategy delivers **predictable releases** through Maven's proven version management, **controlled promotion** via Azure DevOps environment approvals, and **artifact integrity** through Nexus repository separation. The simplified GitFlow branching model provides clear rules without the overhead of pure GitFlow's complexity.

Key implementation priorities: configure batch mode for all Maven operations, store credentials exclusively in secure variables, and enforce the build-once-deploy-many pattern to ensure environment consistency. The complete pipeline executes in under 15 minutes for typical Java projects and scales to multi-module Maven builds with minimal modification.

For teams new to this approach, start with the develop → main flow, adding release branches only when UAT staging becomes necessary. The hotfix workflow can layer in once the core process stabilizes.