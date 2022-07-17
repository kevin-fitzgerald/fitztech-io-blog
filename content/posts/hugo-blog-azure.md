---
title: "Deploying a Hugo Blog to Azure"
date: 2022-07-17T14:25:38-05:00
tags: [azure, hugo, devops]
categories: [Guides]
draft: true
---
## Terraform
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
