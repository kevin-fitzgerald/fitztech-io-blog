---
title: "Deploying a Personal Blog With Hugo and Azure Static Web Apps"
date: 2022-07-17T14:25:38-05:00
tags: [azure, github, cloudflare, hugo, devops, 100DaysOfHomelab]
categories: [Guides]
draft: true
---
# Introduction
In this article, I outline how I deployed [fitztech.io](https://fitztech.io) using Hugo for site generation, Azure Static Web Apps for web hosting, Cloudflare for DNS, and Github for code hosting and automated deployments. Using these free technologies and services, it is easy to host and maitain a highly performant and economical personal blog, even if you don't have experience in web development. With a willingness to dive into documentation and build a basic familiarity with Markdown, HTML, and CSS, its easier than ever to carve out your slice of the internet, and even build some rudimentary devops skills. Before diving into the content of the guide, lets quickly review JAMstack and discuss the technologies I've selected for my site.

## What is JAMstack?
If you've done any research into personal blog hosting in recent years, you've likely encountered the term *JAMstack*. According to [Jamstack.org](https://jamstack.org/glossary/jamstack/), "Jamstack is an architectural approach that decouples the web experience layer from data and business logic, improving flexibility, scalability, performance, and maintainability." In practice, Jamstack typically involves the use of static site generators such as Hugo, Gatsby, and Jekyll to precompile HTML sites from markdown templates. These sites are then deployed to Jamstack hosting services like Netlify, Cloudflare Pages, or Vercel, which often offer serverless compute integrations for extending functionality through REST APIs. Finally, while not core to the Jamstack architecture, most Jamstack platforms offer CI/CD integrations with popular Git hosts like Github and Gitlab to automate site deployments and content updates.

## Why This Tech Stack?
### Hugo vs Jekyll vs Gatsby
Hugo, Jekyll, and Gatsby are probably the biggest names among static site generators suited for people not looking to become frontend web developers or designers. Each of these products are roughly equivalent in terms of functionality, and realistically none of them are a bad choice. I selected Hugo for the following reasons:
1. For uninteresting reasons, I had some minimal prior experience with Hugo.
2. Hugo has an abundance of community built themes. My design skills are very poor, so I wanted something that would look good more or less out of the box.
3. Go lang memes. :smirk:

### Azure Static Web Apps vs Cloudflare Pages
I'm currently focused on mastering the Azure platform for professional reasons, which ultimately is the reason I chose to use Azure Static Web Apps. Cloudflare Pages was my primary alternative, largely because I already host my DNS at Cloudflare. Cloudflare Pages also offers a superior free tier, as detailed in the table below. I never seriously reviewed or considered other competitors like Netlify or Vercel. If I was not already in Azure as my IaaS provider, I would likely have selected Cloudflare Pages.

| Feature               | Azure Static Web Apps    | Cloudflare Pages |
|-----------------------|--------------------------|------------------|
| Bandwidth             | 100 GB/month             | Unlimited        |
| Sites                 | 10                       | Unlimited        |
| Custom Domains        | 2                        | 10               |
| Storage               | 0.5 GB/app, 0.25 GB/file | 0.25 GB/file     |
| Max Concurrent Builds | Unlimited                | 1                |
| Max Monthly Builds    | Unlimited                | 500              |

### Github vs Azure Devops
While there is much speculation about the future of Azure Devops in light of Microsoft's acquisition of Github, I've generally not been overly concerned about ADO's prospects given it's current adoption level among enterprises. This project, however, proved a good example of one of the common fears brought up by ADO doom-sayers.

I am an Azure AD user and prefer to integrate SSO into applications whenever possible.  Azure Devops allows Azure AD sign-in integration at all license levels. Github, by contrast, only allows SSO with a Github Enterprise license, which costs $21/user/month at time of writing. As a result, I attempted to use my Azure Devops organization for this project initially, but quickly encountered problems setting up the automated deployment integration via the Azure Static Web Apps deployment wizard in the Azure Portal (I will go into more detail on why I used the Portal below). Despite my account being both a project and organization owner in Azure Devops, the deployment wizard for the web app failed to populate the repository selection dropdowns for Azure Devops, effectively breaking my ability to use the native Azure Devops integration. After doing more research, I found that the integration from Azure Static Web Apps to Azure Devops is fairly new, being released in May 2022, two months prior to this writing.  The Github integration, by contrast, has existed since Azure Static Web Apps released from public preview, and I was able to set up the integration to my personal Github account with little issue.

Many onlookers of the Github-Microsoft acquisition predicted that Github and ADO would continue side by side with Github receiving top priority for new features, while increasingly lagged behind in features and support. My experience with this deployment seems to support this concern. Despite this, I may still attempt to migrate back to Azure Devops to avoid paying the Github SSO tax, but my primary objective was to get this blog up as quickly as possible, and for the time being that meant using Github.

### Cloudflare vs Azure DNS
As previously stated, Azure is my primary IaaS platform, so it may seem natural that I would host my DNS in Azure as well. While I have considered migrating my DNS to Azure, I have been a Cloudflare user for several years and have been consistently pleased with the quality of their service (not to mention the price). Azure DNS still lags behind Cloudflare in terms of functionality for public DNS, with Azure DNS still not supporting seemingly basic features like DNSSEC. By staying on Cloudflare, I also get to benefit from Cloudflare's magic-like ability to propogate DNS change instantaneously worldwide, and potentially leverage features like Cloudflare Access down the road.

### Azure Portal vs Terraform
In both work and homelab, I am strong proponent of infrastructure automation and IaC. When I embarked on this project, I intended to build the resources using Terraform, but I soon discovered that Static Web Apps appear to be designed for an interactive deployment. Here is what I found:
1. The deployment of a Static Web App is almost trivially easy, reducing the value of using Terraform to automate the deployment. The Static Web App is a single resource in Azure, so the corresponding Terraform is extremely basic, as shown below.
```terraform
resource "azurerm_resource_group" "blog" {
    name = "rg-web-blog"

    location = "South Central US"
    tags     = local.tags
}

resource "azurerm_static_site" "blog" {
    name     = "ss-fitztech-io-blog"
    sku_tier = "Free"
    sku_size = "Free"

    resource_group_name = azurerm_resource_group.blog.name
    location            = "Central US"
    tags                = local.tags   
}
```
2. Automated integration with Github and Azure Devops (when it works) is only available when building the Static Web App resource via the Azure portal.
3. Adding custom domains (which involves the creation of certificates in addition to building DNS aliases), must be done in the Azure Portal after deployment, regardless of your initial deployment type.

If you are an automation enthusiast at heart, it can be difficult to put away the fancy tooling and do things the pointy-clicky way. In the case of Azure Static Web Apps, however, the GUI is the more efficient deployment tool.

# Walkthrough