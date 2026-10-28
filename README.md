# Reproduction Package: Alternate Data Stream Injection in Azure.Storage.DataMovement

This repository contains the reproduction material for a vulnerability found in
`Azure.Storage.DataMovement.Blobs`, specifically in the public method
`BlobContainerClient.DownloadToDirectoryAsync`. The issue arises because the
library builds the local destination path for a downloaded blob directly from
the blob's name, without checking for characters that carry a different
meaning on the destination filesystem.

On Windows, the colon character separates a file from one of its NTFS
Alternate Data Streams. Azure Blob Storage, unlike most local filesystems,
treats the colon as an ordinary character and allows it almost anywhere in a
blob name, which was confirmed against a live, non-emulator storage account
before any of the tests below were written. Taken together, these two facts
mean that a blob named, for instance, `legituser.exe:Zone.Identifier` is not
written as a new file when downloaded. It is written as a hidden stream
attached to a file named `legituser.exe` that may already exist on the
victim's machine.

The `Zone.Identifier` stream happens to be where Windows stores the Mark of
the Web, the marker that SmartScreen, Office Protected View, and several
other security features rely on to decide whether a file came from the
internet. An attacker who can write even one blob into a container that the
victim later syncs or downloads can therefore attach or overwrite that marker
on files the victim already has, without ever touching the file's actual
content.

## What is in this repository

The workflow at `.github/workflows/ads-poc.yml` runs on a genuine Windows
machine, a `windows-latest` GitHub Actions runner, rather than a local
simulation. It exercises two scenarios against a real Azure Storage account.

The first scenario downloads a container that includes a blob named
`probe.txt:simple-ads.bin`. No special options are set. The test confirms
that a regular file `probe.txt` is created and that its alternate data
stream contains the attacker-supplied content, which demonstrates that this
particular behavior requires nothing beyond default usage of the SDK.

The second scenario is closer to how an attack would actually unfold. A file
named `legituser.exe` is created on the runner beforehand, with its
`Zone.Identifier` stream deliberately set to `ZoneId=3`, the value Windows
assigns to content from the internet. This stands in for a file the victim
already downloaded and that Windows is already treating with appropriate
suspicion. The same container is then downloaded again, this time with
`CreationMode` set to `OverwriteIfExists`, a setting commonly used by sync,
backup, and CI artifact tools. After the download, the file's
`Zone.Identifier` stream is read back and found to contain `ZoneId=0`, the
value Windows assigns to locally trusted content. The marker that should have
kept this file under scrutiny has been silently replaced.

Both outcomes are checked programmatically rather than read by eye, and the
workflow prints a clear pass or fail line for each one along with a final
summary.

## Reproducing the result

Fork or clone this repository, add a repository secret named
`AZURE_STORAGE_CONNECTION_STRING` pointing at any Azure Storage account, and
create a container named `test-container` holding two blobs: `probe.txt:simple-ads.bin`
with arbitrary content, and `legituser.exe:Zone.Identifier` with the content
`[ZoneTransfer]\r\nZoneId=0\r\n`. Running the `ADS PoC` workflow, either from
the Actions tab or with `gh workflow run ads-poc.yml`, reproduces both
scenarios end to end.

A confirmed run, executed under this account on a freshly created storage
account with no prior state, is available here:
https://github.com/mohamadpishdar/ads-poc/actions/runs/40713802168

## Relationship to a prior report on the same package

This finding is unrelated to an earlier report on the same package
concerning path traversal through ".." in blob names. That earlier issue
could not be reproduced against a production storage account, since Azure's
HTTP transport layer rejects such names before they reach the service. The
present issue does not rely on ".." at all. It relies on the colon, a
character Azure Blob Storage was already confirmed to accept, which is why
the two findings needed to be investigated and reported separately.
