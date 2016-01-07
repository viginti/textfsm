# Table of Contents #



# Overview #

This codelab is designed to help users of the TextFSM python module in writing new templates. This flexible generic text parser is primarily used for parsing the output of router CLI commands in Python. By writing relatively simple regex based text files, a multitude of command outputs can be parsed with minimal code.

This codelab starts with the output of a simple router command, and will guide you in the steps required to successfully write a template and use it in a simple Python script.

You should already have a basic understanding in Python and have read the TextTableFSM Howto page, and have it open for reference.

# First example - basic text #

For the first example, we will parse simple linear (non-row based) text - the output of a Cisco 'show clock':

```
18:42:41.321 PST Sun Feb 8 2009
```

Only the actual date/time is going to be parsed - we don't parse the prompts. The information that we'd like to extract here is:

  * Year
  * Day of month
  * Month
  * Timezone
  * Time (to the second)

Each of these bits of information will be extracted into its own FSM 'Value', which will make each one individually available to a Python script using it. For this example, we will intentionally not try to match the weekday. Thus the first thing we need to do is populate 'Value' lines into our template.

Open a new file, called `cisco_clock_template`, and put the following lines into it:

```
Value Year (\d+)
Value MonthDay (\d+)
Value Month (\w+)
Value Timezone (\S+)
Value Time (..:..:..)

```

These are our Value definitions. On each line, the first word is the keyword 'Value', and tells the FSM that the line defines a column in a row. The next word is the name of the value - this can be any alphanumeric word. Lastly we have a regex containing at least one paren-wrapped expression. These are sub-expressions containing a regex to match exactly the value required. For example, the `(\d+)` will match the numeric digit sequence denoting a year.

Insert a blank line after the 'Time' value definition. This denotes the end of the 'Value' definitions. After this, we start defining states of our FSM. The state 'Start' is always required, and the FSM will begin here when first parsing text. So the next line of the FSM has the single word 'Start':

```
Value Year (\d+)
Value MonthDay (\d+)
Value Month (\w+)
Value Timezone (\S+)
Value Time (..:..:..)

Start
```

Following the state label, we insert one or more rules. When the state machine enters a new state, it takes its current line of input, and attempts to match it against each rule in turn. Once matched, any 'Value's are substituted and optional actions are taken. This template will have only one rule, so complete the template as follows, and save it as `cisco_clock_template`:

```
Value Year (\d+)
Value MonthDay (\d+)
Value Month (\w+)
Value Timezone (\S+)
Value Time (..:..:..)

Start
  ^${Time}.* ${Timezone} \w+ ${Month} ${MonthDay} ${Year} -> Record
```

Internally, the rule(s) will be expanded, such that any reference to a Value will be replaced by the regex specified in the Value. The single rule in this template will be expanded internally to this regex:

```
^(..:..:..).* (\S+) \w+ (\w+) (\d+) (\d+)
```

When the single line of input (18:42:41.321 PST Sun Feb 8 2009) is applied to it, the regex will match, and each of the regex match groups will contain the following values:

| **Group** |	**Value** |
|:----------|:----------|
| 1         | 18:42:41  |
| 2         | PST       |
| 3         | Feb       |
| 4         | 8         |
| 5         | 2009      |

Because of the successful match, each of the Values associated with the group will be assigned the matching text. This will fill the one (and only) row.

We also have an associated action Record. This instructs the FSM to insert the row into the final table. There is also an implicit 'Next' action (see TextTableFSM that tells the FSM to read the next line of text in. As there is no further text (we only have one line), the FSM completes and returns just the single matched row.

We will try the FSM on some input text:

```
$ cat > router_output.txt
18:42:41.321 PST Sun Feb 8 2009
$ ./textfsm.py cisco_clock_template router_output.txt
FSM Template:
Value Year (\d+)
Value MonthDay (\d+)
Value Month (\w+)
Value Timezone (\S+)
Value Time (..:..:..)

Start
  ^${Time}.* ${Timezone} \w+ ${Month} ${MonthDay} ${Year} -> Record
```

FSM Table:

```
['Year', 'MonthDay', 'Month', 'Timezone', 'Time']
['2009', '8', 'Feb', 'PST', '18:42:41']
```

Success! The first section of the output shows the parsed FSM, and the last section shows the final table. The first row of this table is the header, showing the columns (or 'Value's) that we've defined. The next line is the row of values that were matched.

# Multiple lines #

We'll now look at something slightly more complicated - matching values from multiple lines of text. For this, we will attempt to parse the output of a Cisco 'show version' command. It looks like this:

```
Cisco IOS Software, Catalyst 4500 L3 Switch Software (cat4500-ENTSERVICESK9-M), Version 12.2(31)SGA1, RELEASE SOFTWARE (fc3)
Technical Support: http://www.cisco.com/techsupport
Copyright (c) 1986-2007 by Cisco Systems, Inc.
Compiled Fri 26-Jan-07 14:28 by kellythw
Image text-base: 0x10000000, data-base: 0x118AD800

ROM: 12.2(31r)SGA
Pod Revision 0, Force Revision 34, Gill Revision 20

router.abc uptime is 11 weeks, 4 days, 20 hours, 26 minutes
System returned to ROM by reload
System restarted at 22:49:40 PST Tue Nov 18 2008
System image file is "bootflash:cat4500-entservicesk9-mz.122-31.SGA1.bin"


This product contains cryptographic features and is subject to United
States and local country laws governing import, export, transfer and
use. Delivery of Cisco cryptographic products does not imply
third-party authority to import, export, distribute or use encryption.
Importers, exporters, distributors and users are responsible for
compliance with U.S. and local country laws. By using this product you
agree to comply with applicable laws and regulations. If you are unable
to comply with U.S. and local laws, return this product immediately.

A summary of U.S. laws governing Cisco cryptographic products may be found at:
http://www.cisco.com/wwl/export/crypto/tool/stqrg.html

If you require further assistance please contact us by sending email to
export@cisco.com.

cisco WS-C4948-10GE (MPC8540) processor (revision 5) with 262144K bytes of memory.
Processor board ID FOX111700ZN
MPC8540 CPU at 667Mhz, Fixed Module
Last reset from Reload
2 Virtual Ethernet interfaces
48 Gigabit Ethernet interfaces
2 Ten Gigabit Ethernet interfaces
511K bytes of non-volatile configuration memory.

Configuration register is 0x2102
```

We will be extracting several bits of information from this:

  * Software version
  * System uptime
  * Configuration register
  * Reset reason

This gives us four columns in our table (which will end up as a single row). To start, we will thus define our 'Value' statements. Start with a new file, called 'cisco\_version\_template' and put the following into it:

```
Value Version ([^ ,]+)
Value Uptime (.*)
Value ConfigRegister (\w+)
Value ResetReason (.*)

Start
  ^Cisco IOS .*Version ${Version},
  ^.*uptime is ${Uptime}
  ^Last reset from ${ResetReason}
  ^Configuration register is ${ConfigRegister} -> Record
```

This is a little more complicated that the first example. Most importantly, the template only matches some of the input lines. For each line of input, each rule in the state is checked. If there is no match with a rule, the FSM simply moves to the next rule. Once all rules have been checked (in this case, after the 'Configuration register' rule), then the next line of input is fetched, and rule are checked from the start of the state again.

So the following will happen here:

The first line of input will be matched, and the 'Version' value is assigned "12.2(31)SGA1".
Until the 'uptime is' line, all other lines fail to match and are discarded.
The 'uptime' line will match, and 'Uptime' is assigned "11 weeks, 4 days, 20 hours, 26 minutes"
Lines are not matched, and thus discarded, until the 'Last reset' line
At this point, there is a match and 'ResetReason' value is assigned 'Reload'
Several more lines pass until 'ConfigRegister' is assigned '0x2102'
This line also triggers a 'Record' action, and the row is saved.
After this line, we reach EOF and the FSM terminates.
Run it against the FSM, after saving your template and copying the router output to 'router\_output.txt':

```
$;./textfsm.py cisco_version_template router_output.txt
FSM Template:
Value Version ([^ ,]+)
Value Uptime (.*)
Value ConfigRegister (\w+)
Value ResetReason (.*)

Start
  ^Cisco IOS .*Version ${Version},
  ^.*uptime is ${Uptime}
  ^Last reset from ${ResetReason}
  ^Configuration register is ${ConfigRegister} -> Record


FSM Table:
['Version', 'Uptime', 'ConfigRegister', 'ResetReason']
['12.2(31)SGA1', '11 weeks, 4 days, 20 hours, 26 minutes', '0x2102', 'Reload']
```

# Parsing tabular data #

The previous examples showed the simplest application of the FSM - parsing non-repeating data into a single row. In this example, we will look at a more interesting example - the output of the Juniper command show chassis fpc will be parsed. The raw router output from one device looks like this:

```
                     Temp  CPU Utilization (%)   Memory    Utilization (%)
Slot State            (C)  Total  Interrupt      DRAM (MB) Heap     Buffer
  0  Online            24      7          0       256        38         51
  1  Online            25      7          0       256        38         51
  2  Online            24      3          0       256        37         49
  3  Online            23      3          0       256        37         49
  4  Empty
  5  Empty
  6  Empty
  7  Empty
```

This text is simple to parse - its information is neatly contained in separate rows, and the FSMs data is always presented as rows. This compatibility will ensure a relatively painless template creation. For our application we'll create a single row of data for each FPC defined, but this will be easy as the text is presented as one line per FPC slot.

The information we want to extract is:

  * Slot number
  * State
  * Temperature
  * DRAM
  * Buffer utilisation

This give us five values of interest. Create a new file juniper\_fpc\_template, and first put in it the following Value definitions:

```
Value Slot (\d)
Value State (\w+)
Value Temperature (\d+)
Value DRAM (\d+)
Value Buffer (\d+)
```

Next we put in the State definition and associated rules. We may be able to get away with a single rule.

```
Value Slot (\d)
Value State (\w+)
Value Temperature (\d+)
Value DRAM (\d+)
Value Buffer (\d+)

Start
  # We can't use .* for unused placeholders, as greedy matching will break it.
  ^\s+${Slot}\s+${State}\s+${Temperature}\s+\d+\s+\d+\s+${DRAM}\s+\d+\s+${Buffer} -> Record
```

Copy the above raw router output to `router_output.txt`, save the template, and run it:

```
$ ./textfsm.py juniper_fpc_template router_output.txt
FSM Template:
Value Slot (\d)
Value State (\w+)
Value Temperature (\d+)
Value DRAM (\d+)
Value Buffer (\d+)

Start
  ^\s+${Slot}\s+${State}\s+${Temperature}\s+\d+\s+\d+\s+${DRAM}\s+\d+\s+${Buffer} -> Record


FSM Table:
['Slot', 'State', 'Temperature', 'DRAM', 'Buffer']
['0', 'Online', '24', '256', '51']
['1', 'Online', '25', '256', '51']
['2', 'Online', '24', '256', '49']
['3', 'Online', '23', '256', '49']
```

Our template has successfully parsed all of the online FPC slots and extracted the correct values.

But what about empty slots? If we need to also extract information about every slot, then we won't see them here as the empty ones only have the Slot and State columns. The way to deal with this is to create a second rule to catch these lines.

Add one more rule after the existing one, as follows:

```
Value Slot (\d)
Value State (\w+)
Value Temperature (\d+)
Value DRAM (\d+)
Value Buffer (\d+)

State
  # We can't use .* for unused placeholders, as greedy matching will break it.
  ^\s+${Slot}\s+${State}\s+${Temperature}\s+\d+\s+\d+\s+${DRAM}\s+\d+\s+${Buffer} -> Record
  ^\s+${Slot}\s+${State} -> Record
```

Now run the FSM:

```
$;./textfsm.py juniper_fpc_template router_output.txt
FSM Template:
Value Slot (\d)
Value State (\w+)
Value Temperature (\d+)
Value DRAM (\d+)
Value Buffer (\d+)

Start
  ^\s+${Slot}\s+${State}\s+${Temperature}\s+\d+\s+\d+\s+${DRAM}\s+\d+\s+${Buffer} -> Record
  ^\s+${Slot}\s+${State} -> Record


FSM Table:
['Slot', 'State', 'Temperature', 'DRAM', 'Buffer']
['0', 'Online', '24', '256', '51']
['1', 'Online', '25', '256', '51']
['2', 'Online', '24', '256', '49']
['3', 'Online', '23', '256', '49']
['4', 'Empty', '', '', '']
['5', 'Empty', '', '', '']
['6', 'Empty', '', '', '']
['7', 'Empty', '', '', '']
```

Success! We've now extracted info about both filled and empty slots.

# Using 'Filldown' #

Now that we can extract a table from show chassis fpc, we have to do an enhancement. Whilst it works with the simple example above, it isn't entirely effective for the more complicated output from a Juniper TX matrix, which looks like this:

```
lcc0-re0:
--------------------------------------------------------------------------
                     Temp  CPU Utilization (%)   Memory    Utilization (%)
Slot State            (C)  Total  Interrupt      DRAM (MB) Heap     Buffer
  0  Online            24      8          1       512        16         52
  1  Online            23      7          1       256        36         53
  2  Online            23      5          1       256        36         49
  3  Online            21      7          1       256        36         49
  4  Empty
  5  Empty
  6  Empty
  7  Empty

lcc1-re1:
--------------------------------------------------------------------------
                     Temp  CPU Utilization (%)   Memory    Utilization (%)
Slot State            (C)  Total  Interrupt      DRAM (MB) Heap     Buffer
  0  Online            20      9          1       256        36         50
  1  Online            20     13          0       256        36         49
  2  Online            21      6          1       256        36         49
  3  Online            20      6          0       256        36         49
  4  Online            18      5          0       256        35         49
  5  Empty
  6  Empty
  7  Empty
```

There are two physical chassis in this node (lcc0-re0 and lcc1-re0), and we have two of each slot number, one per chassis.

If we run this output against the FSM, we will see the following:

```
FSM Template:
Value Slot (\d)
Value State (\w+)
Value Temperature (\d+)
Value DRAM (\d+)
Value Buffer (\d+)

Start
  ^\s+${Slot}\s+${State}\s+${Temperature}\s+\d+\s+\d+\s+${DRAM}\s+\d+\s+${Buffer} -> Record
  ^\s+${Slot}\s+${State} -> Record


FSM Table:
['Slot', 'State', 'Temperature', 'DRAM', 'Buffer']
['0', 'Online', '24', '512', '52']
['1', 'Online', '23', '256', '53']
['2', 'Online', '23', '256', '49']
['3', 'Online', '21', '256', '49']
['4', 'Empty', '', '', '']
['5', 'Empty', '', '', '']
['6', 'Empty', '', '', '']
['7', 'Empty', '', '', '']
['0', 'Online', '20', '256', '50']
['1', 'Online', '20', '256', '49']
['2', 'Online', '21', '256', '49']
['3', 'Online', '20', '256', '49']
['4', 'Online', '18', '256', '49']
['5', 'Empty', '', '', '']
['6', 'Empty', '', '', '']
['7', 'Empty', '', '', '']
```

Unfortunately there's no way to see if the 512Mb FPC is in lcc0 or lcc1, because we have not stored and chassis info. So we can do this by adding another value 'Chassis' which matches ^\S+:. Update the template by adding the new Value and rule:

```
Value Chassis (\S+)
Value Slot (\d)
Value State (\w+)
Value Temperature (\d+)
Value DRAM (\d+)
Value Buffer (\d+)

Start
  ^${Chassis}:
  ^\s+${Slot}\s+${State}\s+${Temperature}\s+\d+\s+\d+\s+${DRAM}\s+\d+\s+${Buffer} -> Record
  ^\s+${Slot}\s+${State} -> Record
```

But when this is run, there is a slight problem:

```
FSM Template:
Value Chassis (\S+)
Value Slot (\d)
Value State (\w+)
Value Temperature (\d+)
Value DRAM (\d+)
Value Buffer (\d+)

Start
  ^${Chassis}:
  ^\s+${Slot}\s+${State}\s+${Temperature}\s+\d+\s+\d+\s+${DRAM}\s+\d+\s+${Buffer} -> Record
  ^\s+${Slot}\s+${State} -> Record


FSM Table:
['Chassis', 'Slot', 'State', 'Temperature', 'DRAM', 'Buffer']
['lcc0-re0', '0', 'Online', '24', '512', '52']
['', '1', 'Online', '23', '256', '53']
['', '2', 'Online', '23', '256', '49']
['', '3', 'Online', '21', '256', '49']
['', '4', 'Empty', '', '', '']
['', '5', 'Empty', '', '', '']
['', '6', 'Empty', '', '', '']
['', '7', 'Empty', '', '', '']
['lcc1-re1', '0', 'Online', '20', '256', '50']
['', '1', 'Online', '20', '256', '49']
['', '2', 'Online', '21', '256', '49']
['', '3', 'Online', '20', '256', '49']
['', '4', 'Online', '18', '256', '49']
['', '5', 'Empty', '', '', '']
['', '6', 'Empty', '', '', '']
['', '7', 'Empty', '', '', '']
```

The 'Chassis' column is only filled for the first slot in each chassis. Why?

The problem is that each time a 'Record' action is taken, the row is saved, then each Value is cleared. Thus when corresponding rows are filled and saved, there is no longer anything to match the 'Chassis' rule, so this never gets filled until the next chassis specification in the router output.

To fix this, we will add the 'Filldown' option to the Chassis value. This option tells the FSM to retain the value after each 'Record', so that it isnt cleared. Its value will only change if there is either a 'Clearall' action, or if a rule overwrites the value.

Modify the Chassis Value line with a 'Filldown' option, as follows:

```
Value Filldown Chassis (\S+)
```

Now run the text through the FSM again:

```
$; /textfsm.py juniper_fpc_template router_output.txt
FSM Template:
Value Filldown Chassis (\S+)
Value Slot (\d)
Value State (\w+)
Value Temperature (\d+)
Value DRAM (\d+)
Value Buffer (\d+)

Start
  ^${Chassis}:
  ^\s+${Slot}\s+${State}\s+${Temperature}\s+\d+\s+\d+\s+${DRAM}\s+\d+\s+${Buffer} -> Record
  ^\s+${Slot}\s+${State} -> Record


FSM Table:
['Chassis', 'Slot', 'State', 'Temperature', 'DRAM', 'Buffer']
['lcc0-re0', '0', 'Online', '24', '512', '52']
['lcc0-re0', '1', 'Online', '23', '256', '53']
['lcc0-re0', '2', 'Online', '23', '256', '49']
['lcc0-re0', '3', 'Online', '21', '256', '49']
['lcc0-re0', '4', 'Empty', '', '', '']
['lcc0-re0', '5', 'Empty', '', '', '']
['lcc0-re0', '6', 'Empty', '', '', '']
['lcc0-re0', '7', 'Empty', '', '', '']
['lcc1-re1', '0', 'Online', '20', '256', '50']
['lcc1-re1', '1', 'Online', '20', '256', '49']
['lcc1-re1', '2', 'Online', '21', '256', '49']
['lcc1-re1', '3', 'Online', '20', '256', '49']
['lcc1-re1', '4', 'Online', '18', '256', '49']
['lcc1-re1', '5', 'Empty', '', '', '']
['lcc1-re1', '6', 'Empty', '', '', '']
['lcc1-re1', '7', 'Empty', '', '', '']
['lcc1-re1', '', '', '', '', '']
```

Now we have the Chassis values all 'filled down' in the table. It's almost perfect except for one problem. There is an odd almost-empty row at the bottom of the table. What is it doing there?

# Using 'Required' #

That row comes about because of the Filldown option. When the last valid row is saved (lcc-re1, slot 7), the FSM creates a new empty row ready to have values filled in. Normally the FSM will discard empty rows when it terminates, but in this case the 'Filldown' option has populated the 'Chassis' column, so the FSM keeps this non-empty row and saves it when the FSM terminates.

In order to fix this, we will make use of another Value option - Required. This option specifies that the value must have been matched otherwise a row will not be saved. For this template, we'll ensure that 'slot' and 'state' both contain a value, so we'll modify as follows:

```
Value Required Slot (\d)
Value Required State (\w+)
Now we finally have a correct table:

FSM Template:
Value Filldown Chassis (\S+)
Value Required Slot (\d)
Value Required State (\w+)
Value Temperature (\d+)
Value DRAM (\d+)
Value Buffer (\d+)

Start
  ^${Chassis}:
  ^\s+${Slot}\s+${State}\s+${Temperature}\s+\d+\s+\d+\s+${DRAM}\s+\d+\s+${Buffer} -> Record
  ^\s+${Slot}\s+${State} -> Record


FSM Table:
['Chassis', 'Slot', 'State', 'Temperature', 'DRAM', 'Buffer']
['lcc0-re0', '0', 'Online', '24', '512', '52']
['lcc0-re0', '1', 'Online', '23', '256', '53']
['lcc0-re0', '2', 'Online', '23', '256', '49']
['lcc0-re0', '3', 'Online', '21', '256', '49']
['lcc0-re0', '4', 'Empty', '', '', '']
['lcc0-re0', '5', 'Empty', '', '', '']
['lcc0-re0', '6', 'Empty', '', '', '']
['lcc0-re0', '7', 'Empty', '', '', '']
['lcc1-re1', '0', 'Online', '20', '256', '50']
['lcc1-re1', '1', 'Online', '20', '256', '49']
['lcc1-re1', '2', 'Online', '21', '256', '49']
['lcc1-re1', '3', 'Online', '20', '256', '49']
['lcc1-re1', '4', 'Online', '18', '256', '49']
['lcc1-re1', '5', 'Empty', '', '', '']
['lcc1-re1', '6', 'Empty', '', '', '']
['lcc1-re1', '7', 'Empty', '', '', '']
```

# Using 'List' #

Often it may be necessary to have a particular column contain a list of values. For example when parsing a routing table, there may be multiple next hops per prefix. The below example is such a routing table, simplified for the purposes of this example:

```
       Destination        Gateway                      Dist/Metric Last Change
       -----------        -------                      ----------- -----------
  B EX 0.0.0.0/0          via 192.0.2.73                  20/100        4w0d
                          via 192.0.2.201
                          via 192.0.2.202
                          via 192.0.2.74
  B IN 192.0.2.76/30     via 203.0.113.183                200/100        4w2d
  B IN 192.0.2.204/30    via 203.0.113.183                200/100        4w2d
  B IN 192.0.2.80/30     via 203.0.113.183                200/100        4w2d
  B IN 192.0.2.208/30    via 203.0.113.183                200/100        4w2d
```

For this example, we will extract the following information:

  * Protocol
  * Type
  * Destination prefix
  * Gateway (for simplicity's sake we will assume all entries will be of the form "via a.b.c.d")
  * Distance
  * Metric
  * Last Change

Copy the above routing table output to a file called 'routes.txt'.

We will build a simple FSM to extract data. Put the below text into a file called 'routes.tmpl':

```
Value Protocol (\S)
Value Type (\S\S)
Value Required Prefix (\S+)
Value Gateway (\S+)
Value Distance (\d+)
Value Metric (\d+)
Value LastChange (\S+)

Start
  ^.*----- -> Routes

Routes
  ^  ${Protocol} ${Type} ${Prefix}\s+via ${Gateway}\s+${Distance}/${Metric}\s+${LastChange} -> Record
```

Note above that in the 'Start' state we have a special match for a line of dashes. This implicitly discards input text until after the headers are complete. It's not really needed or this case, but is useful to demonstrate for other cases where header data may be confused with actual row data.

Now run the template against the data:

```
$ ./textfsm.py route.tmpl routes.txt
FSM Template:
Value Protocol (\S)
Value Type (\S\S)
Value Prefix (\S+)
Value Gateway (\S+)
Value Distance (\d+)
Value Metric (\d+)
Value LastChange (\S+)

Start
  ^.*----- -> Routes

Routes
  ^  ${Protocol} ${Type} ${Prefix}\s+via ${Gateway}\s+${Distance}/${Metric}\s+${LastChange} -> Record


FSM Table:
['Protocol', 'Type', 'Prefix', 'Gateway', 'Distance', 'Metric', 'LastChange']
['B', 'EX', '0.0.0.0/0', '192.0.2.73', '20', '100', '4w0d']
['B', 'IN', '192.0.2.76/30', '203.0.113.183', '200', '100', '4w2d']
['B', 'IN', '192.0.2.204/30', '203.0.113.183', '200', '100', '4w2d']
['B', 'IN', '192.0.2.80/30', '203.0.113.183', '200', '100', '4w2d']
['B', 'IN', '192.0.2.208/30', '203.0.113.183', '200', '100', '4w2d']
```

The table is mostly filled...except that we only see the first Gateway for the default prefix. It's easy to see why - the template does not match the lines defining the other gateways. So how to work around this?

The option 'List' is useful here, as it can be used to specify a column as a list of strings, rather than just a single string. So we will add this option to the 'Gateway' value definition:

```
Value Protocol (\S)
Value Type (\S\S)
Value Required Prefix (\S+)
Value List Gateway (\S+)
Value Distance (\d+)
Value Metric (\d+)
Value LastChange (\S+)
Run it again, and we see that the Gateway columns are now a list, albeit with one entry:


FSM Table:
['Protocol', 'Type', 'Prefix', 'Gateway', 'Distance', 'Metric', 'LastChange']
['B', 'EX', '0.0.0.0/0', ['192.0.2.73'], '20', '100', '4w0d']
['B', 'IN', '192.0.2.76/30', ['203.0.113.183'], '200', '100', '4w2d']
['B', 'IN', '192.0.2.204/30', ['203.0.113.183'], '200', '100', '4w2d']
['B', 'IN', '192.0.2.80/30', ['203.0.113.183'], '200', '100', '4w2d']
['B', 'IN', '192.0.2.208/30', ['203.0.113.183'], '200', '100', '4w2d']
```

To extract the other Gateway entries, we need to do this in two steps:

Not 'Record' until we start the next Prefix entry (in case of additional Gateway lines)., and
Parse the other Gateway lines.
To take care of the first, we will need to do make two changes. Firstly we remove the existing 'Record' statement. Secondly, we add a statement before the existing one that upon matching a new Prefix entry it 'Record's the existing one, then continues with the current input line:

```
Routes
  ^  \S \S\S -> Continue.Record
  ^  ${Protocol} ${Type} ${Prefix}\s+via ${Gateway}\s+${Distance}/${Metric}\s+${LastChange}
```

The new line does not use value substitutions, as we need to save the previous prefix entry records, not overwrite them. The 'Continue' statement then resumes at the next rule after 'Record'ing the previous data, using the current input line. If we run this now, we see the output is identical to before.

Step two in getting the List to work here is to create a rule to match additional Gateway entries. This will go at the end. Note that we do not Record here - that happens only when the next entry is processed. Remember also that an EOF will also cause the current entry to be 'Record'ed.

```
Routes
  ^  \S \S\S -> Continue.Record
  ^  ${Protocol} ${Type} ${Prefix}\s+via ${Gateway}\s+${Distance}/${Metric}\s+${LastChange}
  ^\s+via ${Gateway}
```

Now we run it, and note that we now have our list of all Gateway entries for the default Prefix:

```
FSM Table:
['Protocol', 'Type', 'Prefix', 'Gateway', 'Distance', 'Metric', 'LastChange']
['B', 'EX', '0.0.0.0/0', ['192.0.2.73', '192.0.2.201', '192.0.2.202', '192.0.2.74'], '20', '100', '4w0d']
['B', 'IN', '192.0.2.76/30', ['203.0.113.183'], '200', '100', '4w2d']
['B', 'IN', '192.0.2.204/30', ['203.0.113.183'], '200', '100', '4w2d']
['B', 'IN', '192.0.2.80/30', ['203.0.113.183'], '200', '100', '4w2d']
['B', 'IN', '192.0.2.208/30', ['203.0.113.183'], '200', '100', '4w2d']
```

This example was somewhat more complex than the average List usage. If there is a clear delimiter between rows, then that can be used for the Record rather than having to do the 'forward match' style here.

# Using TextFSM in Python #

By itself, the FSM is not entirely useful. It's intended to be used by Python programs for parsing of textual information. Here we will work through a short example of using the FSM within a Python script.

To start, create a new file called 'routes.py', and add the following into it:

```
#!/usr/bin/python2.4

import textfsm
import sys

# Open the template file, and initialise a new TextFSM object with it.
template_file = sys.argv[1]
fsm = textfsm.TextFSM(open(template_file))

# Read stdin until EOF, then pass this to the FSM for parsing.
input_data = sys.stdin.read()
fsm_results = fsm.ParseText(input_data)

print 'Header:'
print fsm.header

print 'Prefix             | Gateway(s)'
print '-------------------------------'

for row in fsm_results:
  print '%-18.18s  %s' % (row[2], ', '.join(row[3]))
```

Make it executable, and run it as follows:

```
$ ./routes.py routes.tmpl < routes.txt
```

Examine the output, and note how each row 'Record'ed in the FSM is represented as a single list type entry in the fsm\_results list. Also note that the 3rd entry in the row is a list type, which we connect with a join(). The header, as a list of Value names, is printed also.