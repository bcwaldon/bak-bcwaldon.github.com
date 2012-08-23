---
layout: post
title: OpenStack Folsom &amp; Glance
tags:
  - openstack
  - folsom
  - glance
  - images-api

---
As we are coming up on the Folsom OpenStack release, I thought it would be a good idea to cover what landed in Glance over the last several months. It is becoming increasingly difficult to keep up with all of our projects, so I hope this overview helps!

## Python Client Rewrite

A new client library, [python-glanceclient](http://github.com/openstack/python-glanceclient), was created during the Folsom release cycle to replace the client that currently lives in the Glance repository. This new client has already been integrated into Nova, Cinder, Horizon and Devstack.

As a [library project](http://wiki.openstack.org/ProjectTypes), python-glanceclient does not follow the same release cycle as core projects such as Glance. It uses a three-part versioning mechanism and is distributed through OpenStack-supporting distros, [pypi](http://pypi.python.org/pypi/python-glanceclient), and [GitHub](http://github.com/openstack/python-glanceclient).

The importance bit is that the (now considered 'legacy') client code in Glance is officially deprecated in favor of this new client. It will still ship with Folsom, but it prints warnings to stderr when used from the command-line and raises `UserWarning` when imported directly in python. Expect the legacy client code to be removed from the Glance repository sometime early in the Grizzly release cycle.

The new client does maintain 100% CLI compatibility for interactions with the v1 Images API, but that legacy usage will never be updated to work with the v2 API. The python interface does not maintain compatibility with the legacy client, however, the python module does not conflict with that installed by the Glance source. Expect to see more about python-glanceclient in a subsequent post.

## OpenStack Images API v2.0

Version 2.0 of the OpenStack Images API is largely defined based on the work that was finished in the Folsom release of Glance. Expect a separate post covering the v2.0 API spec itself.

I wanted to specifically thank Mark Washenberger, Alex Meade, Iccha Sethi and Nikhil Komawar of Rackspace for their outstanding work on the v2.0 API implementation!

See the [super-blueprint on launchpad](https://blueprints.launchpad.net/glance/+spec/api-2).

## High-Profile Bugs

Features are always awesome, but the biggest request of the OpenStack community has been to work on stabilization and bug-fixing. The following is a list of major bugs that were fixed in the Folsom release cycle, many more lower-priority bugs were fixed and can be seen on launchpad. Bugs that affect stable/essex have been backported accordingly.

### Keystone Integration

* [1010519](https://bugs.launchpad.net/glance/+bug/1010519): Use a case-insensitive check when determining if a user has admin privileges

### Swift Integration

* [979745](https://bugs.launchpad.net/glance/+bug/979745): Actually delete images from swift when using Keystone authentication.
* [1002791](https://bugs.launchpad.net/glance/+bug/1002791): Replace usage of `swift.client` with `swiftclient` - swift dependency replaced with python-swiftclient.
* [994296](https://bugs.launchpad.net/glance/+bug/994296): Allow swift account name to contain the '@' character.

### SSL

* [1032451](https://bugs.launchpad.net/glance/+bug/1032451): Allow server to validate client certificate using CA cert specified in `ca_file` config option.
* [1007093](https://bugs.launchpad.net/glance/+bug/1007093): Images less than 4 KB in size are now uploaded only once; data is no longer duplicated.

### Database

* [1012823](https://bugs.launchpad.net/glance/+bug/1012823): Prevent database auto-creation - configurable using `db_auto_create` config option.
* [975651](https://bugs.launchpad.net/ubuntu/+source/glance/+bug/975651): Update all image id references in db migration 12; image properties were being ignored.

### Miscellaneous

* [1038994](https://bugs.launchpad.net/glance/+bug/1038994): Maximum image size is now configurable using `image_size_cap` (default 1 TB) and chunked transfer-encoding HTTP requests are now properly validated.
* [978130](https://bugs.launchpad.net/glance/+bug/978130): Multiprocess-enabled servers now respect CTRL+C.
* [1031842](https://bugs.launchpad.net/glance/+bug/1031842): Partially-cached images are removed when a client connection is prematurely closed.
* [1021054](https://bugs.launchpad.net/glance/+bug/1021054), [1021740](https://bugs.launchpad.net/glance/+bug/1021740): Admins can now share images regardless of ownership or existing sharing permissions.

## Show me the Features!

### Tenant-Specific Storage Locations in Swift

The ability to store image data in the image owner-specific locations has always been an interesting feature on our backlog as storing all users' images in a single account is a no-go for some deployers. We ended up with a solution that is only implemented for Swift, but done in such a way that we can expand it easily during Grizzly.

The implementation adds a couple of configuration options: 

* `swift_store_multi_tenant`: this must be set to 'True' to enable this feature (it defaults to 'False'). 
* `swift_store_admin_tenants`: this is a list of tenants, referenced by id, that should be granted read and write access to all swift containers created by Glance.

Assuming you configured 'swift' as your `default_store` and you enable this feature as described above, images will be stored in a swift endpoint pulled from the authenticated user's service_catalog. The created image data will only be accessible through Glance by the tenant that owns it and any tenants defined in `swift_store_admin_tenants` that are identified as having admin-level accounts.

Special thanks to Dan Prince of RedHat for tackling this feature!

See the [blueprint on launchpad](https://blueprints.launchpad.net/glance/+spec/swift-tenant-specific-storage).

### Image Replication Tool

At the Folsom OpenStack Design Summit, we decided that the best first-run at image replication would be to write a simple python script that could talk to multiple Glance endpoints and sync images between them. During this release cycle, we developed the `glance-replicator` tool.

It is currently only available through [GitHub](http://github.com/openstack/glance/tree/master/bin/glance-replicator), but we hope to develop a better distribution method in Grizzly.

Once you have it installed, you should run the help command to determine specific usage: `glance-replicator help`. The tool already has a great list of features:

* Replicate binary image data along with metadata
  * Local replication will download image data to a local directory
  * Live replication can circumvent local disk access
* Compare Glance endpoints to see what images need to be replicated
* Determine the effective size of a Glance endpoint
* Prevent replication of specific metadata attributes
* Provide separate authentication tokens for master and slave endpoints

Thanks a ton to Michael Still of Canonical for his fantastic work on this tool!

See the [blueprint on launchpad](https://blueprints.launchpad.net/glance/+spec/image-replication).

### Database Driver Pluggability

The internal pythonic interface used to interact with sqlalchemy models, a.k.a. the database driver, has limited deployers and developers to using only the backends that sqlalchemy itself supported (MySQL, PostgreSQL, SQLite, etc.).

The database driver has been made pluggable using the `data_api` configuration option. A deployer of Glance can use the default sqlalchemy API, or one could specify any python module path that implements the same interface.

The in-memory database driver, referred to as the 'simple' driver, has been promoted from a test helper to the first sqlalchemy alternative. It should only be used for testing and in extremely rare deployment configurations. It doesn't write anything to the local filesystem, so expect your data to be lost if you restart your services. It cannot share between services, so you are limited to a single glance-registry and glance-api process (the `workers` config must be 0). Additionally, data will not be shared between the v1 and v2 APIs.

If you are interested in writing a driver for your favorite backend, you can use the test suite defined in [glance/tests/functional/db/\_\_init\_\_.py](http://github.com/openstack/glance/tree/master/glance/tests/functional/db/__init__.py). See the test modules [test_simple](https://github.com/openstack/glance/blob/master/glance/tests/functional/db/test_simple.py) and [test_sqlalchemy](https://github.com/openstack/glance/blob/master/glance/tests/functional/db/test_sqlalchemy.py) in that same directory for an example of how to integrate with the tests. Unfortunately, the testing suite is incomplete and cannot yet guarantee complete interface compliance. The sqlalchemy api is the de facto standard as of now, but that responsibility will transition to the test suite in the Grizzly timeframe.

See the [blueprint on launchpad](https://blueprints.launchpad.net/glance/+spec/refactor-db-layer).

### Store Driver Pluggability

Similar to how the database driver was made pluggable, the list of available image store drivers has been pushed into a configuration option `known_stores`:

    known_stores = 
        glance.store.filesystem.Store,
        glance.store.http.Store,
        glance.store.rbd.Store,
        glance.store.s3.Store,
        glance.store.swift.Store,

If you want to write an additional store driver, you can implement the interface defined by the base class in `glance.store.base.Store` and append the pythonic path of your store class to the `known_store` option. Feel free to use that base class as your super class and define the methods that raise `NotImplementedError`, or write your own from scratch.

Testing is not as straightforward at the moment, so please refer to the [files here](https://github.com/openstack/glance/tree/master/glance/tests/functional/v1) for reference.

Thanks to Josh Harlow of Yahoo for making this happen!

See the [blueprint on launchpad](https://blueprints.launchpad.net/glance/+spec/import-dynamic-stores).


## What about Grizzly?

While plans are far from being solidified for the next release cycle, I can throw a few teasers out there:

* **Protected Properties**: Deployers will be able to set read/write permissions on individual image properties.
* **Privileged Services**: Glance endpoints should be able to identify trusted clients (like Nova, Cinder or Horizon) and respond with additional information.
* **Domain Objects**: The internal APIs of Glance have never been well-defined. We hope to create a proper domain object layer to help minimize code repetition in the API controllers and database drivers.
* **Images API v2.1+**: Image sharing will make its way into the v2 API, along with several other yet-to-be finalized features.
