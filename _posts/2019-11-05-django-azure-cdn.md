---
layout: post
title: "Using Azure CDN with Azure blob storage backend for Django"
date: 2019-11-05 21:25:00 +0200
---
Just a few days ago, I've published a post on
[using Azure managed identities with Azure blob storage backend for Django]({{ site.baseurl }}{% post_url 2019-11-01-django-managed-identitites %}). In today's post, I will
continue the story by showing you how to further improve the setup
by adding [Azure CDN] to the mix. It turns out that it's extremely
easy to do that, which is why this post will probably be the shortest
one you will ever read on this blog!

You can create an Azure CDN endpoint directly from your storage
account page by following the instructions from this quickstart:
[Integrate an Azure storage account with Azure CDN]. If you
are super lazy, you don't even have to read it, because all
you have to do is to navigate to **Blob service -> Azure CDN**
in the Azure portal, and then choose the profile name, pricing
tier, and endpoint name:

![](/assets/img/2019-11-05-azure-portal.png)

The only remaining step is to configure django-storages to use the CDN
endpoint you have just created, which you can do easily by setting
the *AZURE_CUSTOM_DOMAIN* parameter to the endpoint's hostname.
For example, if you have a storage account named **djangoazure**,
container named **static**, and the hostname of your CDN endpoint
is **djangoazure**, your final django-storages configurations
should look like this:

```python
from msrestazure.azure_active_directory import MSIAuthentication

DEFAULT_FILE_STORAGE = 'storages.backends.azure_storage.AzureStorage'
AZURE_ACCOUNT_NAME = 'djangoazure'
AZURE_CONTAINER = 'static'
AZURE_CUSTOM_DOMAIN = 'djangoazure.azureedge.net'
AZURE_TOKEN_CREDENTIAL = MSIAuthentication(resource='https://storage.azure.com')
```

That's all for today! Enjoy your monthly Azure bill!

[Azure CDN]: https://docs.microsoft.com/en-us/azure/cdn/cdn-overview
[Integrate an Azure storage account with Azure CDN]: https://docs.microsoft.com/en-us/azure/cdn/cdn-create-a-storage-account-with-cdn
