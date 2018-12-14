# Harbor 升级和数据库迁移向导 - release-1.2.0 分支

> ref: https://github.com/goharbor/harbor/blob/release-1.2.0/docs/migration_guide.md

本文仅摘取关键差异部分；


----------

> NOTE:
> 
> - **From v1.2 on**, you need to use the release version as the tag of the migrator image. 'latest' is no longer used for new release.
> - You must back up your data before any data migration.

## Upgrading Harbor and migrating data

> The migration tool is delivered as a docker image, so you should pull the image from docker hub. Replace `[tag]` with the release version of Harbor (e.g. 1.2) in the below command:

```
docker pull vmware/harbor-db-migrator:[tag]
```

> - Configure Harbor by modifying the file `harbor.cfg`, you may need to refer to the configuration files you've backed up during step 2. Refer to [Installation & Configuration Guide](https://github.com/goharbor/harbor/blob/release-1.2.0/docs/installation_guide.md) for more information. Since the content and format of `harbor.cfg` may have been changed in the new release, **DO NOT directly copy `harbor.cfg` from previous version of Harbor**.

> **IMPORTANT**: If you are upgrading a Harbor instance with LDAP/AD authentication, you must make sure `auth_mode` is set to `ldap_auth` in `harbor.cfg` before launching the new version of Harbor. Otherwise, users may not be able to log in after the upgrade.

> - To assist you in **migrating the `harbor.cfg` file from v0.5.0 to v1.1.x**, a script is provided and described as below. For other versions of Harbor, you need to manually migrate the file `harbor.cfg`.

```
cd harbor
./upgrade --source-loc source_harbor_cfg_loc --source-version 0.5.0 --target-loc target_harbor_cfg_loc --target-version 1.1.x
```

> **NOTE**: After running the script, make sure you go through `harbor.cfg` to verify all the settings are correct. You can make changes to `harbor.cfg` as needed.

## Migration tool reference

> Use `help` command to show instructions of the migration tool:

```
docker run --rm -e DB_USR=root -e DB_PWD=xxxx vmware/harbor-db-migrator:[tag] help
```
