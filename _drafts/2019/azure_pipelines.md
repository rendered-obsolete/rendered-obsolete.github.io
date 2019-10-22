---
layout: post
title: Migrating from Travis/Appveyor to Azuze Pipelines
tags:
- azure
- devops
---

VS Code plugin
https://marketplace.visualstudio.com/items?itemName=ms-azure-devops.azure-pipelines

Multi-platform builds
https://docs.microsoft.com/en-us/azure/devops/pipelines/get-started-multiplatform

Available images
https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/hosted#use-a-microsoft-hosted-agent

Git options:
https://docs.microsoft.com/en-us/azure/devops/pipelines/repos/pipeline-options-for-git

Powershell download files
`Invoke-WebRequest` is the multi-platform option for Powershell Core.

Cross-platform scripts
https://docs.microsoft.com/en-us/azure/devops/pipelines/scripts/cross-platform-scripting

Trouble shooting
https://docs.microsoft.com/en-us/azure/devops/pipelines/troubleshooting
Set variables:
https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables
![](/assets/azure_run_pipeline_variables.png)


Pre-installed software

- [VS2017/Windows Server 2016](https://github.com/microsoft/azure-pipelines-image-generation/blob/master/images/win/Vs2017-Server2016-Readme.md)
- [VS2019/Windows Server 2019](https://github.com/microsoft/azure-pipelines-image-generation/blob/master/images/win/Vs2019-Server2019-Readme.md)
