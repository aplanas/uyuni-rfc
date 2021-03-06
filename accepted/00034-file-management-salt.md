- Feature Name: File management with salt
- Start Date: 2017-08-22
- RFC PR: https://github.com/SUSE/susemanager-rfc/pull/56


# Summary
[summary]: #summary

Allow managing (config) files for salt-systems with `file.managed` and improve
custom state writing.

# Motivation
[motivation]: #motivation

Users complain about missing the possibility to create/upload files
which can be used via the salt filesystem for example in custom states.
Currently we only support creating SLS files using the state catalog.
Importing just files or templates is not possible.

Implementing this also reduces the feature parity gap between salt and traditional systems.

Additionally the users requested revisions of files and the possibility to rollback to an
earlier version.

# Detailed design
[design]: #detailed-design

We reuse the existing mechanism for configuration channels for the new for salt systems.
More precisely, we use only the `Global Configuration Channel`.
The other two (local and sandbox) can only be created via the UI in the system details.
This UI should be not available for Salt Systems.

**What is a configuration channel in salt?**

It could be a subdirectory in the filesystem. To prevent clashes between orgs, we need
to generate the `org_id` into the directory name when we write it on the Filesystem.
Additionally we use it as prefix for IDs. The contents of the subdirectory is
(re)generated when the configuration channel or a file is changed in SUSE Manager.

**A configuration channel contains only files (directories and symlinks). How to
get them deployed?**

All required information is available in the database tables. So we autogenerate the `init.sls`.
It will not exist in the database and has no revision. The content just depends
on the current state of the channel and include the latest config files (symlinks and directories).

**How to assign Config Channels?**

For traditional systems the table `rhnServerConfigChannel` is used (no versioning as in
our `suseStateRevision` mechanism used for custom/package states). This table has also a
position attribute to order the Channels. We will use this table for assignment
to salt systems as well.

**Functions not allowed for salt systems**

* Deploy of individual file from a configuration channel - don't show for salt
  system in the UI, don't allow in the XMLRPC

### Use configuration channels for custom states

A configuration file is a generalization of the custom state as currently
implemented in SUMA. We plan to unify the implementation of custom states and
configuration files so that custom state can be versioned, too.

We want to use the config file mechanism also for CRUD for Custom States - the
admin would write the SLS file (web UI, xmlrpc).
So we generate one config channel per custom state and create a `init.sls`.
If the `init.sls` exists as file in the config channel it will not be autogenerated.
Then it is up to the admin to write it. Also imported files in such a channel might
or might not be referenced by this or any other sls file.
Assignment to a system makes use of existing revisioning we already have (see
table `suseStateRevision` and deps) as we need to assign custom states to groups
and orgs as well.

This requires a new type for the Config Channel. This type can be used to show it in a seperate UI.
We also need a new file type `SLS`. The special thing about a file of type `SLS` is, that it
does not get deployed. We do not need a destination and user, group and mode (`rhnConfigInfo`) is
obsolete as well.

#### Enhance Custom States to write Formulas

The difference between a custom state and a formula is, that a formula has two
additional files
- form.yml
- metadata.yml

We can enhance the Custom State by allowing to convert a custom state into a
formula by importing these files. Of course we would need another file type.


## Pros/Cons of reusing traditional config channel mechanism

### Pros
We can reuse UIs and implementations, but we risk conceptual clashes between
traditional and salt management.

We have to make changes to the existing UI, but they may be small
(introduce and handle new SLS type). Another advantage: When we decide to
re-write the UI in ReactJS, the traditional stack can also use the new UI seamlessly.

Migration from traditional to salt is easier and customers stay with UIs they
already know.

Customers don't need to know that salt is behind it - it's a "implementation
detail".

### Cons
The assignments for systems to channels is different as for traditional systems
from the UI point of view. This can result in confusing UIs.

Using config channels for implementing custom states brings these problems:

* UI duplication / accessing the same UI from 2 different points. We still have
the "Configuration" item in the sidebar but also the "State Catalog" below the
"Salt" item.
* What to do with "State Catalog"? It stays, but uses the UI from
"Configuration". Possibility to move "State Catalog" as a subitem of
"Configuration".

# Advanced features / Enhancements

While thinking about this feature we got some more ideas about improvements which
could be done when implementing it.

## Migration of configuration channel to custom states
A configuration channel that is unassigned or is completely used by salt
systems can be migrated to a custom channel. In this case, the autogenerated
`init.sls` would be converted to a new configuration file.

## Show filelist when writing custom states

When writing a state we could show a list of files and provide the possibility
to insert the `salt://` url of a selected file at the current cursor position.

There is a salt runner module which list *all* files of every configured backend.

    salt-run fileserver.file_list

Problem: to which org does a file belong to?

We need to integrate it somehow in the existing UI for editing configuration
files, which might be a bit more challenging than writing a new UI only for SLS
files.


## Add a toolbar for editing custom states

We could add a toolbar to the custom state edit page which provide buttons
which insert standard state snippets of typical tasks.

* `package.installed`
* `service.running`
* `file managed`
* ... etc.

This needs to be integrated into the existing Configuration File page.  It can
be dependent on the type `SLS`.


## Import a managed file from existing system

Sometimes user wants to import an existing (config) file from an existing system.

This exists already in the `Configuration` UI for the traditional systems. We
need to do an implementation for salt as well (note that uploading files might
be problematic with salt-ssh).


## Migration of existing custom states
[migration-existing-custom-states]: #migration-existing-custom-states

Currently, SUMA stores the contents of the custom states on the **filesystem only**.
As we need to move the source of truth to the database, we need to **migrate**
the custom states from the disk to the database.

There is no silver-bullet idea about **when** the migration should happen as all
options have drawbacks:
1. Doing this during `spacewalk-schema-update` would require non-trivial changes
   in the tool,
2. Doing this in the in a separate script would require notifying users about
   the need of running it,
3. Doing this at the tomcat startup could slow the startup down in case there is
   a lot of files to migrate/check,
4. ??? TODO, maybe there are more options.

## Pillar data management
(Joe Werner) Expose suma variables as pillars.


# Drawbacks
[drawbacks]: #drawbacks

Problematic migration of custom states to the new implementation (see above).

Configuration files are rendered in the salt filesystem which is global and do not
know about organization boundaries, the file is readable and usable by every
org. This cannot be changed as long as salt misses multitenancy.

**Only when we decide to assign configuration using the `suseStateRevision` and
not `rhnServerConfigChannel`**: Problem with versioning emerges: each system can
be in a different revision, which could mean different contents of a
configuration file. Due to salt limitations, this is not possible (only one
version of the file can be generated on the filesystems) and the revisioning
would be only possible on the configuration file level. All assigned systems
would see the same (latest) revision of the file.

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?
* use git as a backend, problems:
  * conflict resolution
  * migration
  * hard to reference from git to DB

# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD?
* Agree on custom states migration (see [here](#migration-existing-custom-states))
* We probably need to re-organize the UI menu items a bit, Suggestion from the
  champions:
  * `Salt->Keys/Remote command` - move under `Systems`
  * `Salt->State catalog/Formula catalog` - move under `Configuration`
* (from Richard Schärtel): Priorities of the config channels
* The old UI has the possibility to deploy only one file from the channel.
  This is difficult with salt and an autogenerated init.sls.
* How to do assignment of config channels to systems in salt? Maybe via pillar
  data and jinja templates. Check the current mechanism for package/custom
  states and inspire there.

Task for us:
* go through all UIs and see what must be hidden/modified for the salt systems
* discover whether our ace editor works reasonably for `jinja` or if we need to
  enhance it. Seems that the combination of `yaml` + `jinja` is the biggest
  issue.
