# Lappsgrid Installation

This role copies the configuration files needed for a Lappsgrid Galaxy server.

1. **datatypes_conf.xml** - specifies data (file) types recognized by LAPPS Galaxy, i.e. LIF, Gate, UIMA, etc.
2. **lappsgrid.tgz** - the datatype converts invoked automatically by Galaxy to convert between recognized formats as specified in the *datatypes_conf.xml* file.
3. **tool_conf.xml** - the *Tools* menu configuration files.
4. **tools.tgz** - this archive contains the tool XML configuration files and startup scripts. The archive is unpacked into Galaxy's tools folder.
5. **index.html** - a customized welcome page.

