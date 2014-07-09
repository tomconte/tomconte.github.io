---
layout: post
date: 2013-09-16
title: How to interact with Windows Azure Blob Storage from Linux using Python
---

We have many Windows Azure SDKs that you can use on Linux to access Windows Azure Blob Storage and upload or download files, all hosted on [GitHub](https://github.com/windowsazure). For example, you could write scripts in [Python](https://github.com/WindowsAzure/azure-sdk-for-python) or [Node.JS](https://github.com/WindowsAzure/azure-sdk-for-node) to upload files to Blob storage.

If you just want to interact with Windows Azure Storage from the command line on a vanilla Linux installation though, chances are that Python is the scripting environment that is pre-installed, not requiring you to compile anything before you can start working.

Here is a quick cheat sheet on how to use the Python SDK from the command line:

## Install the Windows Azure SDK for Python

Just download the latest tarball from Git, extract the contents, and run the install script as root to install the SDK:

	curl -L -O https://github.com/WindowsAzure/azure-sdk-for-python/archive/master.tar.gz
	tar xzf master.tar.gz
	cd azure-sdk-for-python-master/src
	sudo python setup.py install

Now in order to make it simpler to use the Blob Service using the Python module, you can set your Storage credentials as environment variables. You can find these values in the Windows Azure Management Portal, just select your Storage Account and click on the "Manage Access Keys" button to open the pop-up dialog that shows both the account name and the secret keys.

	export AZURE_STORAGE_ACCOUNT=tcontepub
	export AZURE_STORAGE_ACCESS_KEY='secret key'

Now it's super simple to access Blob Storage directly from the Python interpreter; just run `python`, import the module and create the `BlobService` object:

	from azure.storage import BlobService
	blob_service = BlobService()

Now here are a few code snippets you can use...

## List containers in the account

	for i in blob_service.list_containers():
	   print i.name

## Create a container

	blob_service.create_container('foo')

## Upload a file into a container

The first parameter is the container name, then the name of the Blob that will be created, then the data to be sent (typically the contents of a file you read from the local filesystem).

	blob_service.put_blob('foo', 'README.txt', file('/usr/share/doc/grep/README').read(), 'BlockBlob')

Hope this helps. You can find more information and examples on the [Windows Azure SDK for Python Github page](https://github.com/WindowsAzure/azure-sdk-for-python).

## Update: uploading large files

There is a limitation with this simple method: you cannot upload file larger than 64 MB without using a block-based upload approach, that will let you upload files up to 200 GB in size. We have some [sample Python code in the Windows Azure Python Developer Center](http://www.windowsazure.com/en-us/develop/python/how-to-guides/blob-service/#large-blobs) that shows how to upload large files using blocks.
