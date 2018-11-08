

# 1) Bind Elasticsearch to IP address #

Open *elasticsearch.yml* configuration file  
`sudoedit /etc/elasticsearch/elasticsearch.yml`

Find lines with: `network.host:` and `http.port:`. Remove commentation.  
Add IP address and port number to the lines.

Restart elasticsearch:  
`sudo service elasticsearch restart`

Testing elasticsearch with *curl*:  
```
sudo apt-get install -y curl
curl -X GET "your_ip:9200"
```
Output should look something similar:
```
{
  "name" : "2uR_tHN",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "-PI4WBvhRTCHFy4f1WeE8Q",
  "version" : {
    "number" : "6.4.1",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "e36acdb",
    "build_date" : "2018-09-13T22:18:07.696808Z",
    "build_snapshot" : false,
    "lucene_version" : "7.4.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}

```

# 2) Configure server to receive data #  

---------------------------
**Huom!**
At a later stage of the stack configuration, we found out that */var/log/syslog* had filled our entire system and took some 450Gb space. By searching information about the problem, I came upon a post that suggested this is caused by `stdout` output somewhere in the configuration files.

~~The current fix we have, is to disable the following modules in the next step, as you enable *imudp* and *imtcp* modules: *imuxsock* and *imklog*~~ Scratch that. I'm still unsure what caused the looping last week, but I left those modules up and nothing imploded.

--------------------------

We load *imudp* and *imtcp* modules in rsyslog.conf -file, which is located under */etc*. These modules are `Input module UDP` and `Input module TCP`. These modules are used for syslog reception.

`sudoedit /etc/rsyslog.conf`  
Delete the commentation about "imtcp" and "imudp" to bring the modules online. 
`sudo service rsyslog restart`


# 3) Configuring Rsyslog to send data #

`sudoedit /etc/rsyslog.d/50-default.conf`  
add `*.*  @@10.x.x.x:514`  
`sudo service rsyslog restart`


# 4) Format logdata to JSON #

`sudoedit /etc/rsyslog.d/01-json-template.conf`
Copy the following into the file:  
```
template(name="json_lines" type="list" option.json="on") {
      constant(value="{")
      constant(value="\"@timestamp\":\"")
      property(name="timereported" dateFormat="rfc3339")
      constant(value="\",\"@version\":\"1")
      constant(value="\",\"message\":\"")
      property(name="msg" format="json")
      constant(value="\",\"sysloghost\":\"")
      property(name="hostname")
      constant(value="\",\"severity\":\"")
      property(name="syslogseverity-text")
      constant(value="\",\"facility\":\"")
      property(name="syslogfacility-text")
      constant(value="\",\"programname\":\"")
      property(name="programname")
      constant(value="\",\"syslog-tag\":\"")
      property(name="syslogtag")
      constant(value="\"}\n")
}
```


# 5) Configure server to send data to logstash #  

`sudoedit /etc/rsyslog.d/60-output.conf`  
```
*.*                         @private_ip_logstash:10514;json-template
```


# 6) Configure Logstash to receive JSON messages. #

Open the *logstash.conf* file under */etc/logstash/*
sudoedit /etc/logstash/conf.d/logstash.conf

add the following to the *logstash.conf* file:
```
input {
  udp {
    host => "logstash_private_ip"   
    port => 10514
    codec => "json"
    type => "rsyslog"
  }
}
filter { }
output {
  if [type] == "rsyslog" {
    elasticsearch {
      hosts => [ "elasticsearch_private_ip:9200" ]
    }
  }
}
```

`sudo service logstash restart`
`sudo systemctl start logstash.service`

Ensure that Logstash is running in port 10514:  
`netstat -na | grep 10514`  

or:  
`sudo netstat -ntlpu`

Test Logstash configuration with command:  
`sudo -u logstash /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t`

https://www.elastic.co/guide/en/logstash/current/config-setting-files.html


# 7) Configure Kibana #

Open Kibana configuration file under */etc/kibana/*  
`sudoedit /etc/kibana/kibana.yml`

If you want to enable remote connection to Kibana, uncomment the line with:  
`#server.host: "localhost"`  
and replace localhost with your IP. Outcome should look like `server.host: "http://172.28.171.194.9200"`.

Next, you can rename your Kibana server by uncommenting `server.name: " "` line, and adding desired server name.

Finally, uncomment line with `#elasticsearch.url: " "` and add your IP address to match the IP address given in `/etc/elasticseatch/elasticsearch.yml`

One more thing. Restart Kibana to apply configuration changes:  
`sudo service kibana restart`


# 8) Checking Elasticsearch input #

Browse your Kibana. If it doesn't complain about not connecting to elasticsearch, these configurations are fine to some degree. We are still unable to pass data to Kibana, so something is still missing or misconfigured. If your Kibana does connect to Elasticsearch, but there is no data, then the fault should lie in Logstash.  
Browsing `http://elasticsearch_IP:9200/_cat/indices?v` should give you a hint. Whetever is listed there, should work properly.

Kokeiluun:  
https://www.rsyslog.com/coupling-with-logstash-via-redis/#more-2356 

# Troubleshooting #  

**Jussi got his Rsyslog-to-elasticsearch-to-kibana build working!**
 

**This build is now obsolete. We have no reason to use a build with logstash, because it consumes much more resources than a stack with just Rsyslog, Elasticsearch and Kibana. I will however try to complete this build.**


I'm fairly sure that the problem lies in either rsyslog or logstash configuration. I tried to make few different kind of *logstash.conf* files.  

Try 1:  
```
# Pull in syslog & rsyslog data
input {
  file {
    path => [
      "/var/log/syslog",
      "/var/log/auth.log"
    ]
    type => [
      "syslog",
      "rsyslog"
    ]
  }
}

# This input block will listen on port 10514 for logs to come in.

input {
  udp {
    host  => "172.28.172.231"
    port  => 10514
    codec => "json"
    type  => [
      "syslog",
      "rsyslog"
    ]
  }
  tcp {
    host  => "172.28.172.231"
    port  => 10514
    codec => "json"
    type  => [
      "syslog",
      "rsyslog"
    ]
  }
}

# Lets leave this empty for now. The filter plugins are used to enrich and transform data.
filter { }

# This output block will send all events of type "syslog" or "rsyslog" to Elasticsearch at the configured host and port
output {
  if [type] =="rsyslog" {
    elasticsearch {
      hosts => [ "172.28.172.231:9200" ]
    }
  }
  if [type] == "syslog" {
    elasticsearch {
      hosts => [ "172.28.172.231:9200" ]
    }
  }
}
```
source: https://opensource.com/article/17/10/logstash-fundamentals

Try 2 (this should be little more simple approach):  
```
input {
  udp {
    host  => "172.28.172.231"
    port  => 10514
    codec => "json"
    type  => [
      "syslog",
      "rsyslog"
    ]
  }
  tcp {
    host  => "172.28.172.231"
    port  => 10514
    codec => "json"
    type  => [
      "syslog",
      "rsyslog"
    ]
  }
}
output {
  elasticsearch { host => "172.28.172.231:9200" }
  stdout {
    codec    => "json"
    protocol => "http"
  }
}
```
I must continue this another day. This file is still missing input (and filter) block. source: https://discuss.elastic.co/t/error-on-logstash-cant-connect-to-elasticsearch/28674/3  
If the file doesn't work, I will follow my next lead: https://deviantony.wordpress.com/2014/09/23/how-to-setup-an-elasticsearch-cluster-with-logstash-on-ubuntu-12-04/ 

## Where I left off & continuing troubleshooting ##

![kuva3](https://i.imgur.com/SX0bl38.png)

Overly simplified *logstash.conf* file:  
```
input { stdin { } }

filter { }

output {
  elasticsearch { hosts => ["172.28.171.230:9200"] }
  stdout { codec => json }
}

```
According to some sources (I actually lost the source) I should split the configuration file into 3 parts: input, output and filter. I gave this an attempt, but to no avail.

*02-syslog-input.conf*  
```
input {
  file {
    path => ["/var/log/syslog"]
    type => "syslog"
  }
}

```

*10-syslog-filter.conf*  
```
filter {
  if [type] == "syslog" {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:$
      add_field => [ "received_at", "%{@timestamp}" ]
      add_field => [ "received_from", "%{host}" ]
    }
    syslog_pri { }
    date {
      match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
    }
  }
}

```

*30-elasticsearch-output.conf*  
```
output {
  elasticsearch {
      hosts => ["https://172.28.171.230:9200"]
      index => "syslog-%{+YYYY.MM.dd}"
      document_type => "system_logs"
  }
  stdout { codec => rubydebug }
}

```

Previous logstash.conf files didn't seem to work. This leads me to believe one of the following things:  
1) Logstash configuration file is working  
2) If 1) is true, then the fault is most likely in files under rsyslog.d


I tested Logstash configuration with command:  
`sudo -u logstash /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t`  
and the test came out saying "*Configuration OK*". However, there is no data in the directory where logstash.yml directs us (/var/log/logstash/). Seems like something is still off.

Trying different configuration again and running the tests.  
```
input {
  udp {
    host  => "172.28.171.230"
    port  => 10514
    codec => "json"
    type  => [
      "syslog",
      "rsyslog"
    ]
  }
  tcp {
    host  => "172.28.171.230"
    port  => 10514
    codec => "json"
    type  => [
      "syslog",
      "rsyslog"
    ]
  }
}

# Lets leave this empty for now. The filter plugins are used to enrich and transform data.
filter { }

# This output block will send all events of type "syslog" or "rsyslog" to Elasticsearch at the configured host and port
output {
  elasticsearch { host => "172.28.171.230:9200" }
  stdout {
    codec    => "json"
    protocol => "http"
  }
}

```

The part with  
```
type  => [
      "syslog",
      "rsyslog"
    ]
```
doesn't pass the configuration check, so at the absolute least we can say the test actually does something. Fixing and testing again. 

Also had to remove the line with `protocol => "http"` and the configuration test passed.

This should be a working file:
```
# This input block will listen on port 10514 for logs to come in.

input {
  udp {
    host  => "172.28.171.230"
    port  => 10514
    codec => "json"
    type  => "rsyslog"
  }
  tcp {
    host  => "172.28.171.230"
    port  => 10514
    codec => "json"
    type  => "rsyslog"
  }
}

# Lets leave this empty for now. The filter plugins are used to enrich and transform data.
filter { }

# This output block will send all events of type "syslog" or "rsyslog" to Elasticsearch at the configured host and port
output {
  elasticsearch { hosts => "http://172.28.171.230:9200" }
  stdout {
    codec    => "json"
  }
}

```

`sudo netstat -ntlpu`
Logstash doesn't seem to be active in port 10514

Jussi pointed out, that the process should be started from */usr/share/logstash/bin*. Lets start the service from this location, using the configuration files under /etc/logstash/conf.d.  
`./logstash -f /etc/logstash/conf.d/`

Ensure that Logstash is running in port 10514:  
`netstat -na | grep 10514`

![kuva1](https://i.imgur.com/Nkc7kQH.png)

Check Kibana!

![kuva2](https://i.imgur.com/cbOBwm8.png)

Now that we are passing data to Kibana, we might as well clean that logstash.conf file. Removing the TCP part from input block also cleaned the messages a lot in Kibana.

```
input {
  udp {
    host  => "172.28.171.230"
    port  => 10514
    codec => "json"
    type  => "rsyslog"
  }
}

filter { }

output {
  elasticsearch { hosts => "http://172.28.171.230:9200" }
  stdout {
    codec    => "json"
  }
}
```
