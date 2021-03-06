Graylog Pragmatic Use
=====================

This project contains information to help getting started creating a Graylog server instance. Included are server configurations files for Graylog collector, Logstash pipelines, and custom Extractors for dealing with Java Stack traces and logback default output format.

We started looking at Graylog to save us from the tedious and time consuming process of searching through multiple log files on disparate server systems.  We wanted a centralized location for all of our logs and the ability to extract and query the relevant data.  And we wanted graphical dashboards to display the results of our queries.  But first and foremost, we wanted alerts.  
  
Looking thru logs and running queries over logs and browsing dashboards about logs is still just a little bit like picking up your phone every so often to see if someone is calling you.  We wanted a way to create an alert system.  

When you install the Graylog server and add inputs however, you are still a long way from achieving any of these goals.  Log files come in many different formats and rarely are they populated with all the data needed to centralize them and still be able to determine their context or origin.  We needed to add these data elements and we needed to extract data that was included in each log entry and build a common collection of elements that could be easily queried, regardless of the source of the entries.  

At least three types of inputting messages to the GL server were required immediately.  For apps where we could easily modify source, we wanted to use an appender to push messages to GL.  For legacy apps, we wanted to just monitor the current log files for new entries and push them to GL.  And for both situations, we wanted to upload history log files all in a single shot.  Out of the-box, the third party GELF appender tools, GL Collectors, and Logstash Pipelines necessary for these requirements, all had different data elements and formatting.  Centralizing all of this data would have been useless, at least for building any meaningful queries. 

Descriptions of the tools and the configs used to achieve some standard formatting are described and samples are included in this repository.

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
  
For this reason, inputs should be named after the overall "flavor" of extraction that they do, rather than a particular client application or instance.  In our case, the extraction necessary for an application using Logback and running on WLS was much different than the extraction required for an application using java utility logging running on Glassfish.  GELF appenders require no extraction, so we named our inputs as shown:  
  
*logback_default_extraction_input*  
*glassfish_default_extraction_input*  
*gelfappender_input*

We've included extractor files for both the glassfish and the logback default message formats.  After each input is created, import the appropriate extractor, (cut and paste the contents of the appropriate file into the extractor import page of the graylog web app).
  

Providing Input to the Master GL Server
---------------------------------------

### Tailing Existing Log Files from Remote Applications

##### Graylog Collector
To continuously watch a remote application's current logfile for new entries (think tail) use a Graylog Collector.  Collectors use TCP to send log messages to the GL master server.

-- Install Graylog Collector  
* cd to the applications home folder (for us, */ciminc/app/* and download the latest version of the collector:  
 ``wget https://packages.graylog2.org/releases/graylog-collector/graylog-collector-0.4.2.tgz``  
  
* unpack using:  
 ``tar -zxvf graylog-collector-0.4.2.tgz``  

* rename the unpacked folder from *graylog-collector-0.4.2* to *graylog-collector* and delete the tar

-- Setting up the config  
* copy the sample collector config to the config directory (I used PuTTy):  
 ``pscp graylog_collector.conf root@<remote_logging_machine_ip>:/<graylog_home>/config``  

-- Starting graylog collector   
* modify the config to point to the GL master server and use any new ports if any new inputs were created on the GL master.  
* to start manually, from the graylog home directory (installation directory) run:  
 ``bin/graylog-collector run -f config/collector.conf``  
* add to the startup script so the collector will start after reboots/restarts


##### Logstash Pipeline 
To pipe old (not-current) log files into Graylog, use a logstash pipeline to process files individually and push entries to the master GL server. 

-- Install Logstash  
* cd to the applications home folder (for us, */ciminc/app/* and download the latest version of logstash  
 ``wget https://download.elastic.co/logstash/logstash/logstash-2.1.1.tar.gz``  

* unpack using:  
 ``tar -zxvf logstash-2.1.1.tar.gz``  

* rename the unpacked folder from *logstash-2.1.1* to *logstash* and delete the tar

-- Setting up the config  
* copy the sample logstash config to the installation directory (I used PuTTy):  
 ``pscp logstash_pipeline.conf root@<remote_logging_machine_ip>:/<logstash_home>``  

-- Starting logstash pipeline  
* modify the config to point to the GL master server and use any new ports if any new inputs were created on the GL master.  
* modify the config for each logfile to pipe  
* to start manually, from the logstash home directory (installation directory) run:  
 ``bin/logstash -f logstash_pipeline.conf``  
* add to the startup script so the pipeline will start after reboots/restarts


### Sending Log Entries Directly from a Remote Application 

##### Logging Appender 
To change an application to directly sent log entries to GL, use an appender.  At least two GELF appender plugins can be found on GitHub.  Logstash makes one and there is one authored by MOOCAR.  Each appender creates messages in its default format.  The formats can be altered each to some degree, but the alterations can only re-format to a point.  We chose MOOCAR because it started with a format very close to the structure we wanted and required very few modifications.

To add the MOOCAR GELF appender classes to an application, add the following to the pom.xml:

    <dependency>
        <groupId>me.moocar</groupId>
        <artifactId>logback-gelf</artifactId>
        <version>0.3</version>
    </dependency>


An example appender using the GELF classes from Moocar:

    <appender name="GELF TCP APPENDER" class="me.moocar.logback.net.SocketEncoderAppender">
        <remoteHost>${GRAYLOG_SERVER_IP}</remoteHost>
        <port>${GRAYLOG_SERVER_PORT}</port>
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
                    <value>${APPLICATION_INSTANCE}</value>
                </staticField>
                <staticField class="me.moocar.logbackgelf.Field">
                    <key>app_version</key>
                    <value>1.0.35</value>
                </staticField>
                <staticField class="me.moocar.logbackgelf.Field">
                    <key>environment</key>
                    <value>${ENVIRONMENT}</value>
                </staticField>
            </layout>
        </encoder>
    </appender>


There is a UDP version of the appender from MOOCAR also, (see the attached source).

These appender configuration samples assume the use of four jvm arguments at the container level:

*environment* - the environment in which the instance of the application is running ("DEV", "QA", or "PROD")  
*graylog.server.ip* - the ip address of the Graylog master server  
*graylog.server.port* - the port number of the *gelfappender_input* setup to receive this GELF input  
*application_instance* - the instance of the running application (BIP-1, for instance)  

