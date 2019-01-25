**UPDATED CONDOR PROBE HAS BEEN MOVED TO [HEPiX-batchmonitoring/fifemon-condor-probe](https://github.com/HEPiX-batchmonitoring/fifemon-condor-probe)**

Fifemon
=======

Collect HTCondor statistics and report into time-series database. All modules 
support Graphite, and there is some support for InfluxDB.

Additionally report select job and slot Classads into Elasticsearch via Logstash.

Note: this is a fork of the scripts used for monitoring the HTCondor pools
at Fermilab, and while generally intended to be "generic" for any pool still 
may require some tweaking to work well for your pool.

Copyright Fermi National Accelerator Laboratory (FNAL/Fermilab). See LICENSE.txt.

Requirements
------------

* Python 2.6 or greater recommended.
* HTCondor libraries and Python bindings
    * https://research.cs.wisc.edu/htcondor/downloads/
* A running Graphite server (available in EPEL or via PIP) 
    * http://graphite.readthedocs.org/en/latest/

For current job and slot state:

* A running Elasticsearch cluster
    * https://www.elastic.co/products/elasticsearch
* Logstash (tested with v2.0.0+)
    * https://www.elastic.co/downloads/logstash

Installation
------------

Assuming HTCondor and Python virtualenv packages are already installed:

    cd $INSTALLDIR
    git clone https://github.com/fifemon/probes
    cd probes
    virtualenv --system-site-packages venv
    source venv/bin/activate
    pip install supervisor influxdb

Optionally, for crash mails:

    pip install superlance

Configuration
-------------

### Condor metrics probe

Example probe config is in `etc/condor-probe.cfg`:

    [probe]
    interval = 240   # how often to send data in seconds
    retries = 10     # how many times to retry condor queries
    delay = 30       # seconds to wait beteeen retries
    test = false     # if true, data is output to stdout and not sent downstream
    once = false     # run one time and exit, i.e. for running wtih cron (not recommended)
    
    [graphite]
    enable = true                           # enable output to graphite
    host = localhost                        # graphite host
    port = 2004                             # graphite pickle port
    namespace = clusters.mypool             # base namespace for metrics
    meta_namespace = probes.condor-mypool   # namespace for probe metrics
    
    [influxdb]
    enable = false       # enable output to influxdb (not fully supported)
    host = localhost     # influxdb host
    port = 8086          # influxdb api port
    db = test            # influxdb database
    tags = foo:bar       # extra tags to include with all metrics (comma-separated key:value)
    
    [condor]
    pool = localhost            # condor pool (collector) to query
    post_pool_status = true     # collect basic daemon metrics
    post_pool_slots = true      # collect slot metrics
    post_pool_glideins = false  # collect glidein-specific metrics
    post_pool_prio = false      # collect user priorities
    post_pool_jobs = false      # collect job metrics
    use_gsi_auth = false        # set true if collector requires authentication
    X509_USER_CERT = ""         # location of X.509 certificate to authenticate to condor with
    X509_USER_KEY = ""          # private key


### Supervisor

Example supervisor config is in `etc/supervisord.conf`, it can be used as-is for
basic usage. Reuqires some modification to enable crashmails or to report 
job and slot details to elsaticsearch (via logstash). 

### Job and slot state

The scripts that collect raw job and slot records into elasticsearch are much simpler than the metrics 
probe - simply point at your pool with --pool and JSON records are output to stdout. We use logstash 
to pipe the output to Elasticsearch; see `etc/logstash-fifemon.conf`.

Running
-------

Using uspervisor:

    cd $INSTALLDIR/probes
    source venv/bin/activate

If using influxdb:

    export INFLUXDB_USERNAME=<username>
    export INFLUXDB_PASSWORD=<password>

Start supervisor:

    supervisord
