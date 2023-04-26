# Feature Dependencies

Reference: https://github.com/devcontainers/spec/pull/208/files

An alternate and conflicting proposal can be found here: https://github.com/devcontainers/spec/pull/208/.  If the aforementioned proposal is accepted, this proposal should be withdrawn (and vice versa).

## Motivation

We've seen significant interest in the ability to "reuse" or "extend" a given Feature with one or more additional Features.  Often in software a given tool depends on another (or several) tool(s)/framework(s).  As the dev container Feature ecosystem has grown, there has been a growing need to reduce redundant code in Features by first installing a set of dependent Features.

## Goal

The solution shall provide a way to publish a Feature that "depends" on >= 1 other published Features. Dependent Features will be installed by the orchestrating tool following this specification.

The solution outlined shall not only execute the installation scripts, but also merge the additional development container config, as outlined in the documented [merging logic.](https://containers.dev/implementors/spec/#merge-logic).

The solution should provide guidelines for Feature authorship, and prefer solutions that enable simple authoring of Features, as well as easy consumption by end-users.

A non-goal is to require the use or implementation of a full-blown dependency management system (such as `npm` or `apt`).  The solution should allow Features to operate as "self-contained, shareable units of installation code and development container configuration"[(1)](https://containers.dev/implementors/features/). 

## Definitions

- **User Feature**.  Features defined explicitly by the user in the `features` object of `devcontainer.json`.
- **Published Feature**.  Features published to an OCI registry.

## Existing Solutions

See https://github.com/devcontainers/spec/pull/208/files#diff-a29ffaac693437b6fbf001066b97896f7aef4d6d37dc65b8b98b22a5e19f6c7aR26 for existing solutions and alternative approaches.

## Proposed Specification

### (A) Add `dependsOn` property to Feature metadata.

A new property `dependsOn` can be optionally added to the `devcontainer-feature.json`.  This property mirrors the `features` object in `devcontainer.json`.  Adding Feature(s) to this property tell the orchestrating tool to install the Feature(s) before installing the Feature that declares the dependency. 

The installation order is subject to the algorithm set forth in this document. Where there is ambiguity, it is up to the orchestrating tool to decide the order of installation.

| Property | Type | Description |
|----------|------|-------------|
| `dependsOn` | `string` | The ID of the Feature.  This follows the same semantics of the `id` property in the `devcontainer-feature.json` file. |

An example `devcontainer-feature.json` file with a dependency on three other published Features:

```json
{
    "name": "My Feature",
    "id": "myFeature",
    "version": "1.0.0",
    "dependsOn": {
        "ghcr.io/myotherFeature:1": {
            "flag": true
        },
        "features.azurecr.io/aThirdFeature:1": {},
        "features.azurecr.io/aFourthFeature:1.2.3": {}
    }
}
```

`myfeature` will be installed after `myotherFeature`, `aThirdFeature`, and `aFourthFeature`.

### (B) Add `dependsOn` metadata to Feature manifests

::TODO:: Also need to think about (1) tgz Features and (2) local Features


### (C) `dependsOn` install order algorithm

The orchestrating tool will calculate an installation order before as outlined below.

#### (C1) Building dependency graph

From the user-defined Features, the orchestrating tool will build a dependency graph.  The graph will be built by traversing the `dependsOn` property of each Feature and recursively resolving each Feature from the property.  An accumulator is maintained with each new Feature that has been discovered, as well as a pointer to its dependencies.  If the exact Feature (see **Feature Equality**) has already been added to the accumulator, it will not be added again.  The accumulator will be fed into (C2) after all the Feature tree has been resolved.

::TODO:: psuedocode.


#### (C2) Round-based sorting

Perform a sort on the result of (C1) in rounds. This sort will rearrange and deduplicate Features, producing a sorted list of Features to install.  The sort will be performed as follows: 

Start with all the elements from (C1) in a `worklist` and an empty list `installationOrder`.  While the `worklist` is not empty, iterate through each element in the `worklist` and check if all its dependencies (if any) are already members of `installationOrder`.  If the check is true, add it to an intermediate list `round` and remove it from the `worklist`.  If not, skip it.  Equality is determined in **Feature Equality**.


Before committing each `round` to the `installationOrder`, sort all elements of round according to **Round Stable Sort***.


Repeat for as many rounds as necessary until the worklist is empty.  If there is ever a round where no elements are added to `installationOrder`, the algorithm should terminate and return an error.  This indicates a circular dependency or other error in the dependency graph.


::TODO:: psuedocode.

#### Definition: Feature Equality

This specification defines two Features as equal in the strictest possible sense.  That is, the Feature's manifest (for OCI Features) must be identical and hash to the same value.  That implies that the tgz contents of the Feature and its entire `devcontainer-feature.json` are identical.  Additionally, each Feature's options are compared value by value.  If any of these conditions are not met, the Features are considered not equal.

::TODO:: tgz Features and local Features

#### Definition: Round Stable Sort

To prevent non-deterministic behavior, the algorithm will sort each round according to the following rules:

- Sort Features lexiographically by their resource Id (without version).  If those are equal:
- Sort Features from oldest to newest tag (latest being the "most new").  If those are equal:
- Sort Features by their options ::TODO:: If those are equal:
- Sort Features by their canonical name (Feature Id resolved to the digest hash).

If there is no difference based on these comparator rules, the Features are considered equal.

### (D) Modify existing Feature order installation algorithm

Two existing properties: `installsAfter` on the Feature metadata, and `overrideFeatureInstallationOrder` in the `devcontainer.json` both exist to alter the installation order of user-defined Features. ::TODO::