---
post_title: CLI
nav_title: CLI
menu_order: 050
---

You can use the DC/OS command-line interface (CLI) to manage cluster nodes, install and manage packages, inspect the cluster state, and manage services and tasks.

You can quickly [install](/docs/1.10/cli/install) the CLI from the DC/OS web interface.

To list available commands, either run `dcos` with no parameters or run `dcos help`:

```bash
Command line utility for the Mesosphere Datacenter Operating
System (DC/OS). The Mesosphere DC/OS is a distributed operating
system built around Apache Mesos. This utility provides tools
for easy management of a DC/OS installation.

Available DC/OS commands:

	auth           	Authenticate to DC/OS cluster
	cluster        	Manage connections to DC/OS clusters
	config         	Manage the DC/OS configuration file
	experimental   	Experimental commands. These commands are under development and are subject to change
	help           	Display help information about DC/OS
	job            	Deploy and manage jobs in DC/OS
	marathon       	Deploy and manage applications to DC/OS
	node           	Administer and manage DC/OS cluster nodes
	package        	Install and manage DC/OS software packages
	service        	Manage DC/OS services
	task           	Manage DC/OS tasks

Get detailed command description with `dcos <command> --help`.
```

# Environment variables

These environment variables are supported by the DC/OS CLI and can be set dynamically.

#### DCOS_CONFIG
The path to a DC/OS configuration file. If you put the DC/OS configuration file in `/home/jdoe/config/dcos.toml`, you would set the variable with the command:

```bash
export DCOS_CONFIG=/home/jdoe/config/dcos.toml
```

The `DCOS_CONFIG` variable is supported only before you run the first [`dcos cluster setup`](/docs/1.10/cli/command-reference/dcos-cluster/dcos-cluster-setup) command. `dcos cluster setup` copies the file into `<home-directory>/.dcos/clusters/<cluster_id>/dcos.toml`, after which the variable is ignored. 

#### DCOS_SSL_VERIFY
Indicates whether to verify SSL certificates or set the path to the SSL certificates. You must set this variable manually. Setting this environment variable is equivalent to setting the `dcos config set core.ssl_verify` option in the DC/OS configuration [file](#configuration-files). For example, to indicate that you want to set the path to SSL certificates:

```bash
export DCOS_SSL_VERIFY=false
```

#### DCOS_LOG_LEVEL
Prints log messages to stderr at or above the level indicated. This is equivalent to the `--log-level` command-line option. The severity levels are:

*   **debug** Prints all messages to stderr, including informational, warning, error, and critical.
*   **info** Prints informational, warning, error, and critical messages to stderr.
*   **warning** Prints warning, error, and critical messages to stderr.
*   **error** Prints error and critical messages to stderr.
*   **critical** Prints only critical messages to stderr.

For example, to set the log level to warning:

```bash
export DCOS_LOG_LEVEL=warning
```

#### DCOS_DEBUG
Indicates whether to print additional debug messages to `stdout`. By default this is set to `false`. For example:

```bash
export DCOS_DEBUG=true
```

# <a name="configuration-files"></a>Configuration files

The DC/OS CLI stores its configuration files in a directory called `~/.dcos/clusters/<cluster_id>/dcos.toml` within your HOME directory. 
