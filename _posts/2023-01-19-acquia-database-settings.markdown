---
layout: post
title:  "Alter database settings on Acquia platform"
date:   2023-01-19 17:00:00 +0100
categories: [ace, acsf]
---

On Acquia platform (Acquia Cloud or Acquia Cloud Site Factory), there are situations where you may need to alter
some database settings prior the connection to the database is established. Some examples are the
change of the transaction isolation level from `REPEATABLE-READ` (the default level) to `READ-COMMITTED` to
avoid/minimize the deadlocks (see this [KB article](https://support-acquia.force.com/s/article/360005253954-Fixing-database-deadlocks))
 or to override the default `wait_timeout` value (see this [KB article](https://support-acquia.force.com/s/article/360049944713-How-to-override-the-default-MySQL-wait-timeout-setting)).

**Please note that the code snippets shared here may be dependent of the context, the versions, etc. The goal is not really
to share ready-to-use code snippets but to explain how the database connection is initialized and how to alter the settings.**

## ACE
On Acquia Cloud, the database settings are loaded by [including a `/var/www/site-php/XXXXXX/XXXXX-settings.inc` file](https://docs.acquia.com/cloud-platform/manage/code/require-line/) which
is managed by Acquia on the server.

This file determines the Drupal's version, the environment and the site name to load another file:
```php
$site_info = ah_site_info_keyed();
$ah_site_php_dir = "/var/www/site-php/{$site_info['sitename']}";
$ah_drupal_version = ah_drupal_core_version();
require "{$ah_site_php_dir}/D{$ah_drupal_version}-{$site_info['environment']}-XXXXX-settings.inc";
```

This file contains the `$databases` information and use a function `acquia_hosting_db_choose_active()` to initiate the
connection with the database.

Because of this default behavior which automatically connect to the database, we need to prevent that from happening,
alter the `$databases` information and then connect to the database.

The implementation is slightly different if BLT is used or not on the project but the principle described remains the same. 

### Without BLT
On a project without BLT, the `sites/default/settings.php` is managed by the development team directly, so it is very easy to
alter.
Prior including the required Acquia file, disable the database automatic connection. After including the Acquia file,
alter the `$databases` array and connect to the database.

```php
if (file_exists('/var/www/site-php')) {
  global $conf, $databases;
  $conf['acquia_hosting_settings_autoconnect'] = FALSE;

  require('/var/www/site-php/' . $_ENV['AH_SITE_GROUP'] . '/' . $_ENV['AH_SITE_GROUP'] . '-settings.inc');
  
  $databases['default']['default']['init_commands'] = array(
    'isolation' => "SET SESSION tx_isolation='READ-COMMITTED'",
  );
  acquia_hosting_db_choose_active();
}
```

### With BLT
On a project with BLT, it is [requested to not change](https://docs.acquia.com/blt/install/next-steps/#adding-settings-to-settings-php)
directly the `sites/default/settings.php`. The file includes a specific blt.settings.php file from the vendor directory:
```php
require DRUPAL_ROOT . "/../vendor/acquia/blt/settings/blt.settings.php";
```
This file then takes care of loading various settings files in a specific order, including the Acquia specific file.
As per the comment in `blt.settings.php`:
```php
/**
 * Include additional settings files.
 *
 * Settings are included in a very particular order to ensure that they always
 * go from the most general (default global settings) to the most specific
 * (local custom site-specific settings). Each step in the cascade also includes
 * a global (all sites) and site-specific component. The entire order is:
 *
 * 1. Custom early settings (provided by the project)
 * 2. Acquia Cloud settings (including secret settings)
 * 3. Default general settings (provided by BLT)
 * 4. Custom general settings (provided by the project)
 * 5. Default CI settings (provided by BLT)
 * 6. Custom CI settings (provided by the project)
 * 7. Local settings (provided by the project)
 */
```

To change the database connection settings, we first need to disable the autoconnect in a early settings file.
Create a file `sites/settings/early.settings.php` which contains the following code:
````php
if (file_exists('/var/www/site-php')) {
  global $conf;
  $conf['acquia_hosting_settings_autoconnect'] = FALSE;
}
````

Then a file `sites/settings/databases-settings.php` with the following code:
```php
if (file_exists('/var/www/site-php')) {
  global $databases;
  $databases['default']['default']['init_commands'] = [
    'isolation' => "SET SESSION tx_isolation='READ-COMMITTED'",
  ];
  acquia_hosting_db_choose_active();
}
```

Be sure that you have a file `sites/settings/global.settings.php` which takes care of including the `databases-settings.php` file.
You may want to copy the previous code directly in the `global.settings.php` which would also work. But the file would quickly
become messy if it contains many settings change.
```php
$additionalSettingsFiles = [
  DRUPAL_ROOT . '/sites/settings/databases.settings.php',
];

foreach ($additionalSettingsFiles as $settingsFile) {
  if (file_exists($settingsFile)) {
    require $settingsFile;
  }
}
```

## ACSF
A project with ACSF is slightly different than an ACE one as the development team never manage the `sites/g/settings.php` file directly. Instead,
it is requested to use the `acsf` module which provides a `drush acsf-init` command to alter the settings file. This command is
invoked by ACSF during code deployment. If the settings file is overridden after the command is executed, the deployment will be
stopped by ACSF. The development team will have to execute the drush command locally and commit the settings file.
Because it is not possible to change the settings file, ACSF provides some pre-settings-php and post-settings-php factory hooks.
A summary of the settings file generated is the following:
```php
if (function_exists('acsf_hooks_includes')) {
  foreach (acsf_hooks_includes('pre-settings-php') as $_acsf_include_file) {
    include $_acsf_include_file;
  }
}

...

// Load database connection details for the current site.
$_acsf_include_file = "/var/www/site-php/{$site_settings['site']}.{$site_settings['env']}/D{$drupal_version}-{$site_settings['env']}-{$site_settings['conf']['acsf_db_name']}-settings.inc";
include $_acsf_include_file;

...

if (function_exists('acsf_hooks_includes')) {
  foreach (acsf_hooks_includes('post-settings-php') as $_acsf_include_file) {
    include $_acsf_include_file;
  }
}
```

Create a file `factory-hooks/pre-settings-php/databases.php` with the following code:
```php
global $conf;
$conf['acquia_hosting_settings_autoconnect'] = FALSE;
```
The name can be different, all the files in pre-settings-php are loaded in alphabetical order. If a file loaded before
already does `global $conf;` then this line is optional.

Then a file `factory-hooks/post-settings-php/databases.php` with the following code:
```php
global $conf, $databases;
$databases['default']['default']['init_commands']['isolation'] = "SET SESSION tx_isolation='READ-COMMITTED'";
acquia_hosting_db_choose_active($conf['acquia_hosting_site_info']['db'], 'default', $databases, $conf);
```