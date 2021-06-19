## Getting started with Packer on Azure

### Purpose

The purpose of this document is to share my experience with establishing Packer as an image management tool on Azure.

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

> #1 "*New-AzADServicePrincipal has the ability to assign a role to the service principal with the Role and Scope parameters. If both are omitted, the contributor role is assigned to the service principal. The default values for the Role and Scope parameters are Contributor for the current subscription*." 

The above is actually important because Packer creates a temporary resource group during the build process and it deletes it after the completion of the build. You can define a build resource group in the Packer build file but I didn't do that in my case so I'm not including it here. If you define a build resource group, you can apply an improved principal of least privledge.

> #2 *If your organization's Azure AD is configured with "Users can register applications" setting is set to No the service principal that creates the service principal should belong to the "Application Developer" Azure AD Role which grants the necessry rights to create an App Registration.*

3. A Resource Group. This is the target resource group for the managed image.

4. Identify the Marketplace image you want to deploy. This can be a little tedious. To identify the image you want to include in your Packer file you must do the following.

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

image1

image2



