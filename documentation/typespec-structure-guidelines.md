- [Repository Guidelines for TypeSpec](#repository-guidelines-for-typespec)
  - [Purpose](#purpose)
  - [Repository](#repository)
  - [Structure Overview](#structure-overview)
  - [Service Folders](#service-folders)
    - [Packages](#packages)
    - [Structure](#structure)
  - [Service Family Libraries](#service-family-libraries)
  - [Sample Project](#sample-project)
- [Spec Versioning](#spec-versioning)
    - [Service Folder](#service-folder)
    - [Working in Feature Branches](#working-in-feature-branches)
    - [Publishing Specs](#publishing-specs)

# Repository Guidelines for TypeSpec

## Purpose

We need to formulate a strategy for checking in actual TypeSpec service specs for Azure. Service teams need to know where
and how to check these in and tooling needs to know how to find and consume them. While OpenAPI specs traditionally have been just spec
documents, TypeSpec "specs" could include service-specific functions, decorators and so forth that elevate them to the level of a
library, and to maximize the benefits of TypeSpec, this approach needs to be supported as a first-class citizen.

## Repository

TypeSpec can co-exist with Swagger within the existing `azure-rest-api-specs` repository. This approach will make it easier to generate Swagger artifacts without needing to sync between repos at the downside of having to live with any baggage associated with the repo as it ages. There are currently over 1,000 issues and 500 open PRs in this repo.

## Structure Overview

This proposal strives to align with [Azure SDK Repo Structure Guidelines](https://azure.github.io/azure-sdk/policies_repostructure.html).

We eliminate the distinction between data-plane and resource-manager, treating all packages as siblings under the service family folder. This is what the SDKs do today. The legacy `data-plane` and `resource-manager` folders where Swagger specs live will remain.

A given TypeSpec folder could represent any of the following scenarios:

- an SDK and service spec (simplest, common for small services)
- a service spec only
- an SDK spec that aggregates many small service specs (one SDK to many microservices)
- an SDK spec that extracts a portion of a large service spec (many SDKs to a single large service)
- a shared library that represents neither a service spec nor an SDK

Examples below:

```
-> specification
   -> confidentialledger
      -> ConfidentialLedger               (data-plane SDK + service)
      -> data-plane                       (swagger)
      -> resource-manager                 (swagger)
   -> compute
      -> Compute.Management               (mgmt SDK + service)
      -> Compute.Management.Shared        (shared)
      -> Compute.CloudService.Management  (service)
      -> Compute.Diagnostic.Management    (service)
      -> Compute.Disk.Management          (service)
      -> Compute.Gallery.Management       (service)
      -> Compute.Skus.Management          (service)
      -> data-plane                       (swagger)
      -> resource-manager                 (swagger)
   -> keyvault
      -> KeyVault.Certificates            (data-plane SDK + service)
      -> KeyVault.Keys                    (data-plane SDK + service)
      -> KeyVault.Secrets                 (data-plane SDK + service)
      -> KeyVault.Management              (mgmt SDK + service)
      -> data-plane                       (swagger)
      -> resource-manager                 (swagger)
```

## Service Folders

The service folder contains the entire TypeSpec library specification for a service, which could include custom linter rules, methods, etc.

The folder name should correspond to the RP-name, but dropping the leading "Azure" or "Microsoft" prefix. Two additional naming conventions should be following:

- `Management` at the end of the service RP-name indicates a management (resource manager) libarary.
- `Shared` at the end of the name indicates a shared library. If a management library has a shared component (unlikely), `Shared` should follow `Management`.

### Packages

All services and service family libraries are modeled as TypeSpec packages, since you must install supporting librarie for proper tooling support. Per this proposal, all packages defined in the repo would be **unpublished**. Only packages in `typespec-azure` would be published. 

All packages defined in this repo would use the `@typespec-api-spec` scope and use a lowercased, dashed form of the service namespace (ex: `@typespec-api-spec/azure-storage-blob`).

### Structure

Each package should have the following minimum structure:

- `typespec-project.yaml`
- `main.tsp` file
- Supporting `*.tsp` files
- `examples/` folder for example JSON files

Authors may use folders as desired for organizing TypeSpec files.

Additionally, packages which wish to define custom linter rules or otherwise use TypeScript must place any `*.ts` files under a `src/` folder. Within the `src/` folder, the author may use as many subfolders as desired for organizing code.

To distinguish between folders which define a service, an SDK, or both, one can look to the `typespec-project.yaml`.

- SDKs will take dependencies on the `@azure-tools/typespec-dpg` library, as well as SDK-specific emitters such as `@azure-tools/typespec-python` and configure their options within `typespec-project.yaml` but the emitter version should not be configured within `typespec-project.yaml`, and it should be configured globally in SDK repo instead.
- SDKs _may_ have a sidecar file to customize how the SDK is shaped. Folders that describe service definitions only will not have a sidecar file. _Note: the absence of a sidecar does not mean that a folder does not describe an SDK, but the presence of one means it is an SDK._
- Services will take a dependency on `@azure-tools/autorest` and configure the emitter in `typespec-project.yaml`.
- Services should **not** have a `package.json` directly in the TypeSpec directory as they all should be using the global `package.json` in the root specification folder for installing any dependencies needed. 

## Service Family Libraries

The service family library concept allows a family of services to share common models, linter rules, templates, etc.

Service libraries can reference unpublished service _family_ libraries via relative path import in `*.tsp`:

```typespec
import "../Contoso.WidgetManager.Shared";
```

While this would permit services from importing any service library described in the specs repo, as a matter of policy we should probably avoid that and have tooling to detect this scenario. Service family libraries **should** use versioning decorators and spec packages should reference them as versioned dependencies. Tooling would need to ensure that changes to service family libraries does not result in unexpected changes to any service version. One way to do this would be to diff the projection of the service versions on the `main` branch against the projection of the service versions that result from the change.

We treat the shared library as a sibling with other packages within the service family. This is similar to what we currently do for services that have a "Shared" package and would allow an arbitrary number of shared packages. A shared library folder should not contain `typespec-project.yaml` as it's not requried to be released. See [Sample Project](#sample-project) for reference.

```
-> specification
   -> contosowidgetmanager
      -> Contoso.WidgetManager            (data-plane)
      -> Contoso.WidgetManager.Management (management)
      -> Contoso.WidgetManager.Shared     (shared)
```

Here's an example of how Cognitive Services might use multiple shared libraries:

```
-> specification
   -> cognitiveservices
      -> Language.TextAnalytics  (data-plane)
      -> Language.QnA            (data-plane)
      -> Language.Shared         (shared)
      -> Vision.ComputerVision   (data-plane)
      -> Vision.CustomVision     (data-plane)
      -> Vision.Shared           (shared)
```
## Sample Project

Here's a [typespec-sample-project](../specification/contosowidgetmanager) to demonstrate the files and folders supposed to be included when check in to this `azure-rest-api-specs` repository.

# Spec Versioning

TypeSpec has a versioning library which allows a single spec to represent multiple versions through projections. Service teams have GA service versions as well as public and private Preview service versions. The versioning library is currently optimized for the kinds of changes allowed between GA service releases: long-lived, stable, backward compatible changes. Preview versions are shorter-lived and often have wildly breaking changes from one version to the next, for which the versioning library is not optimized.

### Service Folder

The service folder contains the TypeSpec files for the service package. Services transitioning to TypeSpec may simply begin by modeling their versioned TypeSpec from their latest stable Swagger. Existing services DO NOT need to model past service versions unless a business rationale exists. Future versions should be added to the spec using the necessary versioning decorators. The inital version in the spec need not feature version annotations since it is considered the baseline (i.e. it makes no implications about prior versions since it does not know about them).

### Working in Feature Branches

Feature branches enable diffing the proposed change directly against the main branch. Feature branches should be used for either GA or preview API version development. On the feature branch, the service team should directly modify the typespec within the service folder as this works well with GitHub's diffing strategy.

### Publishing Specs

The purest approach to publish a spec is to merge the PR that modifies the TypeSpec in the service folder. This is the simplest experience for projecting the TypeSpec and generating artifacts, including server-side codegen.

In the event that a major break makes it infeasible to continue using a spec with version annotations, the spec could be reset to some base version (likely the breaking one) and continue versioning from there. The commit hash or tag that represented the spec prior to the reset would need to be tracked in order to regenerate older versions of the spec. At this point, if an update was needed to an older version of the spec no longer represented on the latest commit on `main`, we would need a servicing branch and update the hash pointers to the commit on that servicing branch.

This option is only suitable for _public_ previews. Private previews should live solely in a branch (in either the public or private repo) until/unless they become a public preview.