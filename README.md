# Graal Development Kit for Micronaut&reg; VSCode extensions

This suite provides extensions to Visual Studio Code that supports development of Graal Development Kit for Micronaut (GDK) and Micronaut applications, OCI DevOps support and JDBC support for Oracle Database and MySQL for VSCode.

The extensions are Technology Preview.

## Features

### OCI DevOps Extension

The OCI DevOps extension now supports **Node.js applications** in addition to Java/Micronaut applications. You can:

* Deploy Node.js applications to Oracle Cloud Infrastructure (OCI) DevOps
* Build Node.js Docker container images with automatic package.json validation
* Deploy Node.js containers to Oracle Kubernetes Engine (OKE)
* Automatically handle invalid package names and missing build scripts

### Continuous Integration

This repository includes GitHub Actions workflows for automated building and testing:

* **VSIX Extension Build**: Automatically builds the extension package on push/PR
* **Docker Image Testing**: Tests Node.js Docker image builds with invalid package names
* **Container Runtime Testing**: Validates that built containers start correctly and respond to HTTP requests
* **Artifact Upload**: VSIX packages are available as downloadable artifacts

The workflow is located at `.github/workflows/build-and-test-vsix.yml` and runs automatically on changes to the `oci-devops/` or `common/` directories.

## Build VSIX package from sources

To build VSIX packages of the GDK extensions, take the following steps:
* Install `npm`
  * Install `vsce` if not installed automatically 
* To build all extensions at once run following command at this top level folder.
```bash
$ npm run build
```
* Build each extension separately, e.g. **micronaut** ext:
```bash
npm run build:micronaut
```
Specify name of the folder with extension after **`:`**

## Installation

To install the extension into Visual Studio Code, take the following step:

* Install desired package with `code --install-extension <extension.vsix>`
```bash
code --install-extension micronaut/micronat-*.vsix
```
## Contributing

This project welcomes contributions from the community. Before submitting a pull request, please [review our contribution guide](./CONTRIBUTING.md)

## Security

Please consult the [security guide](./SECURITY.md) for our responsible security vulnerability disclosure process

## License

Copyright (c) 2019, 2024, Oracle and/or its affiliates. All rights reserved.

Released under the Universal Permissive License v1.0 as shown at
<https://oss.oracle.com/licenses/upl/>.

----
Micronaut® is a registered trademark of Object Computing, Inc. Use is for referential purposes and does not imply any endorsement or affiliation with any third-party product. Unauthorized use is strictly prohibited.