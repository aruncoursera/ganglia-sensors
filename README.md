## Description

This repository contains code used to collect monitoring metrics for [Ganglia](https://github.com/ganglia):

> Ganglia is a scalable distributed monitoring system for high-performance computing systems such as clusters and Grids. […] It uses carefully engineered data structures and algorithms to achieve very low per-node overheads and high concurrency. [cf.](http://ganglia.info/)

## Gmetric

All metrics in Ganglia have a **name**, **value**, **type** and optionally **units**:

~~~
» gmetric --name temperature --type int16 --units Celsius --value 45
~~~

By default the host executing is considered to be the source of the collected metric. It is possible to send metrics for other nodes using the `--spoof` option:

~~~
» gmetric -spoof 10.1.1.25:lxdev02.devops.test […]
~~~

String values send to Ganglia are not persistent, and will be lost once gmetad/gmond get restarted:

~~~
» gmetric --type string --name "Service" --value "Ganglia Monitoring Server"
~~~

### Scripts

The following script **gmetric-mpstat** is a more elaborate example. It collects per core statistics using `mpstat`, and sends these to Ganglia. It uses the `--group` option to define a metric collection "cpu_cores".

~~~bash
#!/usr/bin/env bash

args="--type float --group cpu_cores --units percent"

mpstat -P ALL | tail -n +5 | tr -s ' ' | cut -d' ' -f3,4,6,7,12 | while 
  read -r cpu usr sys iowait idle
do 
  `gmetric $args --name "cpu_core_"$cpu"_usr" --value $usr` 
  `gmetric $args --name "cpu_core_"$cpu"_sys" --value $sys` 
  `gmetric $args --name "cpu_core_"$cpu"_iowait" --value $iowait` 
  `gmetric $args --name "cpu_core_"$cpu"_idle" --value $idle`
done
~~~

The option `--name` defines the graph title and file name on the Ganglia server:

    »  ls -1 /var/lib/ganglia/rrds/$cluster/$hostname/cpu_core*
    /var/lib/ganglia/rrds/[…]/cpu_core_0_idle.rrd
    /var/lib/ganglia/rrds/[…]/cpu_core_0_iowait.rrd
    /var/lib/ganglia/rrds/[…]/cpu_core_0_sys.rrd
    /var/lib/ganglia/rrds/[…]/cpu_core_0_usr.rrd

### Execution

It is common to use **cron** to execute a script within an interval to send monitoring information with `gmetric`. Alternatively it is possible to daemonice the collection script itself.

Program | Description
--------|---------------------
[gmetric-infiniband](bin/gmetric-infiniband) | Small daemon sending Infiniband metrics collected from `perfquery` to Ganglia 


## Modules

Ganglia can be extended by Python and C/C++ modules. Modules are executed (in intervals) by gmond in contrast to data collected with gmetric. The Debian package **ganglia-monitor-python** provides the required environment to enable Python modules.

Module | Configuration  | Description
-------|----------------|--------------
[infiniband.py](lib/python_modules/infiniband.py) | [infiniband.pyconf](etc/conf.d/infiniband.pyconf) | Read Infiniband host channel performance metrics from `perfquery`
[ipmi.py](lib/python_modules/infiniband.py) | [ipmi.pyconf](etc/conf.d/ipmi.pyconf) | Read the BMC sensors with `ipmitool`

### Configuration

Path | Description
-----|----------------------
`/usr/lib/ganglia/python_modules` | Default directory for Python modules
`/etc/ganglia/conf.d/*.pyconf` | Module configuration files

The **module** section of the configuration file requires following attributes:

- The **name** value matches the file name of the Python module’s `.py` file.
- The **language** value for a Python module must be "python".
- Each **param** block must include a name and a value, which are passed to the `metric_init()` function of the module. The name of the param block represents the key that corresponds to the parameter value in the Python dictionary object. The value directive specifies the parameter value (always a string!).

The **collection_group** section must contain:

- The **name_match** uses PCRE regex matching to configure metrics.
- The **title** is displayed in the web-interface as name of the graph.

Following examples illustrates the configuration of a module called "infiniband":

~~~
modules {
  module {
    name = "infiniband"
    language = "python"
    param "interval" { value = 20 }
    param "error_metrics" { value = yes }
  }
}

collection_group {

  collect_every = 15
  time_threshold = 45
  metric { 
    name_match = "infiniband_xmtdata_port" 
    title = "Infiniband Send Bytes"
  }
  […]
}
~~~

### Development

Start developing new Python modules from ↴<tt>[lib/python_modules/example.py](lib/python_modules/example.py)</tt>

Check if the metric description is correctly used by gmond:

    » gmond -c /etc/ganglia/gmond.conf -m | grep infini
    infiniband_rcvconstrainterrors_port1     (module python_module)
    infiniband_rcvswrelayerrors_port1        (module python_module)
    infiniband_vl15dropped_port1     (module python_module)
    infiniband_symbolerrors_port1    (module python_module)
    infiniband_rcverrors_port1       (module python_module)
    […]
    » gmond -c /etc/ganglia/gmond.conf -d 2 -f
    […]

Run gmond in foreground and debugging mode the see if everything works as expected.

## License

Copyright 2014-2015 Victor Penso

This is free software: you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version.

This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details.

You should have received a copy of the GNU General Public License along with this program. If not, see http://www.gnu.org/licenses/.

