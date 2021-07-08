## Getting started with Packer on Azure

### Purpose

The purpose of this document is to share my experience with establishing Packer as an image management tool on Azure. Note, based on JSON, not HCL2.

### Why Packer?

There are a lot of articles in support of this position. My rationale is simple. 

1. Automation. Most approaches to imaging remain manual in nature and Packer forces you to automate your build processes.

2. Cloud Agnostic. Packer is a multi-cloud product and thus, you learn it once and you've learned it for all the major cloud platforms. 

3. OS Agnostic. Whether you're building Windows or Linux based images, the approach is the same.

### Installing Packer for local development

You can gain access to packer a number of ways. The purpose of this article is not how to install packer. However, if you run Windows, there are several ways to use packer for local image development.

1. Use Azure Cloud Shell - OK, you don't need Windows for this. But in Azure Cloud Shell Packer is already loaded for you. Note, Cloud Shell may be a minor revision (1.6 vs. 1.7) behind what's currently available from Hashicorp. 

2. Use a package manager for Windows such as Chocolatey and the associated packer package
 
3. Download the pre-compiled binary (.exe) and update your path statement accordingly

For items #2 and #3 above, Hashicorp's website provides adequate installation instructions. To verify the version of Packer simply execute the following from the cmdline.

<pre lang="...">
> Packer --version
</pre>

If the above command lists the Packer version, you are ready to get started.

### Prerequisites

1. Azure subscription

2. Azure Credentials. Packer should authenticate to Azure using a Service Principal (aka App Registration) created in your Azure Active Directory Tenant. Note if you create a service principal the way I did there are two key points to understand.

<pre lang="...">
> New-AzADServicePrincipal -DisplayName "PackerSP$(Get-Random)"
</pre>

> - "*New-AzADServicePrincipal has the ability to assign a role to the service principal with the Role and Scope parameters. If both are omitted, the contributor role is assigned to the service principal. The default values for the Role and Scope parameters are Contributor for the current subscription*." 

The above is actually important because Packer creates a temporary resource group during the build process and it deletes it after the completion of the build. You can define a build resource group in the Packer build file but I didn't do that in my case so I'm not including it here. If you define a build resource group, you can apply an improved principal of least privledge.

> - *If your organization's Azure AD is configured with "Users can register applications" setting is set to No the service principal that creates the service principal should belong to the "Application Developer" Azure AD Role which grants the necessry rights to create an App Registration.*

3. A Resource Group. This is the target resource group for the managed image.

4. Identify the Marketplace image you want to deploy. This can be a little tedious. To identify the image you want to include in your Packer file you must do the following. (Complete the variables)

<pre lang="...">
Get-AzVMImagePublisher -Location $location
Get-AzVMImageOffer -Location $location -PublisherName $publisherName
Get-AzVMImageSku -Location $location -PublisherName $publiserName -Offer $offer
</pre>

Ultimately you will end up with something like this

> "image_publisher": "MicrosoftWindowsDesktop",

> "image_offer": "office-365",

> "image_sku": "21h1-evd-o365pp",

Pro-Tip. I find it a lot easier to provision a Gallery image from the portal and in the last stage of the provisioning workflow click **Download a template for automation** and review the ARM template for these details. See images below.

> ![GitHub Logo](images/packerImage1.png)


> ![GitHub Logo](images/packerImage2.png)

## The Packer build File

The requirements for the Packer Build file are here https://www.packer.io/docs/templates/legacy_json_templates

But let's decompose some key requirements.

### Variables

Optional, but as you would expect, variables add a ton of flexibility to your builds. One or more key/value strings that defines user variables contained in the template. If it is not specified, then no variables are defined. Here is an example.

<pre lang="...">
{
  "variables": {
    "client_id": "",
    "client_secret": "{{env `client_secret`}}",
    "tenant_id": "",
    "subscription_id": ""
},
</pre>

The variables will need to be populated during the build. There are serveral ways to do this.

### Builders

An array of one or more objects that defines the builders that will be used to create machine images for this template, and configures each of those builders. Said differently, the target platform for the Packer image...In this case Azure. Here is an example whereby we're deploying Windows 10 multi-session with Office 365 installed. 

<pre lang="...">
  "builders": [{
    "type": "azure-arm",

    "client_id": "{{user `client_id`}}",
    "client_secret": "{{user `client_secret`}}",
    "tenant_id": "{{user `tenant_id`}}",
    "subscription_id": "{{user `subscription_id`}}",

    "managed_image_resource_group_name": "myPackerGroup",
    "managed_image_name": "myWindowsPackerImage",

    "os_type": "Windows",
    "image_publisher": "MicrosoftWindowsDesktop",
    "image_offer": "office-365",
    "image_sku": "21h1-evd-o365pp",

    "communicator": "winrm",
    "winrm_use_ssl": true,
    "winrm_insecure": true,
    "winrm_timeout": "5m",
    "winrm_username": "packer",

    "azure_tags": {
        "dept": "Engineering",
        "task": "Image deployment"
    },

    "build_resource_group_name": "myPackerGroup",
    "vm_size": "Standard_D2_v2"
  }],
</pre>

### Provisioners

Optional, but powerful. An array of one or more objects that defines the provisioners that will be used to install and configure software for the machines created by each of the builders. This is how you add software to your build and additional customizations usually via shell commands. Here we run a bunch of inline powershell commands, including sysprep.exe.

<pre lang="...">
  "provisioners": [{
    "type": "powershell",
    "inline": [
      "while ((Get-Service RdAgent).Status -ne 'Running') { Start-Sleep -s 5 }",
      "while ((Get-Service WindowsAzureGuestAgent).Status -ne 'Running') { Start-Sleep -s 5 }",
      "& $env:SystemRoot\\System32\\Sysprep\\Sysprep.exe /oobe /generalize /quiet /quit",
      "while($true) { $imageState = Get-ItemProperty HKLM:\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Setup\\State | Select ImageState; if($imageState.ImageState -ne 'IMAGE_STATE_GENERALIZE_RESEAL_TO_OOBE') { Write-Output $imageState.ImageState; Start-Sleep -s 10  } else { break } }"
    ]
  }]
</pre>

## Running Packer

With no variables, it's pretty simple.

> packer build [template.json]

However, you should use variables. And if you do, there are several ways to execute packer build commands.

### Packer Build with Variable File

> packer build -var-file var.pkr.json windows10ms.json

(Note there is no equal sign (=) in the variable file parameter as packer build help file seems to indicate!)

### Packer Build with environment variables

Note if you are doing this stuff in vs code, run it in administrator mode first.

Set the environment variables. For example.

> setx client_secret [value]

You'll need to open a new command prompt to see the variable. For example.

> $env:client_secret

Now, in order ot use the environment variable the build file variable section must refect that. Refer to the variable example listed above but repeated here for emphasis.

>"client_secret": "{{env `client_secret`}}"

To use the client variable, simply run packer build. once again, if you are doing this from vs code, make sure you "run as administrator" or the environment variables can't be read from the script editor. 

> packer build [template.json]

### Packer Build with inline variables

This one is pretty simple too, you're just providing the variables as part of the command execution.

> packer build -var 'client_secret=foo' [template.json]

### Summary

That's it for now. See source code for file examples. 