---
title: Writing Metasploit Modules with Python
layout: post
author: cdm
description: A guide for developing new modules for the Metasploit-Framework using Python
---
# Writing Metasploit Modules with Python

The following information will likely change as the newly implemented Python featureset is built out.

## Requirements

Ensure that the version of Metasploit-Framework you are using is up to date, and the developer version.
You can determine which version you are running by starting up a metasploit environment and typing version once initialized:

```bash
#: msfconsole
msf5 > version
```

The version used for this write up:

```bash
msf5 > version
Framework: 5.0.91-dev
Console  : 5.0.91-dev
```

## Debugging

If you're having issues with your module importing, you can run log after initializing the console to see any errors reported:

```bash
msf5 > log
```

## Style Guidelines

These are the guidelines provided by [jrobles-r7](https://github.com/jrobles-r7) with Rapid7:

* Two lines between functions (but not class methods)
* Two lines between different types of code (like imports and the metadata, see above)
* Four spaces for indenting
* Prefer "foo {}".format('bar') over interpolation with % 
* Keep your callback methods short and readable. If it gets cluttered, break out sub-tasks into well-named functions
* Variable names should be descriptive, readable, and short
* If you really need Python3 features in your module, use #!/usr/bin/env python3 for the shebang
* If you have a lot of legacy code in 2.7 or need a 2.7 library, use #!/usr/bin/env python2.7 (macOS in particular does not ship with a python2 executable by default)
* If possible, have your module compatible with both and use #!/usr/bin/env python 

## Necessary Identifiers

When starting your Python module, you'll need to include two lines to ensure compatibility:

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
```

If you require python2 for your exploit, use:

```python
#!/usr/bin/env python2.7
# -*- coding: utf-8 -*-
```

This allows compatibility for MacOS.

## Import Statements

When importing modules for your exploit, you are required to put them in a try-except block if they are non-standard:

```python
# standard modules
import logging

# extra modules
dependency_missing = False

try:
    import requests
except ImportError:
    dependency_missing = True

from metasploit import module
```

The logging module will be used to issue any output to the metasploit console, and placing everything non-standard 
in the try-except block allows the script to proceed to a point that the console will be able to report any errors 
to the end user.

The last import statement is bringing in necessary functionality so that metasploit can interact with your module.
If you do not have it available from your python environment, your module will still function properly when ran
from the metasploit console.

## Metadata

Every module requires a metadata variable to be defined. This is how metasploit understands what your module is and what parameters it requires:

```python
metadata = {
    'name': 'Example DoS Metadata Module',
    'description': '''
        Shuts off the server room with super crazy DoS powers!
    ''',
    'authors': [
        'Operator'
    ],
    'date': '2047-03-22',
    'license': 'GPL_LICENSE',
    'references': [
        {'type': 'url', 'ref': '<url>'},
        {'type': 'cve', 'ref': '2020-#'},
        {'type': 'edb', 'ref': '#'}
    ],
    'type': 'dos',
    'options': {
        'RHOST': {'type': 'address', 'description': 'Target address', 'required': True, 'default': ''},
        'TIMEOUT': {'type': 'int', 'description': 'Timeout in seconds', 'required': True, 'default': 5}
    }
}
```

Some of these are self explanatory, some are a little more hairy. Whitespace is very important here as well.

### license

This has to come from one of the pre-existing license constants defined in the framework:

```ruby
#
# Global constants
#

# Licenses
MSF_LICENSE      = "Metasploit Framework License (BSD)"
GPL_LICENSE      = "GNU Public License v2.0"
BSD_LICENSE      = "BSD License"
# Location: https://github.com/CoreSecurity/impacket/blob/1dba4c20e0d47ec614521e251d072116f75f3ef8/LICENSE
CORE_LICENSE     = "CORE Security License (Apache 1.1)"
ARTISTIC_LICENSE = "Perl Artistic License"
UNKNOWN_LICENSE  = "Unknown License"
LICENSES         =
  [
    MSF_LICENSE,
    GPL_LICENSE,
    BSD_LICENSE,
    CORE_LICENSE,
    ARTISTIC_LICENSE,
    UNKNOWN_LICENSE
  ]
```

### type

The type parameter controls how the exploit will be understood by metasploit. 
This is an area of development currently so not all modules currently have compatible types.
As of Jun 3 2020, the following types are supported:

```
 capture_server
 dos
 evasion
 multi_scanner
 remote_exploit
 remote_exploit_cmd_stager
 single_host_login_scanner
 single_scanner
```

Each type may or may not require extra parameters in the metadata.

remote_exploit_cmd_stager:
```python
    'targets': [
        {'platform': 'linux', 'arch': 'x64'},
        {'platform': 'linux', 'arch': 'x86'}
    ],
    'payload': {
        'command_stager_flavor': 'wget'
    },
    'privileged': True,
    'wfsdelay': 5,
    'rank': average,
```

remote_exploit:
```python
    'targets': [
        {'platform': 'win', 'arch': 'x64'}
    ],
    'privileged': False,
    'wfsdelay': 5,
    'rank': excellent,
```

evasion:
There are currently no modules of this type.
```python
    'targets': [
        {'platform': 'linux', 'arch': 'x64'},
        {'platform': 'linux', 'arch': 'x86'}
    ],
    'default_options': {
    },
    'advanced_options': {
    },
```

### options/default_options/advanced_options

Options can include anything you may want the user to supply for your module to work.
Depending on the module type, certain options will be automatically included once imported.

Valid option types:

```
 address
 address_local
 address_range
 base
 bool
 enum
 float
 int
 path
 port
 raw
 regexp
 string
```

### targets

Targets assume the role of taking care of two different requirements, platform and architecture.
Some of these are found when listing out platforms for msfvenom payloads, others were collected
by inspecting the /lib/core/msf/module/platform.rb script for aliases on specific platform types.

Also gleaned from inspecting the platform.rb script, it appears that pattern matching is being
done on the platform string being passed, so variations on these values may also be acceptable,
what is listed here is what is explicitly supported.

#### Valid platforms (msfvenom platforms):

```
 linux
 unknown
 win
 windows
 all
 apple_ios
 hardware
 multi
 mainframe
 firefox
 nodejs
 python
 js
 php
 unix
 irix
 hpux
 aix
 freebsd
 netbsd
 bsdi
 openbsd
 bsd
 osx
 solaris
 brocade
 unifi
 juniper
 cisco
 ruby
 r
 java
 android
 netware
```

#### Valid platforms (platform.rb platforms):

```
 8		# Windows 8
 win2008
 win7
 7		# Windows 7 
 vista
 sp0-1 		# vista
 sp0/1 		# vista
 sp0-sp1	# vista
 sp0/sp1	# vista
 Win2003
 2k3		# Win 2003
 2003		# Win 2003
 xp
 winxp
 sp0-1		# XP
 sp0/1		# XP
 sp0-sp1	# XP
 sp0/sp1	# XP
 sp0-2		# XP
 sp0-sp2	# XP
 sp0-e		# XP
 sp0-sp3	# XP
 sp0-4		# windows 2000
 sp0-sp4	# windows 2000
 2000		# windows 2000
 Win2000
 2k		# windows 2000
 WinNT
 NT		# windows NT
 ME		# windows ME
 98		# windows 98
 95		# windows 95
```

#### Valid architectures:

```
 aarch64
 armbe
 armle
 cbea
 cbea64
 cmd
 dalvik
 firefox
 java
 mips
 mips64
 mips64le
 mipsbe
 mipsle
 nodejs
 php
 ppc
 ppc64
 ppc64le
 ppce500v2
 python
 r
 ruby
 sparc
 sparc64
 tty
 x64
 x86
 x86_64
 zarch
```

## Exploit Script

Once you've filled out your metadata section, you'll need your exploit code.
This code is almost identical to what it is being ported from with a few
differences.

### Main Method

You will need a main running method that takes 'args' as a paramter.
'args' will be collected from the end user through metasploit's console utilizing
the options you specified in the options metadata.

```python
def run(args):
    module.LogHandler.setup(msg_prefix='{} - '.format(args['rhost']))
    if dependency_missing:
        logging.error('Module dependency (requests) is missing, cannot continue')
        return

    # Exploit Code
	...
    return
```

Here, we also are setting up module.LogHandler to properly display any output you
want to generate.

### Print/Logging Statements

If your script has any print statements, they will need to be replaced with logging commands
to properly display output through metasploit's console. The following are supported logging
statement types:

```python
 logging.info
 logging.warning
 logging.error
 logging.critical
 logging.debug # Used for debugging, does not display in the console
 logging.exception
```

![Example Logging Outputs](/assets/img/Metasploit-Python/valid-logging-examples.png "Example Logging Outputs")

### Using args

Wherever your exploit script requires user input, you'll need to specify which option that input
is coming from using the 'args' paramter passed to your exploit method.

Examples:

```python
 target = args['rhost'] # string/address/address_local
 port = int(args['rport'] # int/port
 verbose = args['verbose'] == 'true' # bool
```

Ensure that you are properly converting types as you access them via 'args'. Just specifying their
type in the metadata does not mean that the value present in 'args' is that type.

### Calling The Exploit

Now that everything else is done and made compatible with metasploits framework, we need to call our exploit.
At the very end of your script:

```python
 if __name__ == "__main__":
    module.run(metadata, run)
```

This will tell metasploit to run a new module using the metadata created and with the supplied exploit method.

## Importing

To import your python module, simply place it in the most meaningful location in the metasploit directories.
On a typical Linux install this will be

 /usr/share/metasploit-framework/modules/...

When you start up metasploit it will automatically attempt to import the new module. If you can not find it,
check the log to see what might have gone wrong.

## Example Modules

remote_exploit:
* <https://github.com/rapid7/metasploit-framework/blob/master/modules/exploits/windows/smb/ms17_010_eternalblue_win8.py>

remote_exploit_cmd_stager:
* <https://github.com/rapid7/metasploit-framework/blob/master/modules/exploits/linux/smtp/haraka.py>

dos:
* <https://github.com/rapid7/metasploit-framework/blob/master/modules/auxiliary/dos/http/slowloris.py>

single_scanner:
* <https://github.com/rapid7/metasploit-framework/blob/master/modules/auxiliary/gather/get_user_spns.py>
* <https://github.com/rapid7/metasploit-framework/blob/master/modules/auxiliary/gather/office365userenum.py>
* <https://github.com/rapid7/metasploit-framework/blob/master/modules/auxiliary/scanner/smb/impacket/wmiexec.py>
* <https://github.com/rapid7/metasploit-framework/blob/master/modules/auxiliary/scanner/smb/impacket/dcomexec.py>
* <https://github.com/rapid7/metasploit-framework/blob/master/modules/auxiliary/admin/http/grafana_auth_bypass.py>
* <https://github.com/rapid7/metasploit-framework/blob/master/modules/auxiliary/admin/teradata/teradata_odbc_sql.py>
* <https://github.com/rapid7/metasploit-framework/blob/master/modules/auxiliary/scanner/smb/impacket/secretsdump.py>
* <https://github.com/rapid7/metasploit-framework/blob/master/modules/auxiliary/scanner/ssl/bleichenbacher_oracle.py>

multi_scanner:
* <https://github.com/rapid7/metasploit-framework/blob/master/modules/auxiliary/scanner/wproxy/att_open_proxy.py>

single_host_login_scanner:
* <https://github.com/rapid7/metasploit-framework/blob/master/modules/auxiliary/scanner/http/onion_omega2_login.py>
* <https://github.com/rapid7/metasploit-framework/blob/master/modules/auxiliary/scanner/teradata/teradata_odbc_login.py>

evasion:
* None as of yet.

capture_server:
* None as of yet. 

## References

* <https://github.com/rapid7/metasploit-framework>
* <https://github.com/rapid7/metasploit-framework/wiki/Writing-External-Python-Modules>
* <https://blog.rapid7.com/2017/12/28/regifting-python-in-metasploit/>
* <https://blog.rapid7.com/2018/09/05/external-metasploit-modules-the-gift-that-keeps-on-slithering/>

