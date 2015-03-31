puppet-preview(8) -- Puppet catalog compilation diff
========

SYNOPSIS
--------
Compiles two catalogs one in the baseline environment, and one in a preview environment
and computes a diff between the two. Produces the two catalogs, the diff, and the
logs from each compilation for inspection.

USAGE
-----
```
puppet preview [
    [--assert equal|compliant]
    [-d|--debug]
    [-l|--last]
    [-m|--migrate]
    [--preview_outputdir <PATH-TO-OUTPUT-DIR>]
    [--skip_tags]
    [--view summary|baseline|preview|diff|baseline_log|preview_log|none]
    [-vd|--verbose_diff]
    [--trusted]
    [--baseline_environment <ENV-NAME> | --be <ENV-NAME>]
    --preview_environment <ENV-NAME> | --pe <ENV-NAME>
    <NODE-NAME>
  ]|[--schema catalog|catalog_delta|help]
   |[-h|--help]
   |[-V|--version]
```

DESCRIPTION
-----------
This command compiles, compares, and asserts a baseline catalog and a preview catalog
for a node that has previously requested a catalog from a puppet master (thereby making
its facts available). The compilation of the baseline catalog takes place in the
environment configured for the node (optionally overridden with '--baseline_environment').
The compilation of the preview takes place in the environment designated by '--preview_env'.

If '--baseline_environment' is set, the node will first be configured as directed by
an ENC, the environment is then switched to the '--baseline_environment'.
The '--baseline_environment' option is intended to aid when changing code in the
preview environment (for the purpose of making it work with the future parser) while
the original environment is unchanged and is configured with the 3.x current parser
(i.e. what is in 'production').
If the intent is to make backwards compatible changes in the preview
environment (i.e. changes that work for both parsers) it is of value to have yet another
environment configured for future parser where the same code as in the preview environment
is checked out. It is then simple to diff between compilations in any two environments
without having to modify the environment assigned by the ENC. All other assignments
made by the ENC are unchanged.

By default the command outputs a summary report of the difference between the two
catalogs on 'stdout'. This can be changed with '--view' to instead view
one of the catalogs, the diff, or one of the compilation logs. Use the '--last' option
to view one of the results from the last preview compilation instead of again compiling
and computing a diff. Note that '--last' does not reload the information and can
therefore not display a summary.

When the preview compilation is performed, it is possible to turn on extra
migration validation using '--migrate'. This will turn on extra validations
of future compatibility flagging puppet code that needs to be reviewed. This
feature was introduced to help with the migration from puppet 3.x to puppet 4.x.
and requires that the '--preview_env' references an environment configured
to use the future parser in its environment.conf.

All output (except the summary report intended for human use) is written in
JSON format to allow further processing with tools like 'jq' (JSON query).

The output is written to a subdirectory named after the node of the directory appointed
by the setting 'preview_outputdir' (defaults to '$vardir/preview'):

    |- "$preview_output-dir"
    |  |
    |  |- <NODE-NAME-1>
    |  |  |- preview_catalog_.json
    |  |  |- baseline_catalog.json
    |  |  |- preview_log.json
    |  |  |- baseline_log.json
    |  |  |- catalog_diff.json
    |  |  
    |  |- <NODE-NAME-2>
    |  |  |- ...
 
Each new invocation of the command for a given node overwrites the information
already produced.

The two catalogs are written in JSON compliant with a json-schema
('catalog.json'; the format used by puppet to represent catalogs in JSON)
viewable on stdout using '--schema catalog'

The 'catalog_diff.json' file is written in JSON compliant with a json-schema
viewable on stdout using '--schema catalog_delta'.

OPTIONS
-------

Note that any Puppet setting that's valid in the configuration file is also a
valid long argument. For example, 'server' is a valid setting, so you can
specify '--server <servername>' as an argument. Boolean settings translate into
'--setting' and '--no-setting' pairs.

See the configuration file documentation at
http://docs.puppetlabs.com/references/stable/configuration.html for the
full list of acceptable settings. A commented list of all settings can also be
generated by running puppet master with '--genconfig'.

Note that all settings such as 'log_level' affects both compilations.

* --debug:
  Enable full debugging. Debugging output is sent to the respective log outputs
  for baseline and preview compilation. This option is for both compilations.
  Note that debugging information for the startup and end of the application
  itself is sent to the console.

* --help:
  Print this help message.

* --version:
  Print the puppet version number and exit.

* --preview_environment <ENV-NAME>
  Makes the preview compilation take place in the given <ENV-NAME>.
  Uses facts obtained from the configured facts terminus to compile the catalog.

* --baseline_environment <ENV-NAME>
  Makes the baseline compilation take place in the given <ENV-NAME>. This overrides
  the environment set for the node via an ENC.
  Uses facts obtained from the configured facts terminus to compile the catalog.
  Note that the puppet setting '-environment' cannot be used to achieve the same effect.

* --view summary | diff | baseline | preview | baseline_log | preview_log | none
  Specifies what will be output on stdout; the catalog diff, one of the two
  catalogs, or one of the two logs. The option 'none' turns off output to stdout.

* --migrate
  Turns on migration validation for the preview compilation. Validation result
  is produced to the preview log file or optionally to stdout with --view preview_log

* --assert equal | compliant
  Modifies the exit code to be 4 if catalogs are not equal and 5 if the preview
  catalog is not compliant instead of an exit with 0 to indicate that the preview run
  was successful in itself. 

* --preview_outputdir <DIR>
  Defines the directory to which output is produced.
  This is a puppet setting that can be overridden on the command line.

* <NODE-NAME>
  This specifies for which node the preview should produce output. The node must
  have previously requested a catalog from the master to make its facts available.

* --schema catalog | catalog_delta | help
  Outputs the json-schema for the puppet catalog, or for the catalog_delta. The option
  'help' will display the semantics of the catalog-diff schema. Can not be combined with
  any other option.

* --skip_tags
  Ignores comparison of tags, catalogs are considered equal/compliant if they only
  differ in tags.

* --trusted
  Makes trusted node data obtained from a fact terminus retain its authentication
  status of "remote", "local", or false.(the authentication status the write request had).
  If this option is not in effect, any trusted node information is kept, and the
  authenticated key is set to false. The --trusted option is only available when running
  as root, and should only be turned on when also trusting the fact store.

* --verbose_diff
  Includes more information in the catalog diff such as attribute values in
  missing and added resources. Does not affect if catalogs are considered equal or
  compliant.

* --last
  Use the last result obtained for the node instead of performing new compilations
  and diff. (Cannot be combined with --view none or --view summary).

EXAMPLE
-------
To perform a full migration preview that exists with failure if catalogs are not equal:

    puppet preview --preview_env future_production --migrate --assert=equal mynode
    
To perform a preview that exits with failure if preview catalog is not compliant:

    puppet preview --preview_env future_production --assert=compliant mynode

To perform a preview focusing on if code changes resulted in conflicts in
resources of File type using 'jq' to filter the output (the command is given as one line):

    puppet preview --preview_env future_production --view diff mynode 
    | jq -f '.conflicting_resources | map(select(.type == "File"))'
    
View the catalog schema:

    puppet preview --schema catalog
    
View the catalog-diff schema:

    puppet preview --schema catalog_diff
    
Run a diff (with the default summary view) then view the preview log:

    puppet preview --preview_env future_production mynode
    puppet preview --preview_env future_production mynode --view preview_log --last

Node name can be placed anywhere:

    puppet preview mynode --preview_env future_production

    
DIAGNOSTICS
-----------
The '--assert' option controls the exit code of the command.

If '--assert' is not specified the command will exit with 0 if the two compilations
succeeded, 2 if the baseline compilation failed (a catalog could not be
produced), and 3 if the preview compilation did not produce a catalog. Files not produced
may either not exist, or be empty.

If '--assert' is set to 'equal', the command will exit with 4 if the two catalogs
are not equal.

If --assert is set to 'compliant' it will exit with 5 if the content of the
baseline catalog is not a subset of the content of the preview catalog.

The different assert values do not alter what is produced - only the exit value is
different as both equality and compliance is checked in every preview.

The command exits with 1 if there is a general error.

AUTHOR
------
Henrik Lindberg, Puppet Labs


COPYRIGHT
---------
Copyright (c) 2015 Puppet Labs, LLC Licensed under Puppet Labs Commercial.
