---
layout: post
title: Upgrading OpenStack Glance - Essex to Folsom

---

As Glance consists of only two services, the upgrade process is pretty straightforward. This post walks through what I did to upgrade an Essex Glance deployment to Folsom.

Full disclosure - everything on my system has been installed from source.

## Backup

This isn't a required step, but it's always good to keep a backup of config files. I moved all of my config and paste files from `/etc/glance/` to `/etc/glance.essex/`:

    etc/
        glance.essex/
            glance-api.conf
            glance-api-paste.ini
            glance-registry.conf
            glance-registry-paste.ini
            
Don't worry about backing up your actual image data - Folsom will be able to pick up right where Essex left off.

## Configuration

The biggest pain of the upgrade is dealing with the paste configuration files. Thankfully, this should be the last release that you have to go through this process!

Copy the essex config files (not paste files) into a newly-created  `/etc/glance` directory:

    etc/
        glance/
            glance-api.conf
            glance-registry.conf
        glance.essex/
            glance-api.conf
            glance-api-paste.ini
            glance-registry.conf
            glance-registry-paste.ini

Copy the paste files provided by the glance source into the new config directory - not your old essex paste files!

    etc/
        glance/
            glance-api.conf
            glance-api-paste.ini
            glance-registry.conf
            glance-registry-paste.ini
        glance.essex/
            glance-api.conf
            glance-api-paste.ini
            glance-registry.conf
            glance-registry-paste.ini

If you have any custom paste pipelines, now would be the time to copy them into your new paste files. I'm not going to cover this process since I assume you know what you're doing if you wrote your own pipelines.

### Auth Paste Configuration

*NOTE: Skip this step if you aren't using Keystone.*

Move all of the `auth_*` and `admin_*` attributes in the `[filter:authtoken]` sections out of your paste files into new `[keystone_authtoken]` sections of the corresponding configuration files. 

For example, I took the `auth\_*` and `admin_*` values out of `/etc/glance.essex/glance-api-paste.ini`, which looked something like this: 

    [filter:authtoken]
    paste.filter_factory = keystone.middleware.auth_token:filter_factory
    service_protocol = http
    service_host = 127.0.0.1
    service_port = 5000
    auth_host = 10.0.2.15
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = service
    admin_user = glance
    admin_password = secrete
    
And put them at the bottom of `/etc/glance/glance-api.conf` in a new `[keystone_authtoken]` section, which then looked like this: 

    [keystone_authtoken]
    auth_host = 10.0.2.15
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = service
    admin_user = glance
    admin_password = secrete

I did the same thing for `/etc/glance.essex/glance-registry-paste.ini` and `/etc/glance/glance-registry.conf`.

### v2 API Configuration

With the Folsom release comes a new HTTP API, the OpenStack Images API v2.0. You must decide to either leave it on and configure it, or turn it off completely.

To enable the v2 API, simply copy your sql-related glance-registry configuration into your glance-api configuration file. The relevant bits of my `/etc/glance/glance-api.conf` looked like this:

    [DEFAULT]
    enable_v2_api = true
    sql_connection = mysql://root:secrete@localhost/glance?charset=utf8
    sql_idle_timeout = 1800
    sql_max_retries = 10
    sql_retry_interval = 1
    db_auto_create = false

To turn it off, simply set the `enable_v2_api` flag to 'false'.

    [DEFAULT]
    enable_v2_api = false

## Update Keystone

*NOTE: Skip this step if you aren't using Keystone.*

Since Glance depends on importing a piece of middleware owned by Keystone, you must update your local Keystone code to Folsom.

Since this walkthrough is aimed at Glance, I won't dive into what it takes to upgrade a full Keystone deployment. Here's what I did to make the latest Keystone code available to Glance: 

    $ pip install keystone --upgrade
    
## Update Other Dependencies

Use the `pip-requires` file that ships with the Glance source code to determine what you should install or update. I ended up asking pip to install everything for me so I didn't miss any updated version requirements:

    $ pip install -r tools/pip-requires --upgrade
    
I do want to highlight that there are two completely new dependencies that may or may not be packaged on your platform, `jsonschema` and `python-swiftclient`.

## Stop Services

Now stop glance-api and glance-registry.

    $ glance-control api stop
    Stopping glance-api  pid: 29451  signal: 15
    
    $ glance-control registry stop
    Stopping glance-registry  pid: 29447  signal: 15

## Upgrade Database

Run `glance-manage db_sync` to upgrade your database - it should end up at version 15:

    $ glance-manage db_sync
    2012-09-06 00:14:20     INFO [migrate.versioning.api] 13 -> 14... 
    2012-09-06 00:14:20     INFO [glance.db.sqlalchemy.migrate_repo.schema] creating
    table image_tags
    2012-09-06 00:14:20     INFO [migrate.versioning.api] done
    2012-09-06 00:14:20     INFO [migrate.versioning.api] 14 -> 15... 
    2012-09-06 00:14:20     INFO [migrate.versioning.api] done
        
    $ glance-manage db_version
    15

## Start Services

Now start glance-api and glance-registry

    $ glance-control registry start
    Starting glance-registry
    
    $ glance-control api start
    Starting glance-api
