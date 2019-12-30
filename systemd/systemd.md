
# http://man7.org/linux/man-pages/man5/systemd.unit.5.html
*  A unit configuration file encodes information about a service, a socket, a device, a mount point, an automount point, a swap file or partition, a start-up target, a watched file system path, a timer controlled and supervised by systemd(1), a resource management slice or a group of externally created processes.
* Options are configured in [Unit],[Install],[Service] sections
* Various settings are allowed to be specified more than once. They may form a list.
* If systemd encounters an unknown option, it will write a warning log message but continue loading the unit.
* If an option or section name is prefixed with X-, it is ignored completely by systemd.
* Positive values are: 1, yes, true and on
* Negative values are: 0, no, false and off
* Empty lines and lines starting with "#" or ";" are ignored.
* Lines ending in a backslash are concatenated with the following line while reading and the backslash is replaced by a space character.
* Units can be aliased (have an alternative name), by creating a symlink from the new name to the existing name in one of the unit search paths.
* In addition, unit files may specify aliases through the Alias= directive in the [Install] section; those aliases are only effective when the unit is enabled. When the unit is enabled, symlinks will be created for those names, and removed when the unit is disabled.
*  Alias names may be used in commands like enable, disable, start, stop, status, ..., and in unit dependency directives Wants=, Requires=, Before=, After=,..., with the limitation that aliases specified through Alias= are only effective when the unit is enabled. Aliases cannot be used with the preset command.
*  Along with a unit file foo.service, the directory foo.service.wants/ may exist. All unit files symlinked from such a directory are implicitly added as dependencies of type Wants= to the unit.
* Along with a unit file foo.service, a "drop-in" directory foo.service.d/ may exist. All files with the suffix ".conf" from this directory will be parsed after the file itself is parsed.
*  In addition to /etc/systemd/system, the drop-in ".d" directories for system services can be placed in /usr/lib/systemd/system or /run/systemd/system directories.
* Some unit names reflect paths existing in the file system namespace. Example: a device unit dev-sda.device refers to a device with the device node /dev/sda.
* Properly escaped paths can be generated using the systemd-escape(1) command.
