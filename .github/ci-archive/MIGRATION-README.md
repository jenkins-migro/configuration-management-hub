# Jenkins to GitHub Actions Migration Report

## Summary

Migrated four Jenkins pipeline definitions to GitHub Actions workflows and archived the original Jenkinsfiles under `.github/ci-archive/`.

## Source Pipelines

| Original Jenkins file | Pipeline type | Migrated workflow | Notes |
| --- | --- | --- | --- |
| `Jenkinsfile` | Declarative pipeline with `acp3_shared_library` | `.github/workflows/configuration-management.yml` | Preserves control-flow parameters, stage ordering, parallel package-generation gates, and environment values visible in the Jenkinsfile. Less-used Jenkins form parameters are accepted through `additional_parameters_json` to stay within GitHub's 25-input `workflow_dispatch` limit. Shared-library implementations were not present in this repository, so the workflow contains explicit replacement points for each `RunRoutine` call. |
| `sonarqubegradle/Jenkinsfile` | Scripted pipeline | `.github/workflows/sonarqube-gradle.yml` | Converts checkout, Gradle build/test/package, SonarQube analysis, artifact upload, Docker build, and conditional Docker push. |
| `gitoperations/Jenkinsfile` | Scripted pipeline | `.github/workflows/git-operations.yml` | Converts checkout, Git metadata capture, Maven build/test, release tagging on `master`, optional version update, release notes, and artifacts. |
| `toolsagents/Jenkinsfile` | Declarative pipeline | `.github/workflows/tools-agents.yml` | Converts stage-level agents/tools to separate jobs with JDK setup, build/test artifacts, and conditional SonarQube analysis. |

## Archived Files

The original Jenkins pipeline files were moved to:

- `.github/ci-archive/Jenkinsfile`
- `.github/ci-archive/sonarqubegradle/Jenkinsfile`
- `.github/ci-archive/gitoperations/Jenkinsfile`
- `.github/ci-archive/toolsagents/Jenkinsfile`

## Required Secrets and Variables

| Name | Type | Used by | Jenkins source | Purpose |
| --- | --- | --- | --- | --- |
| `SONAR_TOKEN` | Secret | `sonarqube-gradle.yml`, `tools-agents.yml` | `withSonarQubeEnv('SonarQube')`, `credentials('sonar-token')` | Authenticates SonarQube scans. |
| `SONAR_HOST_URL` | Repository variable | `sonarqube-gradle.yml` | Jenkins SonarQube server named `SonarQube` | SonarQube server URL. |
| `REGISTRY_USERNAME` | Secret | `sonarqube-gradle.yml` | `docker-registry-credentials` username | Docker registry login. |
| `REGISTRY_PASSWORD` | Secret | `sonarqube-gradle.yml` | `docker-registry-credentials` password | Docker registry login. |
| `GITHUB_TOKEN` | Built-in secret | `git-operations.yml` | `git-credentials` | Pushes release tags and optional version commits. |

## Shared Library Migration Notes

The root Jenkinsfile imports `acp3_shared_library` classes including `CredentialManager`, `SetInputConfigMap`, `SoftwareClusterFilesystemGenerator`, `ImageCreationEnvironmentSetUp`, `SoftwareClusterImageGenerator`, `SoCImageDownloader`, `SoCDevPackageAssembler`, `SoftwareClusterFilesystemInjector`, `ProdPackageInputGenerator`, `DetermineSpuSocVersion`, production package generators, `UCMDbGenerator`, and `CalProdPackager`.

Those shared-library source files are not present in this repository, so there was no code available to expand inline. The migrated root workflow preserves every visible stage and condition and includes an explicit step for each missing `RunRoutine` implementation. To complete functional parity, port the corresponding shared-library Groovy logic into repository scripts or replace each placeholder step with the supported command-line implementation.

## Action Security

All third-party workflow actions are first-party GitHub-maintained actions pinned to commit SHAs resolved from stable `v4` tags:

- `actions/checkout@11d5960a326750d5838078e36cf38b85af677262` (`v4`)
- `actions/setup-java@cf277c60eb25467037889841efdb72551f06f6c3` (`v4`)
- `actions/upload-artifact@ea165f8d65b6e75b540449e92b4886f43607fa02` (`v4`)
- `actions/download-artifact@d3f86a106a0bac45b974a628896c90dbdf5c8093` (`v4`)

## Validation

- Workflow syntax should be validated with `actionlint`.
- No repository build or test suite exists outside the migrated CI definitions, so no application tests were added.
