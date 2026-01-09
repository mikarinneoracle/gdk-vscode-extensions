# Local Testing Guide for Node.js Support

This guide explains how to test the extension locally after adding Node.js support.

## Prerequisites

1. **Node.js and npm** installed (for building the extension)
2. **VS Code** installed
3. **TypeScript** (installed via npm)
4. **Git** (for version control)

## Quick Start: Testing in Development Mode

The easiest way to test your changes is using VS Code's built-in extension development host.

### Step 0: Remove Existing Extension (Important!)

**Before testing your development version, uninstall any existing OCI DevOps Tools extension:**

1. Open VS Code
2. Go to **Extensions** view (`Cmd+Shift+X` / `Ctrl+Shift+X`)
3. Search for "OCI DevOps Tools"
4. If installed, click **Uninstall**
5. Reload VS Code if prompted

**Why?** Having both the installed extension and the development version can cause:
- Conflicts between versions
- Confusion about which code is running
- Extension loading issues
- Cached extension data interfering with tests

**Note:** When using `F5` (Extension Development Host), VS Code loads your development extension, but if the marketplace version is also installed, there can still be conflicts. It's safest to uninstall first.

### Step 1: Build the Extension

From the root directory:

```bash
# Install dependencies and build common modules
npm run prebuild

# Build the oci-devops extension specifically
cd oci-devops
npm install
npm run compile
```

Or build all extensions:

```bash
# From root directory
npm run build:oci-devops
```

### Step 2: Open Extension in VS Code

1. Open the `oci-devops` folder in VS Code
2. Start debugging using one of these methods:
   - **Mac**: Press `Fn + F5` (or `F5` if function keys are enabled) OR use **Run > Start Debugging** from the menu OR press `Cmd+Shift+P` and type "Debug: Start Debugging"
   - **Windows/Linux**: Press `F5` OR use **Run > Start Debugging** from the menu
3. This will:
   - Compile the extension
   - Launch a new "Extension Development Host" window
   - Load your extension in that window

### Step 3: Test in Extension Development Host

In the new VS Code window (Extension Development Host):

1. **Open a test Node.js project folder**:
   - Create a simple Node.js project with `package.json`
   - Open that folder in the Extension Development Host

2. **Test project detection**:
   - Open the Command Palette (`Cmd+Shift+P` / `Ctrl+Shift+P`)
   - Look for OCI DevOps commands
   - Check if the project is detected as Node.js type

3. **Test build commands**:
   - Try creating an OCI DevOps project
   - Verify build commands are generated correctly

## Building VSIX Package for Manual Installation

To create a VSIX package you can install manually:

### Step 1: Build VSIX Package

```bash
# From root directory
npm run build:oci-devops

# Or from oci-devops directory
cd oci-devops
npm run build
```

This creates a `.vsix` file in the `oci-devops` directory.

### Step 2: Uninstall Existing Extension

**Important:** Uninstall any existing OCI DevOps Tools extension first:

1. Open VS Code
2. Go to **Extensions** view (`Cmd+Shift+X` / `Ctrl+Shift+X`)
3. Search for "OCI DevOps Tools"
4. Click **Uninstall**
5. Reload VS Code

### Step 3: Install VSIX Locally

```bash
# Install the extension from VSIX
code --install-extension oci-devops/oci-devops-*.vsix
```

### Step 4: Reload VS Code

- Press `Cmd+Shift+P` / `Ctrl+Shift+P`
- Run "Developer: Reload Window"

## Testing Workflow

### 1. Create a Test Node.js Project

Create a simple test project to verify detection:

```bash
mkdir test-nodejs-app
cd test-nodejs-app
npm init -y
```

Create a simple `package.json`:

```json
{
  "name": "test-nodejs-app",
  "version": "1.0.0",
  "description": "Test Node.js app for extension",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "build": "echo 'Building...'"
  }
}
```

Create `index.js`:

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello from Node.js!\n');
});

const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### 2. Test Project Detection

1. Open the test project in VS Code (with extension installed)
2. Check the OCI DevOps view
3. Verify the project is detected as `NodeJS` type
4. Check that build system is detected as `NPM`

### 3. Test Build Commands

1. Try to create an OCI DevOps project
2. Verify the generated build commands:
   - Should include `npm install` and `npm run build`
   - Should not include Maven/Gradle commands

### 4. Test Dockerfile Generation

1. Create an OCI DevOps project
2. Check the generated `.devops` folder
3. Verify `Dockerfile.nodejs` is created
4. Verify it uses port 3000 (not 8080)

### 5. Test Build Pipeline Specs

1. Check generated build specs in `.devops` folder
2. Verify `nodejs_build_spec.yaml` is created
3. Verify `docker_nodejs_build_spec.yaml` is created
4. Check that they reference Node.js installation steps

## Running Unit Tests

The project includes test infrastructure. To run tests:

### From oci-devops directory:

```bash
cd oci-devops

# Compile tests
npm run test-compile

# Run tests (requires xvfb on Linux)
npm run pre-test
npm run test
```

### From root directory:

```bash
# Run OCI DevOps tests
npm run tests:oci-devops
```

## Debugging Tips

### 1. Enable Extension Logging

Add to your VS Code settings (`.vscode/settings.json` in the extension folder):

```json
{
  "oci.devops.logLevel": "debug"
}
```

### 2. Check Extension Output

1. Open **View > Output**
2. Select "OCI DevOps Tools" from the dropdown
3. Check for error messages or debug logs

### 3. Use Console Logging

In your extension code, use:

```typescript
import * as logUtils from '../../common/lib/logUtils';

logUtils.logInfo('Your debug message');
logUtils.logError('Error message');
```

### 4. Debug Extension Code

1. Set breakpoints in your TypeScript files
2. Press `F5` to start debugging
3. The debugger will attach to the Extension Development Host

## Testing Specific Features

### Test Project Type Detection

Create a test file to verify detection logic:

```typescript
// In oci-devops/src/test/suite/projectUtils.test.ts
import * as projectUtils from '../../projectUtils';
import * as vscode from 'vscode';

suite('Node.js Project Detection', () => {
    test('Should detect Node.js project', async () => {
        const folder = vscode.workspace.workspaceFolders?.[0];
        if (folder) {
            const project = await projectUtils.getProjectFolder(folder);
            assert.strictEqual(project.projectType, 'NodeJS');
        }
    });
});
```

### Test Build Commands

```typescript
test('Should generate npm build command', async () => {
    const folder = {
        uri: vscode.Uri.file('/path/to/nodejs/project'),
        projectType: 'NodeJS' as const,
        buildSystem: 'NPM' as const,
        subprojects: []
    };
    const command = await projectUtils.getProjectBuildCommand(folder);
    assert(command?.includes('npm install'));
    assert(command?.includes('npm run build'));
});
```

## Manual Testing Checklist

After making changes, test these scenarios:

- [ ] **Project Detection**
  - [ ] Node.js project with `package.json` is detected
  - [ ] Project type is set to `NodeJS`
  - [ ] Build system is detected as `NPM` (or `Yarn`/`PNPM` if applicable)

- [ ] **Build Commands**
  - [ ] Build command includes `npm install`
  - [ ] Build command includes `npm run build`
  - [ ] Test flag is handled correctly

- [ ] **Artifact Detection**
  - [ ] Correct output directory is detected
  - [ ] Handles different frameworks (Express, NestJS, Next.js)

- [ ] **Dockerfile Generation**
  - [ ] `Dockerfile.nodejs` is created
  - [ ] Port is set to 3000 (not 8080)
  - [ ] Node.js version is correct
  - [ ] Entry point is correct

- [ ] **Build Pipeline Specs**
  - [ ] `nodejs_build_spec.yaml` is generated
  - [ ] `docker_nodejs_build_spec.yaml` is generated
  - [ ] Specs include Node.js installation steps

- [ ] **OCI DevOps Integration**
  - [ ] Can create OCI DevOps project for Node.js
  - [ ] Build pipelines are created correctly
  - [ ] Deployment pipelines are created correctly

## Common Issues and Solutions

### Issue: Extension not loading

**Solution**: 
- **First, uninstall any existing extension version**
- Check that `npm run compile` completed successfully
- Verify `dist/extension.js` exists
- Check the Developer Console for errors (Help > Toggle Developer Tools)
- In Extension Development Host, check if extension is enabled (Extensions view)

### Issue: Changes not reflected

**Solution**:
- Rebuild: `npm run compile` or `npm run build`
- Reload the Extension Development Host window
- Or reinstall the VSIX package

### Issue: TypeScript compilation errors

**Solution**:
- Run `npm run compile` to see all errors
- Fix TypeScript errors before testing
- Check `tsconfig.json` configuration

### Issue: Project not detected

**Solution**:
- Verify `package.json` exists in the project root
- Check the detection logic in `getProjectFolder()`
- Add debug logging to see what's being checked

## Continuous Development Workflow

For active development:

1. **Uninstall Existing Extension**: Remove any installed version first

2. **Watch Mode**: Use webpack watch mode for automatic recompilation:
   ```bash
   cd oci-devops
   npm run watch
   ```

3. **Extension Development Host**: Keep the Extension Development Host window open

4. **Test Project**: Keep a test Node.js project open in the Extension Development Host

5. **Iterate**: 
   - Make changes
   - Save files (webpack will recompile)
   - Reload Extension Development Host (`Cmd+R` / `Ctrl+R`)
   - Test your changes

## Extension Management Commands

Quick reference for managing extensions:

```bash
# List installed extensions
code --list-extensions | grep oci-devops

# Uninstall extension
code --uninstall-extension oracle-labs-graalvm.oci-devops

# Install from VSIX
code --install-extension oci-devops/oci-devops-*.vsix
```

## Testing with Real OCI Account

For full end-to-end testing (requires OCI account):

1. **Configure OCI**:
   - Set up `.oci/config` file
   - Configure API keys

2. **Test Deployment**:
   - Create OCI DevOps project
   - Run build pipelines
   - Deploy to OKE (if configured)

3. **Verify**:
   - Check OCI Console for created resources
   - Verify build logs
   - Check deployment status

## Next Steps

After local testing passes:

1. Run full test suite: `npm run tests:oci-devops`
2. Check for linting errors: `npm run lint`
3. Create test fixtures for Node.js projects
4. Add unit tests for new functionality
5. Update documentation

## Resources

- [VS Code Extension Development](https://code.visualstudio.com/api/get-started/your-first-extension)
- [Testing Extensions](https://code.visualstudio.com/api/working-with-extensions/testing-extension)
- [Extension Development Host](https://code.visualstudio.com/api/get-started/extension-anatomy#extension-development-host)

