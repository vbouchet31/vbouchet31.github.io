---
layout: post
title:  "ACSF Tools command line quickstart"
date:   2022-12-15 17:00:00 +0100
categories: [drupal, acsf]
---
Acquia Cloud Site Factory (aka ACSF) is the Acquia solution to ease Drupal multisite.

#### Why you need ACSF Tools?
As a maintainer of sites hosted on an ACSF instance, you may have to execute a Drush command on all sites of
your factory. Out of the box, the only way to do it is to manually do something like:
```shell
drush -l site1.application.acsitefactory.com command
drush -l site2.application.acsitefactory.com command
drush -l site3.application.acsitefactory.com command
```

With a large factory, it quickly becomes unsustainable. It is where [acsf-tools](https://github.com/acquia/acsf-tools)
package can help. This package is not an official Acquia package and is maintained by some of technical architects
based on their current needs on projects or on their spare time.

In this article, we will focus on the few drush commands to execute operation on all or many sites of the factory.
This won't cover the commands which are more wrapper / helper around ACSF API.

#### How it works?
ACSF maintains a file `sites.json` in `/mnt/files/{$_ENV['AH_SITE_GROUP']}.{$_ENV['AH_SITE_ENVIRONMENT']}/files-private/`
which contains the list of the sites with their related informations like the database details, the domains, etc. This is
the file that acsf-tools will leverage to get the list of sites and basically generate the various `drush -l <site> <command>`.

#### ACSF Tools List command
The first command is `acsf-tools-list` which simply list down the sites of the factory with some basic information.

```shell
$ drush acsf-tools-list
site1
  name: nbjtr291
  flags:
    preferred_domain: 1
  conf:
    gardens_site_id: 291
    gardens_db_name: nbjtr291
    acsf_site_id: 291
    acsf_db_name: nbjtr291
    site_api_key: xxxxxxxxxxxxxxxxxxxxxxxxxx
  domains:
    0: site1.application.acsitefactory.com
    1: site1.application.com
  machine_name: site1
site2
  name: nbjtr431
  flags:
    preferred_domain: 1
  conf:
    gardens_site_id: 431
    gardens_db_name: nbjtr431
    acsf_site_id: 431
    acsf_db_name: nbjtr431
    site_api_key: xxxxxxxxxxxxxxxxxxxxxxxxxx
  domains:
    0: site2.application.acsitefactory.com
    1: site2.com
  machine_name: site2
```

It is possible to filter the fields returned:
```shell
$ drush acsf-tools-list --fields=name,domains
site1
  name: nbjtr291
  domains:
    0: site1.application.acsitefactory.com
    1: site1.application.com
    2. site1.backoffice-application.com
site2
  name: nbjtr431
  domains:
    0: site2.application.acsitefactory.com
    1: site2.com
    2: site2.backoffice-application.com
```

#### ACSF Tools ML command
The second command is `acsf-tools-ml` and is probably the most useful command. ML stands for "multiple -l". It will
execute the drush command given in argument on all / many sites of the factory.

This is a very basic example of clearing the Drupal cache on all the sites:
```shell
$ drush acsf-tools-ml cr
=> Running command on site1.application.com
 [success] Cache rebuild complete.
 
=> Running command on site2.com
 [success] Cache rebuild complete.
```

Check `drush acsf-tools-ml --help` to get more insight on formatting the command, specially when the command to be
executed requires arguments or options. The pattern is `drush acsf-tools-ml <command> "'<arg1>' '<arg2>' '<argN>'" "'opt1=val1' 'optN=valN'"`.

The command also comes with many options.

By default, `acsf-tools-ml` command will use the second domain in the list if any. The reason is that the first domain
is always the technical domain provided by ACSF (site.application.acsitefactory.com). The second domain is
the one configured as the live domain. If there is any other custom domain, they will be listed after. There may be some
situation where the drush command should be executed on one of the custom domain. Using the `--domain-pattern` option
can help to achieve this.

```shell
$ drush acsf-tools-ml cr --domain-pattern=backoffice
=> Running command on site1.backoffice-application.com
 [success] Cache rebuild complete.
 
=> Running command on site2.backoffice-application.com
 [success] Cache rebuild complete.
```

In some situation, it may be interesting to execute only on some sites of the factory instead of all. The option
`--sites-filter` can help to achieve this. It is possible to filter on the name, site id, db name and domain. The command
`drush topic docs:output-formats-filters` gives information about the expected format for this option.

```shell
$ drush acsf-tools-ml cr --sites-filter='name*=site1'
=> Running command on site1.application.com
 [success] Cache rebuild complete.
```

Executing some commands on each site of a factory can be impacting the performance of the overall factory. The best example
is probably the cache rebuild command (or any other command which purge some proxy cache) as sites will get extra load
to rebuild the caches. It may be interesting to introduce some delay between each command execution to spread the load.
The option `--delay` accepts a number of second to wait before executing on the next site.

```shell
$ drush acsf-tools-ml cr --delay=5

=> Running command on site1.application.com
 [success] Cache rebuild complete.


=> Sleeping for 5 seconds before running command on next site.

=> Running command on site2.com
 [success] Cache rebuild complete.
```

Another option is to execute the same command multiple times across the sites for specific duration. An example (which may not
be very realistic) is to process a queue during 1 minute on each site and loop during a specific duration. The
`--total-time-limit` accepts a number of second.

```shell
$ drush acsf-tools-ml queue:run "'queue_name'" "'time-limit=60'" --total-time-limit=140
=> Running command on site1.application.com
 [success] Processed 10 items from the queue_name queue in 60 sec.
 
=> Running command on site2.com
 [success] Processed 10 items from the queue_name queue in 60 sec.
 
=> Running command on site1.application.com
 [success] Processed 10 items from the queue_name queue in 60 sec.
```

As you can understand from this example, the command won't be killed when the total time limit is reached. Before
executing the command on the next site, it will check if the total time limit is reached and stop the process if needed.

The other options are default drush ones. The `--format` option to choose the format of the response. The `--fields` option
to choose which fields to return.

#### ACSF Tools MLC command
The command `acsf-tools-mlc` is very similar to `acsf-tools-ml` as the `c` stands for "concurrent". The only
difference is the `--concurrency-limit` option to restrict the number of sites to execute in parallel.

#### ACSF Tools Dump and Restore commands
Taking database dumps alone is not really useful if these can't be restored easily. The `acsf-tools-dump` command
is a wrapper around the drush `sql-dump` command and will automatically use the site name as the filename
of the dump. It allows to easily map the dump file with a site and to automate restoration via `acsf-tools-restore`.

By default, the `acsf-tools-dump` command stores the dump files into `~/drush-backups/YYYYmmdd-hhmm/`. It is possible
to specify a specific folder using the `--result-folder` option. Any other option given to the command will be
forwarded to the `sql-dump` command. For example, using the `--gzip` option will result in dumps being compressed.
```
$ drush acsf-tools-dump --result-folder=/tmp/my-backup --gzip
 Are you sure you want to take database dumps of all the sites of the factory in /tmp/my-backup. (yes/no) [yes]:
 > yes

 Target directory (/tmp/my-backup) does not exist. Do you want to create this directory? (yes/no) [yes]:
 > yes

=> Creating /tmp/my-backup directory ...

 [OK] Folder /tmp/my-backup has been created.


=> Taking database dump of site1 ...

 [OK] Database dump of dhuae site successfully saved in /tmp/my-backup/site1.sql.gz. 


=> Taking database dump of site2 ...

 [OK] Database dump of dhuae site successfully saved in /tmp/my-backup/site2.sql.gz.                                                                                                            
```

To restore the dumps taken, simply use the `acsf-tools-restore` command. Specify the source folder and if the
dumps are gzipped.

```
$ drush acsf-tools-restore --source-folder=/tmp/my-backup --gzip
You are about to run a command on all the sites of your factory.
        Do you confirm you want to do that? If so, type 'yes'

 Do you want to continue? (yes/no) [yes]:
 > yes


=> Restoring the Database on the Domain site1.application.com.

=> Dropping and restoring database on site1.application.com Completed.

=> Restoring the Database on the Domain site2.application.com.

=> Dropping and restoring database on site2.application.com Completed.
```

#### Using drush aliases
Because acsf-tools relies on a file managed by ACSF directly on the Acquia servers, it is possible to use the
drush aliases only if acsf-tools is installed on the target server.

For example, `drush @application.env acsf-tools-ml cr` will first ssh on `@application.env` and then execute
`drush acsf-tools-ml cr`.

If acsf-tools is not installed on the server, most of the command accept the `--alias=application.env` option. Using
this option will use the `drush @application.env rsync` command to download the `sites.json` file from the given alias
and use it to then generate and execute the `drush @application.env -l <site> <command>` commands.