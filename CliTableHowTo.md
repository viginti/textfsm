# Introduction #

Class that reads CLI output and parses into tabular format. Marrying cli command with additional meta data in order to select the appropriate !TextFSM template for parsing the command output.

# Details #

Uses an index file that contains a table for mapping command strings to templates.

## Defining the Index Table ##

An index table corresponding to the templates supplied in the examples directory looks like the following:

```
# First line is the header fields for columns and is mandatory.
# Regular expressions are supported in all fields except the first.
# Last field supports variable length command completion.
# abc[[xyz]] is expanded to abc(x(y(z)?)?)?, regexp inside [[]] is not supported
#
Template, Hostname, Vendor, Command
cisco_bgp_summary_template, .*, Cisco, sh[[ow]] ip bg[[p]] su[[mmary]]
cisco_version_template, .*, Cisco, sh[[ow]] ve[[rsion]]
f10_ip_bgp_summary_template, .*, Force10, sh[[ow]] ip bg[[p]] sum[[mary]]
f10_version_template, .*, Force10, sh[[ow]] ve[[rsion]]
juniper_bgp_summary_template, .*,  Juniper, sh[[ow]] bg[[p]] su[[mmary]]
juniper_version_template, .*, Juniper, sh[[ow]] ve[[rsion]]
unix_ifcfg_template, hostname[abc].*, .*, ifconfig
```

The templates are defined in the first column and the CLI command is defined in the last row. These columns are special and their location is fixed. There is then an optional number of columns in between that can be used to match meta data.

In our example there have been defined two optional columns for the attributes (meta data) **Hostname** and **Vendor**.

## Using the library ##

```
import clitable

# Read in some example command outputs.
raw_cmd_data1 = open('examples/cisco_version_example').read()
raw_cmd_data2 = open('examples/f10_version_example').read()

# Initialise CliTable with the content of the 'index' file.
cli_table = clitable.CliTable('index', 'examples')

# Firstly we will use attributes to match a 'show version' command on a Cisco device.
attributes = {'Command': 'show version' , 'Vendor': 'Cisco'}

cli_table.ParseCmd(raw_cmd_data1, attributes)
print cli_table
>>> Model, Memory, ConfigRegister, Uptime, Version, ReloadReason, ReloadTime, ImageFile
>>> WS-C4948-10GE, 262144K, 0x2102, 3 days, 13 hours, 53 minutes, 12.2(31)SGA1, reload, 05:09:09 PDT Wed Apr 2 2008, bootflash:cat4500-entservicesk9-mz.122-31.SGA1.bin

# This time, the same command but on a Force10.
attributes['Vendor'] = 'Force10'

cli_table.ParseCmd(raw_cmd_data2, attributes)
print cli_table
>>> Chassis, Model, Software, Image
>>> E1200, E1200, 7.7.1.1, flash://FTOS-EF-7.7.1.1.bin
```

The command column in the index file uses the special **[[ ]]** syntax for variable length command completion.
```
# Incomplete command strings are supported.
attributes['Command'] = 'sh vers'

cli_table.ParseCmd(raw_cmd_data2, attributes)
print cli_table
>>> Chassis, Model, Software, Image
>>> E1200, E1200, 7.7.1.1, flash://FTOS-EF-7.7.1.1.bin
```

CliTables with identical column names can be added together.

```
ct = cli_table + cli_table
print ct
>>> Chassis, Model, Software, Image
>>> E1200, E1200, 7.7.1.1, flash://FTOS-EF-7.7.1.1.bin
>>> E1200, E1200, 7.7.1.1, flash://FTOS-EF-7.7.1.1.bin
```

CliTables are built upon a TextTable object, and can use all the features in that class.

```
ct.Append(('E1200', 'E1200', '7.7.2.2', 'flash://FTOS-EF-7.7.2.2.bin'))
print ct.FormattedTable()
>>>  Chassis  Model  Software  Image                       
>>> =======================================================
>>>  E1200    E1200  7.7.1.1   flash://FTOS-EF-7.7.1.1.bin 
>>>  E1200    E1200  7.7.1.1   flash://FTOS-EF-7.7.1.1.bin 
>>>  E1200    E1200  7.7.2.2   flash://FTOS-EF-7.7.2.2.bin 
```