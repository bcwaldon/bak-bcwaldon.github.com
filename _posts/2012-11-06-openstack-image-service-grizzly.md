---
layout: post
title: OpenStack Image Service - Grizzly Planning

---

# So what is Glance?

As there has been some discussion about what Glance's place is in the OpenStack lineup, the first thing that should be made clear is what Glance intends to provide to its consumers. The primary objective of Glance is to publish a catalog of virtual machine images. Rather than own users' image data, Glance simply tracks where that data resides. Glance owns the metadata provided by users.

An important use case of Glance up until now has been the ability to stream image data through the Glance HTTP service. Glance will continue to support this to keep compatibility with the v1 and v2 API specifications, but the feature will not be supported in any future major versions of the API. Glance should not stand in the way of a user booting an image on a cloud - if a user prefers to stash an image in Swift and boot directly from that URI, it should be possible!

# Grizzly Planning

The Glance team has a lot to tackle during the next release cycle. The major pieces have been outlined below…

## Multiple Image Locations

In the interest of being more flexible with the storage locations of image data, Glance will allow users to provide several sources of image data for a single image record. An example use case would be for a cloud operator to dump a high-traffic image on an ephemeral volume for quick cloning at boot time, but keep a copy in a more persistent data store such as Swift in the case that the volume is lost.

- https://blueprints.launchpad.net/glance/+spec/multiple-image-locations
- https://etherpad.openstack.org/GrizzlyMultipleImageLocations

## Image Workers

Asynchronous image processing can be quite taxing to the API nodes, so Glance will provide a framework for offloading these processes to some type of worker. Glance already allows asynchronous image upload through the use of a 'copy_from' parameter, which will be the first process to be offloaded into this new framework. Two major features that have been discussed at previous design summits are image data conversion and inremental image coalescing - both of which can also be tackled once we develop this framework.

- https://etherpad.openstack.org/GrizzlyImageWorkers

## Incremental Images

In order to minimize data duplication and unnecessary network traffic, Glance will support the upload of 'incremental images.' An incremental image is a representation of the changed blocks of a disk with a link back to original image from which it was created.

- https://blueprints.launchpad.net/glance/+spec/hierarchical-images
- https://etherpad.openstack.org/GrizzlyImageHierarchy

## Image Property Protection

Nova cares about several 'magic' properties of images in order to properly boot them - some of the major ones being block_device_mapping, kernel_id, ramdisk_id, instance_uuid and image_type. The problem with this is that the owner of an image can modify those properties at will, possibly rendering it un-bootable. Glance will allow trusted privileged clients such as Nova or the cloud Admin to set specific properties on images that are write- or read-protected from the image owner.

- https://blueprints.launchpad.net/glance/+spec/api-v2-property-protection

## OpenStack Images API v2.1

The v2.0 OpenStack Images API was released with Folsom, yet it lacks a few major features:

**Image Sharing** 

It was decided that the image sharing capabilities (also known as 'image membership') of the v1 Images API needed to be revisited for the v2 API, but that design work was not completed by the time Folsom, and therefore the v2.0 API, was released. This will be tackled in the Grizzly timeframe.

**Create From Location & Copy-From**

Images can not yet be created from an explicit backend store location - this will be enabled based on the work for Multiple Image Locations. Additionally, the asynchronous image upload process will be added to the v2.1 API.


## Domain Object Model

Glance has suffered a bit of a mis-balance between new code and refactoring the existing code. We plan to completely overhaul the internals of Glance with a domain object model that should help re-align the codebase.

- https://blueprints.launchpad.net/glance/+spec/glance-domain-logic-layer

## glance-registry vs glance-api

The v1 and v2 Images APIs were implemented with seperate paths to the Glance database. The first of which proxies queries through a subsequent HTTP service (glance-registry) while the second talks directly to the database. As these two APIs should be talking to an equivalent system, we will be realigning their internal paths to talk through the service layer (created with the domain object model) directly to the database, effectively deprecating the glance-registry service.
