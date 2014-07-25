---
layout: post
date: 2014-07-25
title: Snapshot management utility in Python for Azure Blob Storage
---

I recently worked with a partner who wanted to manage Blob snapshots from a Linux machine, mostly to create, delete and restore snapshots of Virtual Hard Drives. However it turns out we do not yet have any snapshotting commands in the [Azure cross-platform CLI](https://github.com/Azure/azure-sdk-tools-xplat). So I came up with a little Python sample that shows how to use the Storage API from the [Python Azure SDK](https://github.com/Azure/azure-sdk-for-python/) to implement a simple snapshot utility.

The utility has basically four commands:

- List snapshots: given a container and a blob name, list all existing snapshots.
- Take a new snapshot.
- Delete a snapshot: given a snapshot ID (basically a time stamp), delete the corresponding snapshot.
- Copy a snapshot: asynchronously copy a snapshot to a separate, new blob; this would be used to restore a given snapshot.

The basic idea behing the tool is that you can take snapshots whenever you want (once per day for example), and delete old snapshots when you don't need them anymore (only keep the 10 latest snapshots for example).

When you need to restore a snapshot, you copy it to a new, separate blob. You can then create a disk on top of this blob, and attach the disk to a VM in order to restore files, or you can just attach this restored disk to your VM in place of the original one.

{% gist tomconte/03216b95315caba9caba %}
