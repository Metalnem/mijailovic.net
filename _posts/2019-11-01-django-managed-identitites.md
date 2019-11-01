---
layout: post
title: "Using Azure managed identities with Azure blob storage backend for Django"
date: 2019-11-01 20:35:00 +0200
---

*Dedicated to Bakir and Milica*

My best friend and my wife are making a website with Django,
and I'm sometimes helping them in that adventure. One of the
things they had to implement was image upload. It's super
easy to do it in Django, because it's just a matter of adding
an [ImageField] to a model class. By default, these images
are persisted as files on a local filesystem. This is a horrible
default in my opinion, so I had to intervene and do two things:

- Switch to using Azure blob storage for storing images
- Do it in a secure way (without ever having to handle Azure access keys in code)

The first item is very easy to accomplish. The second one is slightly
more difficult to do, and it's the main topic of this post. I don't want
to go into details that are already explained elsewhere on the internet,
so I'll assume that you are at least somewhat familiar with Django and Azure
(especially [virtual machines], [blob storage], and [role-based access control]).

[ImageField]: https://docs.djangoproject.com/en/2.2/ref/models/fields/#django.db.models.ImageField
[virtual machines]: https://docs.microsoft.com/en-us/azure/virtual-machines/linux/overview
[blob storage]: https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blobs-introduction
[role-based access control]: https://docs.microsoft.com/en-us/azure/role-based-access-control/overview

## Using Azure blob storage backend

[Azure Storage] is one of the custom storage backends for Django
available in the [django-storages] Python package. You can install
it by running this command:

```shell
pip install django-storages[azure]
```

After that, you can add the following lines to your Django settings:

```python
DEFAULT_FILE_STORAGE = 'storages.backends.azure_storage.AzureStorage'
AZURE_ACCOUNT_NAME = 'your_azure_account_name'
AZURE_ACCOUNT_KEY = 'your_azure_account_key'
AZURE_CONTAINER = 'your_azure_container'
```

Voilà! All your images will from now on be saved as blobs in the Azure
storage container of your choice. The problem with this approach is
that now you have to keep your account key secure, which is not so
trivial to accomplish.

If you open your storage account page in the Azure portal, and go to
**Settings -> Access keys**, you will see the following recommendation:

>Store your access keys securely - for example, using
>Azure Key Vault - and don't share them. We recommend
>regenerating your access keys regularly. 

You can't go wrong with using [Azure Key Vault], so this is
a really good general approach for storing application secrets.
In this case, however, we can achieve our security goal in a
much simpler way.

[Azure Storage]: https://django-storages.readthedocs.io/en/latest/backends/azure.html
[django-storages]: https://django-storages.readthedocs.io/en/latest/
[Azure Key Vault]: https://docs.microsoft.com/en-in/azure/key-vault/key-vault-overview

## Using Azure managed identities

[Managed identities for Azure resources] is an awesome Azure
feature that allows you to authenticate to other Azure
services without storing credentials in your code. In a
nutshell, a managed identity is simply a special type of
[service principal]: you can assign some roles to it, and
then attach it to your compute resource (for example,
virtual machine, scale set, or App Service). After you enable
an identity on a service instance, you can request the access
token from the [Azure Instance Metadata service], and then use
that token to authenticate to one of the [services that support
managed identities].

[Managed identities for Azure resources]: https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview
[service principal]: https://docs.microsoft.com/en-us/azure/role-based-access-control/overview#security-principal
[Azure Instance Metadata service]: https://docs.microsoft.com/en-us/azure/virtual-machines/windows/instance-metadata-service
[services that support managed identities]: https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/services-support-managed-identities

I won't go into details about configuring your infrastructure
to use managed identities, since Azure documentation is already
doing a great job in that regard:

- [Use a Linux VM system-assigned managed identity to access Azure Storage]
- [Use a Windows VM system-assigned managed identity to access Azure Storage]

[Use a Linux VM system-assigned managed identity to access Azure Storage]: https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/tutorial-linux-vm-access-storage
[Use a Windows VM system-assigned managed identity to access Azure Storage]: https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/tutorial-vm-windows-access-storage

In essence, you have to do the following:

1. Enable managed identity on a VM
2. Create a storage account
3. Create a blob container in the storage account
4. Grant a managed identity access to the storage account (for example, using the [Storage Blob Data Contributor] role)
5. Get an access token from a VM
6. Use the access token for authentication

The tutorials I've linked will guide you through the first four of these
six steps using the Azure portal, but they will only show you how to work
with access tokens using Azure SDK for C#. In the next section I'll explain
how to work with access tokens using Python.

[Storage Blob Data Contributor]: https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#storage-blob-data-contributor

## Tying it all together

Azure documentation for managed identities does not state this anywhere,
but there is an official package from Microsoft called [msrestazure] that
you can use to acquire the access token:

[msrestazure]: https://pypi.org/project/msrestazure/

```python
from msrestazure.azure_active_directory import MSIAuthentication
token_credential = MSIAuthentication(resource='https://storage.azure.com')
```

You can now use this token credential to authenticate to any Azure service
that accepts Azure Active Directory authentication. For example, you can
access your storage account by creating an instance of **BlockBlobService**:

```python
from azure.storage.blob import BlockBlobService
service = BlockBlobService(account_name, token_credential=token_credential)
```

And how do you use this access token with django-storages? In
addition to already mentioned *AZURE_ACCOUNT_KEY*, django-storages
library also offers a parameter called *AZURE_TOKEN_CREDENTIAL*:

>A token credential used to authenticate HTTPS requests.
>The token value should be updated before its expiration.

When you first read this description, it might not be super obvious
how it is supposed to be used, but this parameter will in fact
accept the access token we have previously obtained! Now we can
simply add this to our Django settings:

```python
from msrestazure.azure_active_directory import MSIAuthentication

DEFAULT_FILE_STORAGE = 'storages.backends.azure_storage.AzureStorage'
AZURE_ACCOUNT_NAME = 'your_azure_account_name'
AZURE_CONTAINER = 'your_azure_container'
AZURE_TOKEN_CREDENTIAL = MSIAuthentication(resource='https://storage.azure.com')
```

And that's all! In just a few lines of code, we have implemented a simple,
elegant, and secure way to upload files from Django to Azure blob storage.
[May God guide you in your quest](https://www.youtube.com/watch?v=SWC08MHYp2M&t=88)!
