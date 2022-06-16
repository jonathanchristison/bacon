---
title: "PiG"
---

* Contents
{:toc}

# Introduction

PiG allows users to setup PNC with the product, build configurations, and much more via a YAML file. This makes it much easier to configure PNC for new releases as well as making the configuration portable across different PNC servers.

# Usage
PiG is run in multiple phases:

- configure
- build
- repo
- licenses
- javadocs
- sources
- shared-content
- docs
- scripts
- addons

The all-in-one phase that combines all the phases is:

- run

The `configure` phase pushes all the settings in the YAML file to PNC, and the `build` phase tells PNC to build everything.

```bash
bacon pig <phase> ...
```


# Configuration

The application is configured via the `build-config.yaml` file.

A PiG `build-config.yaml` looks like [this](https://github.com/project-ncl/bacon/blob/main/example-pig-config.yaml). See [this](build-config.html) for a detailed guide of `build-config.yaml`

# Operation

In all of the operations above, you can override the variables in your `build-config.yaml` by specifying the `-e` or `--env` flag. More details [here](build-config.html#usage-of-yaml-variables)

## Configure PNC Entities

Usage:
```bash
bacon pig configure <directrory containing build-config.yaml>
```

## Building

Usage:
```bash
bacon pig build <directrory containing build-config.yaml>
```

You can specify if you want temporary builds or not with the `-t` flag.

## Deliverables

This section covers the required deliverables for a product release:

- licenses
- maven-repository
- src zip
- javadoc zip

<table bgcolor="#ffff00">
<tr>
<td>
    <b>TODO</b> describe each section more
</td>
</tr>
</table>

### Maven Repository

You can generate a repository by adding a `repositoryGeneration` section to your `build-config.yaml`. The intended generation behaviour can be configured using the appropriate strategy. You can also optionally explicitly include or exclude artifacts based on a pattern.

A simple example configuration is:

```
repositoryGeneration:
  strategy: BUILD_GROUP
```

The above will build a repository based on the PNC build group. For a more detailed explanation of the various strategies for the maven repository generation, see [here](build-config.html#maven-repository-generation)


## Add ons

### Offliner Manifest

PiG is able to generate an offliner manifest in plain text compatible with the offliner tool. 
You can find more information about the tool [here](https://release-engineering.github.io/offliner/). 
This also takes into account any explicit inclusions or exclusions you may have specified as part of your `repositoryGeneration` step.
When the repository generation is set to BUILD_GROUP or to IGNORE then the `excludeSourceBuilds` optional parameter will be used, 
if it is defined, to exclude the builds specified with it.

Adding the following to the bottom of your `build-config.yaml` will generate `offliner.txt` inside the target directory.

```
addons:
  offlineManifestGenerator:
    offlineManifestFileName: offliner.txt

```

### Vertx Artifact Finder

This Add-on automates the process of creating vertx artifacts list which are used in Quarkus. 

Now, in order to create a list of vertx artifacts, you don't need to manually search for it in the project, this addon will do the work for you.

Adding the following to the bottom of your `build-config.yaml` will generate `vertxList.txt` inside the target directory.

```
addons:
   vertxArtifactFinder:

```

### Build From Source Statistics

The `runtimeDependenciesToAlignTree` add-on produces statistics on the percentage of dependencies that are build-from-source compliant and provides a reverse-mapping of dependencies to all of the parents in the tree which brings that dependency in.

```
addons:
   runtimeDependenciesToAlignTree:

```

The `runtimeDependenciesToAlignTree` has optional logging for sorted lists by project of the unique dependencies of that project.    To generate these:

```
addons:
   runtimeDependenciesToAlignTree:
       printProjectLogs: true
```

For each project in the build configuration, a log will be generated within the extras folder.


### Extra Deliverables Downloader

The `extraDeliverablesDownloader` add-on downloads artifacts from one or more PiG build configurations and uploads it to your projects staging directory. Each build configuration accepts two yaml variables: `matching` and `suffix` which should be provided together as a list. 

`matching` is a regex of an artifact that is to be downloaded and `suffix` is the name of the artifact you want to give it.
The final name is created by concatenating `{{project-name}}-{{version}}.{{milestone}}-{{suffix}}`

Make sure to give all artifacts distinct names because two artifacts with the same name can't be uploaded.

An example configuration is:
```
addons:
  extraDeliverablesDownloader:
    {{keycloakParentBuildConfigName}}:
        - matching: 'keycloak-server-dist.*\.zip'
          suffix: 'server-dist.zip'
        - matching: 'keycloak-api-docs.*\.zip'
          suffix: 'api-docs.zip'
    {{keycloakConnectBuildConfigName}}:
        - matching: 'keycloak-connect.*\.zip'
          suffix: 'nodejs-adapter-dist.zip'
```

## PiG Export Feature

### Build Configuration Export
PiG allows you to export an existing build-configuration into YAML format that can then be included into the builds section in a `build-config.yaml`.

The command to generate the YAML output is:
```
$ bacon pig export build-config <build-config-id>
```

### Post Build Product Security Scanning as a Service (PSSaaS)

The `postBuildScanService` addon provides a way to trigger scan for successfully finished builds. Currently PSSaaS supports three cases:
```
PNC build
Brew build
scmURLs of product which is not productized (not built in PNC or Brew)
```

The following command relies on mandatory configuration parameters to be defined directly in the build-config.yaml file (otherwise, bacon addons run will fail due to lack of mandatory parameter). Optional parameters may or may not be defined. Such an approach is impractical due to exposure of sensitive information.
```
bacon pig run --skipBuilds --skipJavadoc --skipLicenses --skipPncUpdate --skipSharedContent --skipSources --verbose .
```

Hiding Sensitive Information
----------------------------
In order to protect sensitive information from being exposed invoke bacon command the following way:
```
bacon pig run --skipBuilds --skipJavadoc --skipLicenses --skipSharedContent --skipSources \
-e pssaas_scan_service_url=${scan_service_url}                                            \
-e pssaas_secret_key=${scan_service_secret_key}                                           \
-e pssaas_secret_value=${scan_service_secret_val}                                         \
--verbose .
```
Make sure all involved environment variables have been defined before running the above command.
```
export scan_service_url=<token1>
export scan_service_secret_key=<token2>
export scan_service_secret_value=<token3>
```
Use parameters passed via '-e' bacon CLI option inside the build-config.yaml file:
```
serviceUrl: {{ pssaas_scan_service_url }}
serviceSecretKey: {{ pssaas_secret_key }}
serviceSecretValue: {{ pssaas_secret_value }}
```

Sensitive information does not leak anymore. What has changed? Suppliying parameters via command line option '-e' protects sensitive information by hiding actual values in the environment variables. This technique can be used to conceal secret parameter values and assign them to so-called 'bridge' variables which are passed-in to the build-config.yaml where they get interpreted and assigned to the ultimate addon configuration parameters (in the above example - serviceUrl, serviceSecretKey, serviceSecretValue).

There is no need to do the same for non-sensitive parameters like productID, eventID, etc. They can be defined directly in the build-config.yaml (see examples below).

Example 1: Brew build scan 
```
addons:
  postBuildScanService:
    productID: 2151
    eventId: 1
    isManagedService: True
    cpaasVersion: 99.0.0
    jobUrl: https://somejenkinspipeline.com/run999
    serviceUrl: {{ pssaas_scan_service_url }}
    serviceSecretKey: {{ pssaas_secret_key }}
    serviceSecretValue: {{ pssaas_secret_value }}
    brewBuilds:
      - 1000
      - 2000
```
      
Example 2: scmURLs based scan
```
addons:
  postBuildScanService:
    productID: 2151
    eventId: 1
    isManagedService: True
    cpaasVersion: 99.0.0
    jobUrl: https://somejenkinspipeline.com/run999
    serviceUrl: {{ pssaas_scan_service_url }}
    serviceSecretKey: {{ pssaas_secret_key }}
    serviceSecretValue: {{ pssaas_secret_value }}
    extraScmUrls:
      - scmUrl: https://github.com/randomurl
        scmRevision: 10.0.0
      - scmUrl: https://github.com/anotherrandomurl
        scmRevision: foobar
```

Using Defaults
--------------
To use default value for an optional parameter leave that parameter undefined in the addons section of your build-config.yaml file.
