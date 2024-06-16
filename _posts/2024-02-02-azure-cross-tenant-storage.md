---
layout: post
title: "Accessing Azure storage accounts in another tenant using private endpoints"
date: 2024-02-02T20:30:00+01:00
background: '/assets/images/azure-storage-private-endpoint.jpg'
toc: true
---

I recently was helping someone troubleshoot an issue some of their users were experiencing.

They have client laptops attempting to connect to an Azure storage account that exists in a customer tenant. The storage account is using IP allow-listing to prevent unauthorised IP addresses from being able to connect to the storage account.

The client laptops have an always-on VPN that routes their traffic to local node, a bit like a proxy. Normally, you would add the IP of the node to the allow list on the storage account and the client laptops would be able to connect.

However, the nodes are hosted in Azure and the customer was seeing connections from the private IP of the node instead of the public IP.

Why would that be?

### How Azure networking works

When you connect to resource hosted in Azure from a resource hosted in Azure, by default, the traffic never leaves Microsoft's global backbone network.

So the traffic hitting the storage account wasn't coming from the internet, it was coming from Microsoft's backbone, even though it was in another tenant - though it was in the same region.

### So how do we fix that?

Using private endpoints.

Microsoft has some [documentation](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview) that details what private endpoints are and how they can be used.

However, this documentation isn't very explicit on whether you need to peer networks or have some other means of connecting them directly. 

It's also not explicitly clear on whether it matters that it's in another tenant.

Below is what the docs have to say on the matter.

Private endpoints enable connectivity between the customers from the same:

- Virtual network
- Regionally peered virtual networks
- Globally peered virtual networks
- On-premises environments that use VPN or Express Route
- Services that are powered by Private Link

This might lead you to think that the virtual network your VM is in has to be peered with a virtual network in the customer tenant.

This perception isn't helped by the [troubleshooting document](https://learn.microsoft.com/en-us/azure/private-link/troubleshoot-private-endpoint-connectivity) which has similar bullet points.

There is an [official guide](https://learn.microsoft.com/en-us/azure/architecture/guide/networking/cross-tenant-secure-access-private-endpoints) on how to use private endpoints to access web apps from a VM in another tenant that doesn't mention peering so...

### Surely the same is true for storage right?

Yes.

But how do you know? Test it yourself!

I have two tenants of my own and I created a [GitHub repo](https://github.com/eannaoCeallaigh/azure-cross-tenant-storage) with some terraform code to deploy a VM and associated networking resources in one tenant and a storage account in another.

I deployed a private endpoint in the first tenant alongside the virtual machine and pointed it to the resource ID of the storage account.

If you are doing this manually, via the portal, and you have permission to access the customer tenant, you can choose the storage account from the dropdowns.

If you don't have permission to access the customer tenant, the resource ID of the storage account will still work.

![https://github.com/eannaoCeallaigh/www/master/assets/images/create-private-endpoint.png](https://raw.githubusercontent.com/eannaoCeallaigh/www/master/assets/images/create-private-endpoint.png)

Another thing to note is, if you don't have permission to the customer tenant, you will need someone who does to approve the connection.

![https://raw.githubusercontent.com/eannaoCeallaigh/www/master/assets/images/storage-account-private-endpoints.png](https://raw.githubusercontent.com/eannaoCeallaigh/www/master/assets/images/storage-account-private-endpoints.png)

When using terraform, there is an option to automatically approve private endpoint connections using the `is_manual_connection` attribute and setting it to `false`, but if the account you're running terraform with doesn't have permission to access the customer tenant, you can set this value to `true`.

The key part of all this, one that is easy to miss in some of the Microsoft documentation, is to ensure the virtual network is associated with the private DNS zone that gets created when you first create a private link. You can do everything else correctly but without the private DNS zone being linked to your virtual network, your VM won't be able to connect. 

![https://github.com/eannaoCeallaigh/www/master/assets/images/dns-vnet-links.png](https://raw.githubusercontent.com/eannaoCeallaigh/www/master/assets/images/dns-vnet-links.png)

For accessing blob storage, the private DNS zone will be named `privatelink.blob.core.windows.net`. It will be different if you're connecting to a file share or another azure resource.

![https://github.com/eannaoCeallaigh/www/master/assets/images/dns-records.png](https://raw.githubusercontent.com/eannaoCeallaigh/www/master/assets/images/dns-records.png)

### Great! What do I need to do?

If you're managing your storage account via the Azure portal, go to `Networking` and then click on `Private endpoint connections` and then click `Create`.

If you're using terraform, you can create a private endpoint in code using the [azurerm_private_endpoint](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/private_endpoint) resource.

### How can I confirm it's working?

From your VM, try to access a file in the storage account. If you can access a file, then everything is working.

For the purposes of testing this all out, I allowed anonymous access to the containers on the storage account I was testing with. **Do not allow anonymous access on a storage account unless you explicitly need to e.g. hosting files for anyone to download**

I placed a simple text file in the storage account and ran curl against the private DNS record.

`curl https://tenant022024sa.privatelink.blob.core.windows.net`

And my terminal returned the contents of the file.

![https://raw.githubusercontent.com/eannaoCeallaigh/www/master/assets/images/file-in-storage-account.png](https://raw.githubusercontent.com/eannaoCeallaigh/www/master/assets/images/file-in-storage-account.png)

![https://raw.githubusercontent.com/eannaoCeallaigh/www/master/assets/images/ssh-to-azure-vm.png](https://raw.githubusercontent.com/eannaoCeallaigh/www/master/assets/images/ssh-to-azure-vm.png)
