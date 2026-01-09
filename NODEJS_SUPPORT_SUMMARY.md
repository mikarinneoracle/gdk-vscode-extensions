# Summary: How to Add Node.js Support to GDK VSCode Extensions

## Current Architecture Overview

This project is a suite of VS Code extensions for Graal Development Kit (GDK) and Micronaut applications, with OCI DevOps integration. Currently, it supports **Java-based frameworks only**:

- **Project Types**: GDK, Micronaut, SpringBoot, Helidon
- **Build Systems**: Maven, Gradle
- **Artifacts**: JAR files, Native executables
- **Containerization**: Docker images (JVM and Native)

## Key Components to Modify

### 1. Project Type Detection (`oci-devops/src/projectUtils.ts`)

**Current Implementation:**
- `ProjectType` enum: `'GDK' | 'Micronaut' | 'SpringBoot' | 'Helidon' | 'Unknown'`
- Detection logic in `getProjectFolder()` checks for:
  - Java-specific files: `pom.xml`, `build.gradle`, `application.properties`, `application.yml`
  - Java project structure: `src/main/java`, `src/main/resources`
  - Framework-specific markers (e.g., `micronaut-cli.yml` for Micronaut)

**Changes Needed:**
```typescript
// Add to ProjectType enum
export type ProjectType = 'GDK' | 'Micronaut' | 'SpringBoot' | 'Helidon' | 'NodeJS' | 'Unknown';

// Add detection logic in getProjectFolder()
// Check for package.json existence
if (fs.existsSync(path.join(folder.uri.fsPath, 'package.json'))) {
    // Optionally check for framework-specific files
    // e.g., Express, NestJS, Next.js indicators
    const projectType: ProjectType = 'NodeJS';
    return Object.assign({}, folder, { projectType, buildSystem: 'NPM', subprojects });
}
```

### 2. Build System Support (`oci-devops/src/projectUtils.ts`)

**Current Implementation:**
- `BuildSystemType`: `'Maven' | 'Gradle'`
- Functions: `isMaven()`, `isGradle()` check for wrapper scripts

**Changes Needed:**
```typescript
// Extend BuildSystemType
export type BuildSystemType = 'Maven' | 'Gradle' | 'NPM' | 'Yarn' | 'PNPM';

// Add detection functions
function isNPM(folder: ProjectFolder) {
    return fs.existsSync(path.join(folder.uri.fsPath, 'package.json'));
}

function isYarn(folder: ProjectFolder) {
    return fs.existsSync(path.join(folder.uri.fsPath, 'yarn.lock'));
}

function isPNPM(folder: ProjectFolder) {
    return fs.existsSync(path.join(folder.uri.fsPath, 'pnpm-lock.yaml'));
}
```

### 3. Build Commands (`oci-devops/src/projectUtils.ts`)

**Current Implementation:**
- `getProjectBuildCommand()` returns Maven/Gradle commands
- `getProjectBuildNativeExecutableCommand()` for native builds

**Changes Needed:**
```typescript
export async function getProjectBuildCommand(
    folder: ProjectFolder, 
    subfolder: string = 'oci', 
    includeTests: boolean = false
): Promise<string | undefined> {
    // ... existing Maven/Gradle logic ...
    
    if (isNPM(folder) || isYarn(folder) || isPNPM(folder)) {
        if (folder.projectType === 'NodeJS') {
            const testFlag = includeTests ? '' : ' --ignore-scripts';
            if (isYarn(folder)) {
                return `yarn install${testFlag} && yarn build`;
            } else if (isPNPM(folder)) {
                return `pnpm install${testFlag} && pnpm build`;
            } else {
                return `npm install${testFlag} && npm run build`;
            }
        }
    }
    return undefined;
}
```

### 4. Artifact Location Detection (`oci-devops/src/projectUtils.ts`)

**Current Implementation:**
- `getProjectBuildArtifactLocation()` returns JAR file paths
- `getProjectNativeExecutableArtifactLocation()` returns native executable paths

**Changes Needed:**
```typescript
export async function getProjectBuildArtifactLocation(
    folder: ProjectFolder, 
    subfolder: string = 'oci', 
    shaded: boolean = true
): Promise<string | undefined> {
    // ... existing Java logic ...
    
    if (folder.projectType === 'NodeJS') {
        // Check package.json for output directory
        const packageJsonPath = path.join(folder.uri.fsPath, 'package.json');
        if (fs.existsSync(packageJsonPath)) {
            const packageJson = JSON.parse(fs.readFileSync(packageJsonPath, 'utf8'));
            // Common output directories
            if (packageJson.main) {
                return packageJson.main;
            }
            // Framework-specific defaults
            if (fs.existsSync(path.join(folder.uri.fsPath, 'dist'))) {
                return 'dist';
            }
            if (fs.existsSync(path.join(folder.uri.fsPath, 'build'))) {
                return 'build';
            }
            if (fs.existsSync(path.join(folder.uri.fsPath, '.next'))) {
                return '.next'; // Next.js
            }
            return '.'; // Root directory for simple apps
        }
    }
    return undefined;
}
```

### 5. Dockerfile Templates (`oci-devops/resources/oci/`)

**Current Templates:**
- `Dockerfile.jvm.handlebars` - For JVM-based Java applications
- `Dockerfile.native.handlebars` - For native executables

**New Templates Needed:**

**`Dockerfile.nodejs.handlebars`** (for standard Node.js apps):
```dockerfile
FROM container-registry.oracle.com/os/oraclelinux:8-slim

ARG NODE_VERSION=18
RUN microdnf install -y nodejs npm && \
    microdnf clean all

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application files
COPY {{{app_files}}} ./

EXPOSE 3000

# Start application
CMD ["node", "{{{entry_point}}}"]
```

**`Dockerfile.nodejs.multi-stage.handlebars`** (for optimized builds):
```dockerfile
# Build stage
FROM container-registry.oracle.com/os/oraclelinux:8-slim AS builder

ARG NODE_VERSION=18
RUN microdnf install -y nodejs npm && \
    microdnf clean all

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Runtime stage
FROM container-registry.oracle.com/os/oraclelinux:8-slim

ARG NODE_VERSION=18
RUN microdnf install -y nodejs npm && \
    microdnf clean all

WORKDIR /app

COPY --from=builder /app/{{{build_output}}} ./
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./

EXPOSE 3000

CMD ["node", "{{{entry_point}}}"]
```

### 6. Build Pipeline Specs (`oci-devops/resources/oci/`)

**Current Specs:**
- `devbuild_spec.yaml.handlebars` - Builds JAR files
- `nibuild_spec.yaml.handlebars` - Builds native executables
- `docker_jvmbuild_spec.yaml.handlebars` - Builds JVM Docker images
- `docker_nibuild_spec.yaml.handlebars` - Builds native Docker images

**New Spec Needed:**

**`nodejs_build_spec.yaml.handlebars`**:
```yaml
version: 0.1
component: build
timeoutInSeconds: 15000
runAs: root
shell: bash
env:
  exportedVariables:
    - NODE_VERSION
steps:
  - type: Command
    name: "Install Node.js"
    timeoutInSeconds: 300
    command: |
      if [ ! -n "${NODE_VERSION}" ]; then export NODE_VERSION={{{default_node_version}}}; fi
      echo "Node.js Version: ${NODE_VERSION}"
      microdnf install -y nodejs npm
      node --version
      npm --version
  - type: Command
    name: "Install dependencies and build"
    timeoutInSeconds: 600
    command: |
      echo "Executing build command: {{{project_build_command}}}"
      {{{project_build_command}}}
    onFailure:
      - type: Command
        command: |
          echo "Build failed"
        timeoutInSeconds: 40
        runAs: root
outputArtifacts:
  - name: {{{deploy_artifact_name}}}
    type: BINARY
    location: ${OCI_PRIMARY_SOURCE_DIR}/{{{project_artifact_location}}}
```

**`docker_nodejs_build_spec.yaml.handlebars`**:
```yaml
version: 0.1
component: build
timeoutInSeconds: 15000
runAs: root
shell: bash
env:
  exportedVariables:
    - NODE_VERSION
steps:
  - type: Command
    name: "Install Node.js"
    timeoutInSeconds: 300
    command: |
      if [ ! -n "${NODE_VERSION}" ]; then export NODE_VERSION={{{default_node_version}}}; fi
      echo "Node.js Version: ${NODE_VERSION}"
      microdnf install -y nodejs npm
  - type: Command
    name: "Define docker image tag"
    timeoutInSeconds: 40
    command: |
      echo "OCI_BUILD_RUN_ID: ${OCI_BUILD_RUN_ID}"
      if [ ! -n "${DOCKER_TAG}" ]; then export DOCKER_TAG={{{docker_tag_value}}}; fi
      echo "DOCKER_TAG: ${DOCKER_TAG}"
  - type: Command
    name: "Docker build"
    command: |
      echo "Running docker image build."
      echo "Docker Image Tag: ${DOCKER_TAG}"
      docker build -f ./.devops/Dockerfile.nodejs \
                  -t {{{image_name}}}:${DOCKER_TAG} .
      echo "Done"
      printf "List of docker images:\n $(docker images)"
outputArtifacts:
  - name: {{{deploy_artifact_name}}}
    type: DOCKER_IMAGE
    location: {{{image_name}}}:${DOCKER_TAG}
```

### 7. Deployment Logic (`oci-devops/src/oci/deployUtils.ts`)

**Current Implementation:**
- `deployFolders()` handles Java project deployment
- Creates build specs, pipelines, and container repositories
- Handles OKE deployment configurations

**Changes Needed:**
- Add Node.js project type handling in the deployment flow
- Create Node.js-specific build pipelines
- Generate Node.js Dockerfiles
- Configure Node.js-specific OKE deployment manifests

**Key Areas to Modify:**
```typescript
// Around line 352-376 in deployUtils.ts
if (projectFolder.projectType === 'NodeJS') {
    totalSteps += 6; // Node.js build spec and pipeline, Docker image, build spec and pipeline, container repository
    if (!bypassArtifacts) {
        totalSteps += 1; // Build artifact
    }
    if (deployData.okeCluster) {
        totalSteps += 4; // OKE deploy spec and artifact, deploy to OKE pipeline
    }
}
```

### 8. Resource Templates (`oci-devops/src/oci/ociResources.ts`)

**Current Implementation:**
- `RESOURCES` object maps template names to file paths
- Templates are loaded and expanded using Handlebars

**Changes Needed:**
```typescript
// Add new template mappings
export const RESOURCES = {
    // ... existing templates ...
    'Dockerfile.nodejs': 'oci/Dockerfile.nodejs.handlebars',
    'nodejs_build_spec.yaml': 'oci/nodejs_build_spec.yaml.handlebars',
    'docker_nodejs_build_spec.yaml': 'oci/docker_nodejs_build_spec.yaml.handlebars',
};
```

### 9. OKE Deployment Configuration (`oci-devops/resources/oci/oke_deploy_config.yaml.handlebars`)

**Current Implementation:**
- Configures Java application deployment to Kubernetes
- Sets up environment variables, ports, resource limits

**Changes Needed:**
- Add Node.js-specific configuration options
- Handle Node.js environment variables (NODE_ENV, PORT, etc.)
- Configure appropriate resource limits for Node.js apps
- **Note**: Node.js applications will use port **3000** by default (instead of 8080 used for Java applications)

### 10. Common Types (`common/src/dialogs.ts`)

**Current Implementation:**
```typescript
export type ProjectType = "GDK" | "Micronaut";
```

**Changes Needed:**
```typescript
export type ProjectType = "GDK" | "Micronaut" | "NodeJS";
```

## Implementation Steps

### Phase 1: Core Detection
1. ✅ Extend `ProjectType` enum to include `'NodeJS'`
2. ✅ Extend `BuildSystemType` to include `'NPM' | 'Yarn' | 'PNPM'`
3. ✅ Add Node.js project detection in `getProjectFolder()`
4. ✅ Add build system detection functions (`isNPM`, `isYarn`, `isPNPM`)

### Phase 2: Build Support
5. ✅ Implement `getProjectBuildCommand()` for Node.js
6. ✅ Implement `getProjectBuildArtifactLocation()` for Node.js
7. ✅ Create Node.js build spec templates
8. ✅ Update `RESOURCES` mapping

### Phase 3: Containerization
9. ✅ Create `Dockerfile.nodejs.handlebars` template
10. ✅ Create `docker_nodejs_build_spec.yaml.handlebars` template
11. ✅ Update deployment logic to generate Node.js Dockerfiles

### Phase 4: Deployment
12. ✅ Update `deployFolders()` to handle Node.js projects
13. ✅ Create Node.js-specific OKE deployment configurations
14. ✅ Update OKE deployment manifests for Node.js

### Phase 5: Testing & Documentation
15. ✅ Add test fixtures for Node.js projects
16. ✅ Update README.md with Node.js support information
17. ✅ Test end-to-end deployment flow

## Key Considerations

### Node.js Version Management
- Default Node.js version should be configurable
- Support for LTS versions (16, 18, 20)
- Consider using NodeSource or official Oracle Linux Node.js packages

### Port Configuration
- **Node.js applications will use port 3000** (standard Node.js convention)
- Java applications use port 8080
- This affects Dockerfile EXPOSE directives and OKE service configurations

### Framework Detection
- Detect popular frameworks (Express, NestJS, Next.js, etc.)
- Framework-specific build commands and artifact locations
- Framework-specific Dockerfile optimizations

### Build Tool Support
- Primary: npm (default)
- Secondary: Yarn (via `yarn.lock` detection)
- Tertiary: pnpm (via `pnpm-lock.yaml` detection)

### Artifact Handling
- Support for different output formats:
  - Single-file applications
  - Multi-file builds (Next.js, Nuxt, etc.)
  - Serverless functions
- Handle `package.json` scripts and entry points

### Native Executables
- Consider GraalVM Native Image for Node.js (experimental)
- Or use alternative tools like `pkg` or `nexe` for native executables
- May require separate handling from Java native images

### Dependencies
- No dependency on Apache NetBeans Language Server for Node.js
- May need to check for Node.js-specific VS Code extensions
- Consider integration with existing Node.js tooling

## Testing Strategy

1. **Unit Tests**: Test project detection logic
2. **Integration Tests**: Test build command generation
3. **E2E Tests**: Test full deployment pipeline with sample Node.js projects:
   - Simple Express app
   - NestJS application
   - Next.js application
   - TypeScript project

## Files to Create/Modify Summary

### New Files:
- `oci-devops/resources/oci/Dockerfile.nodejs.handlebars`
- `oci-devops/resources/oci/nodejs_build_spec.yaml.handlebars`
- `oci-devops/resources/oci/docker_nodejs_build_spec.yaml.handlebars`
- `tests/test-projects/nodejs/` (test fixtures)

### Modified Files:
- `oci-devops/src/projectUtils.ts` (project detection, build commands, artifacts)
- `oci-devops/src/oci/deployUtils.ts` (deployment logic)
- `oci-devops/src/oci/ociResources.ts` (resource templates)
- `common/src/dialogs.ts` (ProjectType enum)
- `oci-devops/README.MD` (documentation)

## Estimated Complexity

- **Low Complexity**: Project type detection, build command generation
- **Medium Complexity**: Dockerfile templates, build specs
- **High Complexity**: Full deployment pipeline integration, OKE configuration

## Notes

- The current architecture is heavily Java-centric, relying on Apache NetBeans Language Server
- Node.js support will be more straightforward as it doesn't require the same level of language server integration
- Consider creating a plugin/extension architecture to make adding new project types easier in the future
- Node.js projects may have different deployment patterns (serverless, static sites, etc.) that may need separate handling

