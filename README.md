Graylog configurations
======================

This project contains information to help getting started creating a Graylog server instance. Included are server configurations files for Graylog collector, Logstash pipelines, and custom Extractors for dealing with Java Stack traces and logback default output format.

Graylog Master Server Setup
---------------------------

### Installation and Setup

##### We used the Graylog image for Open Stack

### Adding Input and Extractors

##### Inputs
In order for the GL Master Server to receive input from remote applications, you will at a minimum need a TCP GELF input.  Collectors only push log entries using TCP and we used the Moocar Logback appender to transmit TCP as well.  We also wanted to be able to load some historical log files (in case we wanted to do some trend analysis).  To do this you will need a UDP GELF input to receive the Logstash input.  

Make sure to note the ports used for each of the inputs and use them respectively when setting up the collectors, pipelines, or appenders.

Extractors are necessary on the GELF inputs, but the same set of extractors can be used on both UDP and TCP inputs listening to applications using similar logging systems/formats.  


##### Extractors
Extractors are used to format or perform conversion on incoming messages received by Graylog inputs.  The goal of extraction should be to extract and normalize enough data from the raw data contained in each message to  enable querying the data, providing useable results.  Each input containing a different or new message format will require a new set of extractors, tailored to parse the data appropriately.  
  
For this reason, inputs should be named after the overall "flavor" of extraction that they do, rather than a particular client application or instance.  In our case, the extraction necessary for an application using Logback and running on WLS was much different than the extraction required for an application using Logback but running on Glassfish.  We named our inputs *logback_default_extraction_input* and *logback_glassfish_extraction_input* because the WLS and logback config seemed to be a more normalized format.


Providing Input to the Master GL Server
---------------------------------------

### Tailing Existing Log Files from Remote Applications

##### Graylog Collector
To continuously watch a remote application's current logfile for new entries (think tail) use a Graylog Collector.  Collectors use TCP to send log messages to the GL master server.

-- Install Graylog Collector  
* download the collector:  
 ``wget https://packages.graylog2.org/releases/graylog-collector/graylog-collector-0.4.2.tgz``  

* move the tar to /<graylog_intallation_path>/graylog-collector and unpack using:  
 ``tar -zxvf graylog-collector-0.4.2.tgz``  

-- Setting up the config  
* copy the sample collector config to the config directory (I used PuTTy):  
 ``pscp graylog_collector.conf root@<remote_logging_machine_ip>:/<graylog_home>/graylog-collector/config``  

-- Starting graylog collector from command line  
* modify the config to point to the GL master server and use any new ports if any new inputs were created on the GL master.  
* from the graylog home directory (installation directory) run:  
 ``bin/graylog-collector run -f graylog_collector.conf``  


##### Logstash Pipeline 
To pipe old (not-current) log files into Graylog, use a logstash pipeline to process files individually and push entries to the master GL server. 

-- Install Logstash  
* download logstash  
 ``wget https://download.elastic.co/logstash/logstash/logstash-2.1.1.tar.gz``  

* move the tar to /<logstash_intallation_path>/logstash-2.1.1 and unpack using:  
 ``tar -zxvf logstash-2.1.1.tar.gz``  

-- Setting up the config  
* copy the sample logstash config to the installation directory (I used PuTTy):  
 ``pscp logstash_pipeline.conf root@<remote_logging_machine_ip>:/<logstash_home>/logstash-2.1.1``  

-- Starting logstash pipeline from command line  
* modify the config to point to the GL master server and use any new ports if any new inputs were created on the GL master.  
* modify the config for each logfile to pipe  
* from the logstash home directory (installation directory) run:  
 ``bin/logstash -f logstash_pipeline.conf``  


### Sending Log Entries Directly from a Remote Application 

##### Logging Appender 
To change an application to directly sent log entries to GL, use an appender.  

An example appender using the GELF classes from Moocar:

    <appender name="GELF TCP APPENDER" class="me.moocar.logback.net.SocketEncoderAppender">
        <remoteHost>${GRAYLOG_SERVER_IP}</remoteHost>
        <port>${GRAYLOG_LOGBACK_TCP_GELF_INPUT_PORT}</port>
        <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
            <layout class="me.moocar.logbackgelf.GelfLayout">
                <shortMessageLayout class="ch.qos.logback.classic.PatternLayout">
                    <pattern>%m%n%ex{short}</pattern>
                </shortMessageLayout>
                <fullMessageLayout class="ch.qos.logback.classic.PatternLayout">
                    <pattern>%ex{full}</pattern> <!-- %relative%thread%mdc%level%logger%msg -->
                </fullMessageLayout>
                <useLoggerName>true</useLoggerName>
                <useThreadName>true</useThreadName>
                <useMarker>true</useMarker>
                <staticField class="me.moocar.logbackgelf.Field">
                    <key>app_name</key>
                    <value>UASNY</value>
                </staticField>
                <staticField class="me.moocar.logbackgelf.Field">
                    <key>app_instance</key>
                    <value>KEV-1</value>
                </staticField>
                <staticField class="me.moocar.logbackgelf.Field">
                    <key>app_version</key>
                    <value>1.0.35</value>
                </staticField>
            </layout>
        </encoder>
    </appender>




