# Transitioning recording assets from language repositories into <https://github.com/Azure/azure-sdk-assets>

## Setting some context

The azure-sdk monorepos are growing quickly due to the presence of recordings. Due to this, the engineering system team has been tasked with providing a mechanism that allows recordings to live _elsewhere_. The actual implementation of this goal is already present within the `test-proxy` tool, and this document reflects how to TRANSITION to storing recordings elsewhere!

The script `generate-assets-json.ps1` will execute the initial migration of your recordings from within a language repo to the [assets repo](https://github.com/Azure/azure-sdk-assets) as well as creating the assets.json file for those assets.

The script is [generate-assets-json.ps1](https://github.com/Azure/azure-sdk-tools/blob/main/tools/test-proxy/transition-scripts/generate-assets-json.ps1)

## Setup

Before running the script, understand that **only services that have migrated to use the `test-proxy` as their record/playback solution can store recordings into the external assets repository.** The CLI command commands are invoked from the `test-proxy` itself, so if it is NOT being used for record/playback, the work to transition to the `test-proxy` must be done before recordings can be moved.

Running the script requires the following:

- [x] The targeted library is already migrated to use the test-proxy.
- [x] Git version `>2.25.0` needs to be on the machine and in the path. Git is used by the script and test-proxy.
- [x] Test-proxy needs to be on the machine and in the path. Instructions for that are [here](https://github.com/Azure/azure-sdk-tools/blob/main/tools/test-proxy/Azure.Sdk.Tools.TestProxy/README.md#installation). The script uses test-proxy's CLI commands.
- [x] [Powershell Core](https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell?view=powershell-7.2) at least version 7.

## Permissions

Check your github group membership. If you are part of the group `azure-sdk-write` directly or through a sub-team, you have the necessary permissions to create tags in the assets repository.

You will not be able to clean them up however. There exists [planned work](https://github.com/Azure/azure-sdk-tools/issues/4298) to clean up unused assets repo tags. Erroneously pushed tags will be auto cleaned.

## Nomenclature

- `language` repo - An individual language repository eg. azure-sdk-for-python or azure-sdk-for-net etc.
- `assets` repo - The repository where assets are being moved to. <https://github.com/Azure/azure-sdk-assets>

The `test-proxy` tool is integrated with the ability to automatically restore these assets. This process is kick-started by the presence of an `assets.json` alongside a dev's actual code. This means that while assets will be cloned down externally, the _map_ to those assets will be stored alongside the tests. Normally, it is recommended to create an `assets.json` under the path `sdk/<ServiceDirectory>`. However, more granular storage is also possible.

Service/Package-Level examples:

- `sdk/storage/assets.json`
- `sdk/storage/azure-storage-file-datalake/assets.json`

The location of the actual test code is referred to as the `language repo`.

The location of the automatically restored assets is colloquially referred to as the `assets repo`. There is an individual `assets repo` cloned for **each `assets.json` in the language repo.**

## Running the script

[generate-assets-json.ps1](https://github.com/Azure/azure-sdk-tools/blob/main/tools/test-proxy/transition-scripts/generate-assets-json.ps1) is a standalone powershell script with no supporting script requirements. The easiest way to run the script would be to pull the <https://github.com/Azure/azure-sdk-tools> repository locally and run the script directly from there.

```powershell
cd "<target-language-repo>/sdk/<service>"
<language-repo-root>/tools/test-proxy/transition-scripts/generate-assets-json.ps1
```

The script needs to be executed inside an `sdk/<ServiceDirectory>` or deeper, in an up to date language repository. A good rule here would be look at where the ci.yml is for an service directory. In the case where each library for a given service directory has their own pipelines, at the `sdk/<ServiceDirectory><Library>` level, it is recommended that the assets.json is created there. If the `ci.yml` exists deeper then the `sdk/<ServiceDirectory>/<Library>` level, then it is recommended to run the script from that directory.

```powershell
# in practice, given local clones of azure-sdk-for-java and azure-sdk-tools
cd c:/src/azure-sdk-for-java/sdk/attestation
c:/src/azure-sdk-tools/tools/test-proxy/transition-scripts/generate-assets-json.ps1 -InitialPush $true
```

After running a script, executing a `git status` from within the language repo, where the script was invoked from, will reflect two primary results:

- A new `assets.json` present in the directory from which they invoked the transition script.
- A **bunch** of deleted files from where their recordings _were_ before they were pushed to the assets repo.

Running the script without the `-InitialPush` option will just create the assets.json with an empty tag. No data movement.

### What's the script doing behind the scenes?

Given the previous example of `sdk/attestation` transition script invocation, users should see the following:

- Creation of the assets.json file in the `sdk/attestation` directory.
  - If `-InitialPush $true` has not been set, the script stops here and exits.
- A temp directory is created and the test-proxy's CLI restore is called on the current assets.json. Since there's nothing there, it'll just initialize an empty assets directory.
- The recordings are moved from their initial directories within the language repo into a temp directory that was created in the previous step.
  - The relative paths from root are preserved.
  - For example, the recordings for `C:/src/azure-sdk-for-python/sdk/tables` live in the `azure-data-tables/tests/recordings` subdirectory and in the target repository they'll live in `python/sdk/tables/azure-data-tables/tests/recordings`. All the azure-sdk supported languages will leverage [Azure/azure-sdk-assets](https://github.com/Azure/azure-sdk-assets), so adding a prefix to the output path `python` ensures that these recordings can live alongside others in the assets repo.
- Call `test-proxy push` on the assets.json created in the first step. The push will happen automatically and not require a manual PR.
  - On completion of the push, the newly created tag will be stamped into the assets.json.

At this point the script is complete. The assets.json and deleted recording files will need to be pushed into the language repository as a manual PR.

#### Why does the script analyze the remotes to compute the language?

This is necessary because the language is used in several places.

1. The AssetsRepoPrefixPath in assets.json is set to the language.
2. The TagPrefix is set to the `<language>/<ServiceDirectory>` or `<language>/<ServiceDirectory>/<Library>` etc.
3. The language also used to determine what the [recording directories within a repository are named](https://github.com/Azure/azure-sdk-tools/blob/main/tools/test-proxy/transition-scripts/generate-assets-json.ps1#L47).

## A final note about the initial push

If a directory with several thousand recordings is being migrated, the move and the initial push can take several minutes. For example, java storage recordings were used as a stress test. There are 4,693 files, with a combined size of 666 MB, and the initial push took about 7 minutes. This is a one time cost as the files do not exist yet within the assets repository. Subsequent pushes should have dramatically reduced push time.
