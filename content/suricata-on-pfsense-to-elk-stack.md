Title: Suricata on pfSense to ELK Stack
Date: 2016-03-16 12:32

##Introduction

Suricata is an excellent Open Source IPS/IDS. While there is an official package for pfSense, I found very little documentation on how to properly get it working. Furthermore, there does not appear to be any native functionality to ship the logs it generates to alternative collectors, other than through syslog which I am already exporting to other sources. Below I will detail the steps I performed to get Suricata to ship logs a server running the ELK stack.

Technologies in use:

* [pfSense](https://www.pfsense.org/) - Fully featured open source firewall based on FreeBSD. Will act as the edge device and integrated Suricata host. As of this writing, the latest version is 2.2.6 using FreeBSD 10.1.

* [Suricata](http://suricata-ids.org/) - Open Source IPS/IDS to collect and analyze data to look for possible security risks in the network.

* [ELK Stack](https://www.elastic.co/) - Comprised of Elasticsearch, Logstash, and Kibana. Absolutely fantastic suite of tools for centralizing, analyzing, and visualizing logs. Elasticsearch is used for log storage and search, Logstash for processing the logs into a digestible format for Elasticsearch to consume, and Kibana acts a front end for easy search and visualization. This will run on a separate server from pfSense within the network.

* [Filebeat](https://www.elastic.co/products/beats/filebeat) - Tool for shipping logs to Elasticsearch/Logstash. Will run from pfSense and look for changes to the Suricata logs.

**Warning:**
It is worth mentioning that I am by no means knowledgeable on FreeBSD (I'm really more of a Linux guy), so there are likely things that are not done to best practices. Furthermore, this procedure involves installing unofficial software on pfSense which is absolutely [not supported](https://doc.pfsense.org/index.php/Installing_FreeBSD_Packages). It is also worth pointing out that while pfSense is generally very light on resources, Suricata increases utilization by a good bit.

##pfSense and ELK Setup
I will not be diving into the actual setup of either pfSense or an ELK server and assume both are already operational. pfSense has a good getting started guide [here](https://doc.pfsense.org/index.php/Installing_pfSense), while I found DigitalOcean's ELK guides for [Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-install-elasticsearch-logstash-and-kibana-elk-stack-on-ubuntu-14-04) and [CentOS](https://www.digitalocean.com/community/tutorials/how-to-install-elasticsearch-logstash-and-kibana-elk-stack-on-centos-7) to be very helpful. Elastic's [official documentation](https://www.elastic.co/guide/index.html) is also great.

##Suricata Setup

####Install
In the pfSense web interface, select *System*->*Packages*. Open the *Available Packages* tab, Suricata can be found under the *Security* tab. Use the plus sign on the right side to begin the install. Once complete, Suricata's settings can be accessed from the *Services* menu. First stop is the *Global Settings* tab.

![](/images/2016/03/2016-03-10-18_51_57-pf1-zisk-lan---Services_-Suricata-2-0-9-RELEASE-pkg-v2-1-9-1---Intrusion-Detecti-1.png)

####Configure Rules
Suricata uses downloaded rule sets to determine when to alert. There are two different types of rules that can be set up from this screen:

1. Emerging Threats (now owned by [ProofPoint](https://www.proofpoint.com/us/threat-intelligence-overview)) - This comes in a (free) Open Source version and Pro version which requires a subscription. The free version is more limited than the paid version, but still very robust. To enable, just check the box next to the Open Source version.

2. [Snort](https://www.snort.org/) - Snort is another Open Source IDS product, similar to Suricata, now owned by Cisco. Rules for Snort will work with Suricata. There are several tiers of rule set available ranging from a totally open set, to a set that requires registration and a handful of high paid tiers. The more expensive tiers get new rules sooner. Use the links to sign up for an account at the desired price level and enter the name of the file that can be found after signing in. Suricata also requires your Oinkmaster Code which can be found in your profile after signing up. Check the box to enable Snort rules. If you just want to use the community rules, just check the third box.

Set the *Automatic Update* to field to 12 hours and hit save at the bottom of the screen.

![](/images/2016/03/2016-03-10-19_18_15-pf1-zisk-lan---Suricata_-Global-Settings.png)

Now move on to the update tab to kick off the initial download of the selected rule sets. Just the hit update button and wait  for everything to finish.

####Configure An Interface
Now jump back to the interface tab. Click on the *Add Interface Mapping* button on the right side.
![](/images/2016/03/2016-03-10-19_41_45-pf1-zisk-lan---Services_-Suricata-2-0-9-RELEASE-pkg-v2-1-9-1---Intrusion-Detecti.png)

Pick your WAN interface in the drop down. The majority of the other settings can be left at their defaults. Now check off the [*EVE JSON Log*](https://redmine.openinfosecfoundation.org/projects/suricata/wiki/EveJSONFormat) setting. Make sure the *Eve Output Type* is left one "file" and all of the check boxes in *Eve Logged Info* boxes are checked off. For now, make sure both settings under *Alert Settings* are unchecked. This can be used to automatically kill sessions that match one of the rules we will be applying, but its best to let things run for a few days to ensure everything is operating as expected. Save the settings.

![](/images/2016/03/2016-03-10-20_54_01-pf1-zisk-lan---Suricata_-Interface-WAN---Edit-Settings.png)

Now to apply the rules we downloaded to this interface. This is done in the *Categories* tab. When you open this section you should see all of the rules that were downloaded and extracted earlier. Simply hit the *Select All* then *Save*. 

![](/images/2016/03/2016-03-10-21_22_31-pf1-zisk-lan---Suricata-IDS_-Interface-WAN---Categories.png)

Next, check out the *Rules* section. This is where you can tweak which specific rules from the extract sets actually alert. For now you can just leave everything enabled. I did end up disabling rules under the *decoder-events.rules* and *stream-event.rules*. These seemed to create a lot of noise for things that didn't really appear to be issues related to low-level network behaviors. These events can be filtered from views in Kibana, but I found it easier just to disable them at the source. 

After everything is set, got back to the *Interfaces* page. If there is a red X in the *Enabled* section, click it to start inspection. It should to a green check mark after a few seconds.

![](/images/2016/03/2016-03-10-21_40_29-pf1-zisk-lan---Services_-Suricata-2-0-9-RELEASE-pkg-v2-1-9-1---Intrusion-Detecti.png)

In few minutes, you should start to events that match an applied rule show up in the *Alerts* tab.

![](/images/2016/03/2016-03-10-21_43_39-pf1-zisk-lan---Suricata_-Alerts.png)

At this point, everything should be running that we need to on the Suricata side of things. Now its time to set up Filebeat to forward logs to an ELK server.

##Filebeat Installation/Configuration
There is no official package from pfSense for Filebeat. In order to install it, you will need to manually download to package and set it to run on boot. This must be done from an shell session. If you have not already done so, [enable SSH](https://doc.pfsense.org/index.php/HOWTO_enable_SSH_access) and get logged into the shell (8 from the main menu).

####Find the package
Elastic does not [officially release FreeBSD](https://github.com/elastic/beats/issues/1034) variants of Filebeat. Luckily their build system produces [new releases automatically](https://github.com/elastic/beats/issues/583#issuecomment-166628792). You just have to find the latest release.

First, find the hash of the latest push to the master branch. I did this from another machine:

```bash
curl -XGET https://api.github.com/repos/elastic/beats/git/refs/heads/master
```
You will get a JSON array back. Look for the sha field under *object*.

```json
{
  "ref": "refs/heads/master",
  "url": "https://api.github.com/repos/elastic/beats/git/refs/heads/master",
  "object": {
    "sha": "e84484e5b5dd09caf159f83c2c8359c5e988b3b4",
    "type": "commit",
    "url": "https://api.github.com/repos/elastic/beats/git/commits/e84484e5b5dd09caf159f83c2c8359c5e988b3b4"
  }
}
```
Now find the corresponding build on [this](https://beats-nightlies.s3.amazonaws.com/index.html?prefix=jenkins/filebeat/) page. You want the *filebeat-freebsd-amd64* package. Copy the URL then fetch it right on your pfSense box.

```bash
cd ~
fetch https://beats-nightlies.s3.amazonaws.com/jenkins/filebeat/321-e84484e5b5dd09caf159f83c2c8359c5e988b3b4/filebeat-freebsd-amd64
```
Move filebeats to where we'll run it out of and ensure it is executable:

```bash
mkdir /etc/filebeat
mv ~/filebeat-freebsd-amd64 /etc/filebeat/
chmod +x /etc/filebeat/filebeat
```

####Configure Filebeat
By default, Filebeat will look in its current directory for a configuration file called [filebeat.yml](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-configuration.html). An example configuration can be found [here](https://raw.githubusercontent.com/elastic/filebeat/master/etc/filebeat.yml). Ultimately we just need to ship any EVE files so the configuration will be fairly minimal, but first we need to find where those files are located. Generally, they seem to be located in `/var/log/suricata/<interface_name>/eve.json` but you can also locate them with:

```bash
find / -name "eve.json"
```

The contents of `/etc/filebeat/filebeat.yml` should be:

```yaml
filebeat:
  prospectors:
    -
      paths:
        - /var/log/suricata/*/eve.json 
      input_type: log
      document_type: SuricataIDPS 
output:
  logstash:
    hosts: ["192.0.2.15:5044"]

```

Replace hosts with the IP address or host name of your Logstash server. Remember that white space is significant in [YAML](https://www.elastic.co/guide/en/beats/filebeat/current/_yaml_tips_and_gotchas.html) and tabs are not allowed.  This is about as simple a configuration that you can get away with but there is [much more](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-configuration-details.html) that can be configured from here. Make note of the *document_type* field, which will be used in some Logstash filters later. Test for syntax errors with:

```bash
/etc/filebeat/filebeat -configtest
```
It will return nothing if the config file is properly formatted. 

####Set Filebeat To Start Automatically
The easiest way to set Filebeat to start at boot is to add a shellcmd to pfSense's config. Per the official [documentation](https://doc.pfsense.org/index.php/Executing_commands_at_boot_time) there are two ways to accomplish this: manually editing the config or via an installable package. I had a lot of issues when doing it manually, so the package route seems like the better way to go.

Navigate to *System*, *Packages* in the pfSense menu and locate the *Shellcmd* package. Click on the Plus sign on the right to kick off the install. Once complete, the package can be configured via the *Services* menu. Click the plus sign to add a new command. In the *Command* box enter `/etc/filebeat/filebeat`. Leave the type box as `shellcmd`. Save the configuration. You will need to restart for Filebeat to begin running in the background.

##Logstash Configuration
Now we're going to jump over to the ELK server. Before we get into the actual configuration, let's grab a geolocation dataset that we be useful for mapping IP addresses to locations on a map. I generally just store it in the Logstash directory:

```bash
cd /etc/logstash
wget http://geolite.maxmind.com/download/geoip/database/GeoLiteCity.dat.gz
gzip -d GeoLiteCity.dat.gz
```
Logstash configurations consists of three parts:

1. An input, where to accept the stream for the log. In our case it will be a port to receive the Filebeat traffic.
2. A filter, used to transform the data so that it is ready to be placed in the output source. Since Suricata is already outputting JSON the only thing that is really being done is a look up of the IP address against the geolocation database we downloaded earlier.
3. An output, where to store the data. In this case, the Elasticsearch instance.

All of these could be defined in a single file but it is much more manageable to split them into separate files. Create and populate the following under /etc/logstash/conf.d:

Input file:
`01-beatin.conf`
```
input {
  beats {
    port => 5044
    codec => json
  }
}
```

Filter (make sure the path to the geo data matches):
`10-suricata.conf`
```
filter {
  if [type] == "SuricataIDPS" {
    date {
      match => [ "timestamp", "ISO8601" ]
    }
    ruby {
      code => "if event['event_type'] == 'fileinfo'; event['fileinfo']['type']=event['fileinfo']['magic'].to_s.split(',')[0]; end;"
    }
  }

  if [src_ip]  {
    geoip {
      source => "src_ip"
      target => "geoip"
      #database => "/etc/logstash/GeoLiteCity.dat"
      add_field => [ "[geoip][coordinates]", "%{[geoip][longitude]}" ]
      add_field => [ "[geoip][coordinates]", "%{[geoip][latitude]}"  ]
    }
    mutate {
      convert => [ "[geoip][coordinates]", "float" ]
    }
    if ![geoip.ip] {
      if [dest_ip]  {
        geoip {
          source => "dest_ip"
          target => "geoip"
          #database => "/etc/logstash/GeoLiteCity.dat"
          add_field => [ "[geoip][coordinates]", "%{[geoip][longitude]}" ]
          add_field => [ "[geoip][coordinates]", "%{[geoip][latitude]}"  ]
        }
        mutate {
          convert => [ "[geoip][coordinates]", "float" ]
        }
      }
    }
  }
}
```
Output:
`30-elasticout.conf`
```
output {
  elasticsearch {
    host => localhost
  }
}

```

Now check the configuration, on Ubuntu 14.04 it is done with the following command:
```bash
 service logstash configtest 
```
The command should return `Configuration OK` if everything is good. Restart Logstash with `service logstash restart` to make sure it picked up the configuration changes.

You should now be able to see the events from Suricata in Kibana:

![](/images/2016/03/2016-03-14-14_30_29-.png)

With the default configuration there is quite a bit logged since DNS and TLS connects are both included, but these can be filtered out by setting the `event_type` to *alert*.

##Notes
* I did not set up any security for Filebeats during this tutorial, but it is recommended that you set up encryption, which is detailed in the DigitalOcean tutorial above. Elastic also makes a product called [Shield](https://www.elastic.co/products/shield), which I have not tried yet but is used for securing various parts of the ELK stack.

* I don't know if its necessary but I did set up Logrotate to cycle the EVE files when they get too large. This can be [installed](https://doc.pfsense.org/index.php/Installing_FreeBSD_Packages) using the `pkg `command on pfSense. I used the following config and setup a daily cronjob:
```
/var/log/suricata/*/eve.json {
        size 500M
        rotate 0
        missingok
        postrotate
                killall -HUP suricata
        endscript
}
```
* I have not set up any sort of maintenance on the Elasticsearch instance so records will keep getting added until it fills the disk. I am going to look at [Curator](https://www.elastic.co/guide/en/elasticsearch/client/curator/current/index.html) for this. 

