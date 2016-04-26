# ESxSNMP: The ESnet Extensible SNMP System #

# Overview #

ESxSNMP is a system for collecting, storing, visualizing and analyzing large sets of SNMP data. It was driven by the needs of the ESnet engineering team but is likely useful to a much wider audience. ESxSNMP makes use of the Thrift lightweight RPC package for communication between system elements.

ESxSNMP uses a hybrid model for storing data. Time series data such as interface counters is stored using [TSDB](http://code.google.com/p/tsdb/). TSDB is a library for storing time series data with no loss of information. TSDB optimizes the store of it's data so that data which share similar timestamps is stored nearby on the disk allowing very fast access to specific time ranges. Data such as interface description and interface type are stored in an SQL database. Storing this data in an SQL database allows us to use the full expressiveness of SQL to query this data. Since this data changes relatively infrequently the demands placed on the SQL server are fairly modest.

## Architecture ##

## Desgin Goals ##

ESxSNMP was designed to meet the needs of the Network Engineering group at [ESnet](http://www.es.net/).  The key design goals were:

  * data collection should be very reliable
  * data visualization should be very reliable but not at the expense of data collection
  * raw data should never be discarded
  * new interfaces should be detected automatically
  * automate as much as possible
  * provide a clean interface for programatic control


# Installation and Configuration #

## Prerequisites ##

  * [Python](http://www.python.org/), version 2.4 or greater
  * [yapsnmp](http://yapsnmp.sourceforge.net/)
  * [PostgreSQL](http://www.postgresql.org/), verson 8.0 or greater
  * [psycopg2](http://www.initd.org/pub/software/psycopg/PSYCOPG-2-0/)
  * [SQLAlchemy](http://www.sqlalchemy.org/)
  * [Thrift](http://developer.facebook.org/thrift/)
  * [TSDB](http://code.google.com/p/tsdb/)


## Installation ##

### Using buildout ###

### Traditional Installation ###

## Performance Tuning ##

## Configuration ##

Data collection is controlled by the configuration stored in the database.  A `device` is any device from which data needs to be extracted.  Each device can be configured to have one or more `OIDSet` s collected.  An OIDSet is a list of (generally) related `OID` s to collect together.

# `espolld` #

`espolld` is the process which polls the devices, correlates the data (if necessary) and stores the data.

## Operation ##

When `espolld` is started it will query the database for a list of currently active devices and which OIDSets should be polled for each device.  A separate process will be spawned for each OIDSet for each device.  `espolld` monitors the child processes and restarts them as necessary.

Each child process has two threads of execution: a polling thread and a storage/aggregator thread.  This separation is made to ensure that polling is not held up if writing the data to disk takes a prolonged amount of time. Every time a datapoint is collected it is written to disk and it's lowest level aggregate is updated.  This can be fairly demanding in terms of I/O load.

`espolld` checks the database for new devices, addition or removal of OIDSets, changes in device status and SNMP community every 5 minutes.  When it detects changes it stops, starts or restarts the relevant collector processes as necessary.  Note that if the OIDs in an OIDSet change the `espolld` process must be restarted.

If a child process dies repeatedly `espolld` will take note of this and report it.  `espolld` will only try to respawn a child 3 times in the course of 30 seconds before it will put the child process into the penalty box.  If `espolld` is sent the SIGUSR1 signal it will try to start all of the children in the penalty box.  If they misbehave again they will be put back into the penalty box.

At present `espolld` does not use `esdbd` for database interactions but instead contacts the SQL and TSDB databases directly.  This issue will be addressed in future versions of ESxSNMP.

## Configuration ##

# `esdbd` #

`esdbd` provides a consistent interface to the ESxSNMP databases.  It provides a front end service to query both the SQL and TSDB datastores.

# `esxsnmp` #

`esxsnmp` is the command line interface to ESxSNMP.  It is used to administer the system.