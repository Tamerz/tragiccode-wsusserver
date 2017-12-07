# WSUSServer

#### Table of Contents

1. [Description](#description)
1. [Setup - The basics of getting started with wsusserver](#setup)
    * [Setup requirements](#setup-requirements)
    * [Beginning with wsusserver](#beginning-with-wsusserver)
1. [Usage - Configuration options and additional functionality](#usage)
    * [Host binaries at microsoft](#host-binaries-at-microsoft)
    * [Configuring Server-Side Targeting](#configuring-server-side-targeting)
    * [Configuring Client-Side Targeting](#configuring-client-side-targeting)
    * [Customizing the Synchronization Schedule](#customizing-the-synchronization-schedule)
    * [Creating computer target groups](#creating-computer-target-groups)
    * [Removing computer target groups](#removing-computer-target-groups)
    * [Creating automatic approval rules](#creating-automatic-approval-rules)
    * [Removing automatic approval rules](#removing-automatic-approval-rules)
1. [Reference - An under-the-hood peek at what the module is doing and how](#reference)
1. [Development - Guide for contributing to the module](#development)

## Description

The wsusserver module installs, configures, and manages Windows Server Update Services (WSUS) on Windows systems.

Windows Server Update Services (WSUS) allows for administrators to manage the deployment of updates for products (SQL Server, Window Server 2016..etc) released by Microsoft.

## Setup

### Setup Requirements

The wsusserver module requires the following:

* Puppet Agent 4.7.1 or later.
* Windows Server 2016.

### Beginning with wsusserver

To get started with the wsusserver module simply include the following in your manifest:

```puppet
class { 'wsusserver':
    package_ensure => 'present',
}
```

This example installs, configures, and manages the wsusserver service.  After running this you should be able to access the WSUS Console to begin you enterprise management of updates.

A more advanced configuration including most attributes available for the base/main class:

```puppet
class { 'wsusserver':
    package_ensure                     => 'present',
    include_management_console         => true,
    service_manage                     => true,
    service_ensure                     => 'running',
    service_enable                     => true
    wsus_directory                     => 'C:\\WSUS',
    join_improvement_program           => false,
    sync_from_microsoft_update         => true,
    update_languages                   => ['en'],
    products                           => [
      'Active Directory Rights Management Services Client 2.0',
      'ASP.NET Web Frameworks',
      'Microsoft SQL Server 2012',
      'SQL Server Feature Pack',
      'SQL Server 2012 Product Updates for Setup',
      'Windows Server 2016',
    ],
    update_classifications             => [
        'Windows Server 2012
        'Windows Server 2016',
    ],
    targeting_mode                     => 'Client',
    host_binaries_on_microsoft_update  => false,
    synchronize_automatically          => true,
    synchronize_time_of_day            => '03:00:00', # 3AM ( UTC ) 24H Clock
    number_of_synchronizations_per_day => 1,
}
```

The above is just an example of the flexibility you have with this module.  You will generally never need to specify every or even so many parameters as shown.

## Usage

### Host binaries at microsoft

Updates can be downloaded locally on the wsusserver and machines can pull approved and appropriate updates for installation straight from the wsusserver.  Since updates from microsoft are very large in size this requires a good amount of disk space to be available.  One option is to use the wsus server for approval and checking of updates and inform the clients they should pull updates directly from microsoft's updates servers relieving your server of the load and disk space.  This can be configured like so.

```puppet
class { 'wsusserver':
    package_ensure                     => 'present',
    host_binaries_on_microsoft_update  => true,
}
```

**NOTE: In order to use this option each machine needs someway to get out to the public internet to pull their updates directly from microsoft.**

### Configuring Server-Side Targeting

Updates can be targeted in 2 ways.  Client-side or Server-side.  Server-side targeting indicates some administrator must move computers in the appropriate computer groups in order for them to get their desired updates.  By default all computers that need to be manually classified will be found in the UnAssigned Computers group.  Server-side target is rarely used since it requires manual intervention.  While not recommended this can be done like so:

```puppet
class { 'wsusserver':
    package_ensure => 'present',
    targeting_mode => 'Server',
}
```

### Configuring Client-Side Targeting

Typically Client-Side targeting is used which is why it's ths default in this module if not specified.  Below is an example of setting client-side targeting explicitly.

```puppet
class { 'wsusserver':
    package_ensure => 'present',
    targeting_mode => 'Client',
}
```

This is typically done through 1 of 2 ways.

1. Group-Policy
1. Registry

It's recommended to avoid group policy all together and simply utilize the [puppetlabs/wsus_client](https://github.com/puppetlabs/puppetlabs-wsus_client) module ( which is a supported module by puppetlabs ) which handles configuring clients for client-side targeting.

### Customizing the Synchronization Schedule

The process of wsus server checking for new categories, products, and updates is called synchronization. While this can be done manually, it's typically done automatically on a configured scheduled.  Below is an example of how to configure automatic synchronization every day at around 3:00AM UTC ( which is the default configured by this module ).

```puppet
class { 'wsusserver':
    package_ensure                     => 'present',
    synchronize_automatically          => true,
    synchronize_time_of_day            => '03:00:00', # 3AM ( UTC ) 24H Clock
    number_of_synchronizations_per_day => 1,
}
```

If you want four synchronizations a day starting at 3:00AM UTC your schedule would look like below and be configured as such.

Number | Time To Sync
-------|-------------
1      | 3:00AM UTC
2      | 9:00AM UTC
3      | 3:00PM UTC
4      | 9:00PM UTC

```puppet
class { 'wsusserver':
    package_ensure                     => 'present',
    synchronize_automatically          => true,
    synchronize_time_of_day            => '03:00:00', # 3AM ( UTC ) 24H Clock
    number_of_synchronizations_per_day => 4 ,
}
```

**NOTE: The synchronization time of day has a random offset up to 30 minutes from the what you specify.  This is done automatically by microsoft in order to prevent a stampeeding herd on their servers.**

### Customizing Languages and Products

You typically don't want WSUS to manage updates in all languages or even for all products, there is just too many and this would require a huge server just to pull this load.  Therefore, you typically specify a subset of the full list of languages and products you would like wsus server to manage updates for.  Below is an example of this.

```puppet
class { 'wsusserver':
    package_ensure                     => 'present',
    update_languages                   => ['en'],
    products                           => [
      'Active Directory Rights Management Services Client 2.0',
      'ASP.NET Web Frameworks',
      'Microsoft SQL Server 2012',
      'SQL Server Feature Pack',
      'SQL Server 2012 Product Updates for Setup',
      'Windows Server 2016',
    ],
    update_classifications             => [
        'Windows Server 2012
        'Windows Server 2016',
    ]
}
```

**NOTE: The list of products, classifications, and languages for microsoft is constantly changing and currently i'm unable to find an updated list of where these can be found.  The best solution at the moment is to open wsusserver, Go to Options => Products and Classifications and picking a name of the product based on the tree view shown.**

### Creating computer target groups

Typically you group servers into groups so that you can role out changes in a controlled fashion or in a certain way.  Below shows how to create computer target groups in order to do this.

```puppet
wsusserver::computertargetgroup { ['Development', 'Staging', 'Production']:
    ensure => 'present',
}
```

### Removing computer target groups

Removing is just as simple as creating is.

```puppet
wsusserver::computertargetgroup { ['Development', 'Staging', 'Production']:
    ensure => 'absent',
}
```

### Removing automatic approval rules

When you initially install wsus ( and possibly in the future ) you might want to remove certain approval rules.  Below shows how to remove the built in approval rule.

```puppet
wsusserver::approvalrule { 'Default Automatic Approval Rule':
    ensure => 'absent'
}
```

### Creating automatic approval rules

Approving updates that you will always want/need can be a waste of time.  Automatic approval rules can be created in order to help remove this burden.  Below we automatically approval all Windows Server 2016 Security and Critical updates that apply to our servers in our production environment.

```puppet
wsusserver::approvalrule { 'Automatic Approval for Security and Critical Updates Rule':
    ensure          => 'present',
    enabled         => true,
    classifications => ['Security Updates', 'Critical Updates'],
    products        => ['Windows Server 2016'],
    computer_groups => ['Production'],
}
```

### Removing automatic approval rules

When you initially install wsus ( and possibly in the future ) you might want to remove certain approval rules.  Below shows how to remove the built in approval rule.

```puppet
wsusserver::approvalrule { 'Default Automatic Approval Rule':
    ensure => 'absent'
  }
```

## Reference

### Classes

Parameters are optional unless otherwise noted.

### `wsusserver`

Installs Windows Server Update Services (WSUS) and manages the configuration, service, and management tools.

#### `package_ensure`

Specifies whether the Windows Server Update Services (WSUS) role should be present. Valid options: 'present' and 'absent'.

Default: 'present'.

#### `include_management_console`

Specified if the management console should be installed.  This is the GUI used to manage wsus.

Default: true.

#### `service_manage`

Specifies whether or not to manage the desired state of the WSUS windows service.

Default: true.

#### `service_ensure`

Specifies whether the WSUS windows service should be running or stopped. Valid options: 'stopped' and 'running'.

Default: 'running'.

#### `service_enable`

Whether or not the WSUS windows service should be enabled at boot, disabled, or must be manually started. Valid options: true, false, and 'manual'

Default: true.

#### `wsus_directory`

Specifies the absolute path on the target system to store downloaded updates.

Default: 'C:\WSUS'.

#### `join_improvement_program`

Allows microsoft to collect statistics about your system configuration, events, and configuration of wsus to help microsoft improve the quality, reliability and performance of the product. Valid options: true, false

Default: true.

#### `sync_from_microsoft_update`

Specifices that this server should perform synchronizations against microsoft upate servers.  This assumes this wsus server is an upstream wsus server in your environment. Valid options: true, false

Default: true.

#### `upstream_wsus_server_name`

The name of the upstream wsus server in your environment in which to synchronize this wsus server against.

Default: undef.

#### `upstream_wsus_server_port`

The port of the upstream wsus server in your environment in which to synchronize this wsus server against.

Default: 80.

#### `upstream_wsus_server_use_ssl`

Specifies if the upstream wsus server in your environment in which to synchronize this wsus server against is using SSL.  Valid options: true, false

Default: false.

#### `update_languages`

*Required.*

The languages in which you want updates for.

**NOTE: This is required because this is specific to your organization's requirements.**

#### `products`

*Required.*

The products in which you want updates for.

**NOTE: This is required because this is specific to your organization's requirements.**

#### `update_classifications`

*Required.*

The classifications in which you want updates for.

**NOTE: This is required because this is specific to your organization's requirements.**

#### `targeting_mode`

Specifies wether or not server-side or client-side targeting is to be utilized. Valid options: 'Server' and 'Client'.

Default: 'Client'.

#### `host_binaries_on_microsoft_update`

Whether or not computers should download updates directly from Microsoft. Valid options: true, false

Default: true.

#### `synchronize_automatically`

Whether or not to perform synchronizations automatically. Valid options: true, false

Default: true.

#### `synchronize_time_of_day`

At what time synchronizations should happen in UTC time.

Default: 03:00:00. (Midnight UTC)

#### `number_of_synchronizations_per_day`

The number of times to perform synchronizations spreadout throughout the day starting at the [synchronize_time_of_day](#synchronize_time_of_day) parameter.

Default: 1.

#### `trigger_full_synchronization_post_install`

Specifies that an intial synchronization of wsus should happen immediately after installation. Valid options: true, false

Default: true.

**NOTE: Be aware that this step could take hours to finish!  This will not cause any issues with puppet but it might look like it's stalled.  It's not so just wait patiently!  If you are testing this module out for the first time you probably want to set this value to false.**

**NOTE: The time it takes to finish the initial synchronization depends on the languages, products, and update classifications that were selected.  It also depends on your internete connection as well.**

**NOTE: When you perform post-installation configuration tasks in the wsus wizard, this is the part at the end that has a check box asking if you want to begin initial synchronization.**

### Defined Types

Parameters are optional unless otherwise noted.

#### `wsusserver::approvalrule`

Installs, configures, and manages WSUS Approval Rules.

##### `rule_name`

Specifies a approval rule to manage. Valid options: a string containing the name of your approval rule.

Default: the title of your declared resource.

##### `ensure`

Specifies whether the approval rule should be present. Valid options: 'present' and 'absent'.

Default: 'present'.

##### `enabled`

Specifies whether the approval rule should be enabled. Valid options: true and false.

Default: true.

##### `computer_groups`

*Required.*

Specifies which computer groups this approval rule should apply to.

##### `products`

*Required.*

Specifies which products this approval rule should apply to.

##### `classifications`

*Required.*

Specifies which classifications this approval rule should apply to.

#### `wsusserver::computertargetgroup`

Installs, configures, and manages WSUS computer target groups.

##### `group_name`

Specifies a computer target group to manage. Valid options: a string containing the name of your computer target group.

Default: the title of your declared resource.

##### `ensure`

Specifies whether the computer target group should be present. Valid options: 'present' and 'absent'.

Default: 'present'.

## Development

## Contributing

1. Fork it ( <https://github.com/tragiccode/tragiccode-wsusserver/fork> )
1. Create your feature branch (`git checkout -b my-new-feature`)
1. Commit your changes (`git commit -am 'Add some feature'`)
1. Push to the branch (`git push origin my-new-feature`)
1. Create a new Pull Request