# MongoDB-Guardium Logstash Filter plugin

This is a [Logstash](https://github.com/elastic/logstash) filter plugin for [Guardium universal connector][KC-UC-main], a feature in IBM Security Guardium. 

This filter plugin parses Logstash events from a MongoDB audit log and transforms them into a Guardium Record object, which Guardium universal connector then inserts into Guardium. Potentially, this allows Guardium to monitor activity from any data source or service. 

Use this project as an example and starting point when you develop other plugins for Guardium universal connector.

This project is fully free and fully open-source. The license is Apache 2.0, meaning you are free to use it however you want, like use it as a starting point to develop another filter plugin for Guardium universal connector, to support your data source.


## How to use
To use this filter plugin, here's an overview of the process:
1. Set-up your dev environment
2. Build this filter into a GEM file
3. Install GEM on a local Logstash distributation
4. Run Logstash with a test configuration to see how this filter behaves.

Follow the steps below: 

### Set-up dev environment
Before you can build & create an updated GEM of this filter plugin, set up your environment as follows: 
1. Clone Logstash codebase & build its libraries as as specified in [How to write a Java filter plugin][logstash-java-plugin-dev]. Use branch 7.x (this filter was developed alongside 7.5 branch).  
2. Create _gradle.properties_ and add LOGSTASH_CORE_PATH variable with the path to the logstash-core folder you created in the previous step. For example: 

    ```LOGSTASH_CORE_PATH=/Users/taldan/logstash76/logstash-core```

3. Clone the [guardium-universalconnector-commons][github-uc-commons] project and build a JAR from it according to instructions specified there. The project contains Guardium Record structure you need to adjust, so Guardium universal connector can eventually feed your filter's output into Guardium. 
4. Edit _gradle.properties_ and add a GUARDIUM_UNIVERSALCONNECTOR_COMMONS_PATH variable with the path to the built JAR. For example:

    ```GUARDIUM_UNIVERSALCONNECTOR_COMMONS_PATH=../guardium-universalconnector-commons/build/libs```

If you'd like to start with the most simple filter plugin, we recommend to follow all the steps in [How to write a Java filter plugin][logstash-java-plugin-dev] tutorial.

### Build plugin GEM
To build this filter project into a GEM that can be installed onto Logstash, run 

    $ ./gradlew.unix gem --info

Sometimes, especially after major changes, clean the artifacts before you run the build gem task:

    $ ./gradlew.unix clean

### Install
To install this plugin on your local developer machine with Logstash installed, run:
    
    $ ~/Downloads/logstash-7.5.2/bin/logstash-plugin install ./logstash-filter-mongodb_guardium_filter-?.?.?.gem

**Notes:** 
* Replace "?" with this plugin version
* logstash-plugin may not handle relative paths well, so try to install the gem from a simple path, as in the example above. 

### Run on local Logstash
To test your filter using your local Logstash installation, run 

    $ ~/Downloads/logstash-7.5.2/bin/logstash -f filter-test-generator.conf
    
This configuration file generates an Event and send it thru the installed filter plugin. 

### Install the plugin on a Guardium collector
The last step of development is to test that the plugin works on a staging Guardium collector. [Installing and testing the plugin on a staging Guarduim collector][KC-UC-staging] topic in IBM Knowledge Center details the steps.  

**Note** that if you intend to install this plugin, you will need to rename its filter and package names before you install it on a Guardium collector. 

## MongoDB-Guardium filter capabilities
### Supported audit messages & commands: 
* authCheck: 
    * find, insert, delete, update, create, drop, ... 
    * aggregate with $lookup(s) or $graphLookup(s)
    * applyOps: An internal command but can be triggered manually to create/drop collection. It's object is written as "\[json-object\]" in Guardium; details are included inGuardium Full SQL field, if available. 
* authenticate (with error only) 

### Supported errors:  
* Authentication error (18) – A failed login error.
* Authorization error (13) - To see the "Unauthorized ..." description in Guardium, you'll need to extend the report and add the "Exception description" field. 

__Notes:__ 
* For these events to be handled propertly, some conditions must occur: 
    * MongoDB access control must be set, as messages without users are removed. 
    * authCheck and authenticate events should not be filtered-out from the MongoDB audit log messages.
* Other MongoDB events/messages are removed from pipeline, as their data is already parsed in authcheck message.
* Non-MongoDB events are only skipped and not removed from pipline, as they may be used by other filters along the Connector configuration pipeline. In that case, the event is tagged with "_mongoguardium_skip_not_mongodb". 
* If a JSON parsing error occurs, the event is tagged with "_mongoguardium_json_parse_error" and not removed (this may happen, for example, if syslog message is too long and got truncated). These tags can be useful for debugging purposes. 
* To send errors, MongoDB Access control must be configured so these events will be logged. For example, edit _/etc/mongod.conf_ to contain:
```    
    security:  
        authorization: enabled
```
* The filter supports events sent thru Syslog or Filebeat, and counts on a "mongod:" prefix in the event message, before the JSON part of the audit is parsed. This could be improved, but on the otherhand it saves parsing time. 
* Field _server_hostname_ (required) - Server hostname is expected (extracted from syslog message, 2nd field, or populated by Filebeat configuration).
* Field _server_ip_ - States the IP of the MongoDB server; if it is available for the filter plugin, the filter will use it instead localhost IPs reported by MongoDB. 
* Client "Source program" is currently not available in messages sent by MongoDB, as this datum is sent only on the first audit log message upon DB connection, and this filter plugin doesn't aggregate data from different messages, but rather only parses the current message at hand.  
* If events with "(NONE)" local/remote IP are not filtered (unlikely, as messages without users are filtered-out), the filter plugin will convert the IP to "0.0.0.0", as a valid format for IP is needed.
* Events that reach the filter are not removed, but tagged if not parsed.
* The filter redacts (masks) the audit messages of type MongoDB authCheck: Currently, most field values are replaced with "?" in a naïve process, where most command arguments are redacted, apart from the the _command_, _$db_, and _$lookup_/_$graphLookup_ required arguments (_from_, _localField_, _foreignField_, _as_, _connectFromField_, and _connectToField_).

## Example 
A Syslog message from the input stage reaches this filter as a message string within a Logstash Event: 

    2020-01-26T10:47:41.225272-05:00 test-server05 mongod: { "atype" : "authCheck", "ts" : { "$date" : "2020-06-11T09:44:11.070-0400" }, "local" : { "ip" : "9.70.147.59", "port" : 27017 }, "remote" : { "ip" : "9.148.202.94", "port" : 60185 }, "users" : [ { "user" : "realAdmin", "db" : "admin" } ], "roles" : [ { "role" : "readWriteAnyDatabase", "db" : "admin" }, { "role" : "userAdminAnyDatabase", "db" : "admin" } ], "param" : { "command" : "find", "ns" : "admin.USERS", "args" : { "find" : "USERS", "filter" : {}, "lsid" : { "id" : { "$binary" : "mV20eHvvRha2ELTeqJxQJg==", "$type" : "04" } }, "$db" : "admin", "$readPreference" : { "mode" : "primaryPreferred" } } }, "result" : 0 }

"mongod:" is used in this filter as a greedy signal that it's a valid MongoDB message, so the latter JSON part is passed for parsing and processing. 

This Filter then populates a Guarium Record object, and add it to the incoming Event as a new _GuardRecord_ field:

    {

      "sequence" => 0,
        "GuardRecord" => "{"sessionId":"mV20eHvvRha2ELTeqJxQJg\u003d\u003d","dbName":"admin","appUserName":"","time":{"timstamp":1591883051070,"minOffsetFromGMT":-240,"minDst":0},"sessionLocator":{"clientIp":"9.148.202.94","clientPort":60185,"serverIp":"9.70.147.59","serverPort":27017,"isIpv6":false,"clientIpv6":"","serverIpv6":""},"accessor":{"dbUser":"realAdmin ","serverType":"MongoDB","serverOs":"","clientOs":"","clientHostName":"","serverHostName":"","commProtocol":"","dbProtocol":"MongoDB native audit","dbProtocolVersion":"","osUser":"","sourceProgram":"","client_mac":"","serverDescription":"","serviceName":"admin","language":"FREE_TEXT","dataType":"CONSTRUCT"},"data":{"construct":{"sentences":[{"verb":"find","objects":[{"name":"USERS","type":"collection","fields":[],"schema":""}],"descendants":[],"fields":[]}],"fullSql":"{\"atype\":\"authCheck\",\"ts\":{\"$date\":\"2020-06-11T09:44:11.070-0400\"},\"local\":{\"ip\":\"9.70.147.59\",\"port\":27017},\"remote\":{\"ip\":\"9.148.202.94\",\"port\":60185},\"users\":[{\"user\":\"realAdmin\",\"db\":\"admin\"}],\"roles\":[{\"role\":\"readWriteAnyDatabase\",\"db\":\"admin\"},{\"role\":\"userAdminAnyDatabase\",\"db\":\"admin\"}],\"param\":{\"command\":\"find\",\"ns\":\"admin.USERS\",\"args\":{\"find\":\"USERS\",\"filter\":{},\"lsid\":{\"id\":{\"$binary\":\"mV20eHvvRha2ELTeqJxQJg\u003d\u003d\",\"$type\":\"04\"}},\"$db\":\"admin\",\"$readPreference\":{\"mode\":\"primaryPreferred\"}}},\"result\":0}","redactedSensitiveDataSql":"{\"atype\":\"authCheck\",\"ts\":{\"$date\":\"2020-06-11T09:44:11.070-0400\"},\"local\":{\"ip\":\"9.70.147.59\",\"port\":27017},\"remote\":{\"ip\":\"9.148.202.94\",\"port\":60185},\"users\":[{\"user\":\"realAdmin\",\"db\":\"admin\"}],\"roles\":[{\"role\":\"readWriteAnyDatabase\",\"db\":\"admin\"},{\"role\":\"userAdminAnyDatabase\",\"db\":\"admin\"}],\"param\":{\"command\":\"find\",\"ns\":\"admin.USERS\",\"args\":{\"filter\":{},\"lsid\":{\"id\":{\"$binary\":\"?\",\"$type\":\"?\"}},\"$readPreference\":{\"mode\":\"?\"},\"find\":\"USERS\",\"$db\":\"admin\"}},\"result\":0}"},"originalSqlCommand":""},"exception":null}",
        "@version" => "1",
        "@timestamp" => 2020-02-25T12:32:16.314Z,
        "type" => "syslog"
    }

Here ends the filter responsibility and process. Note that as this filter takes responsiblity of parsing the command grammar, it parses the DB command into it's atomic parts, which are described in a Construct, though there's another way, where a filter plugin can send the command as-is and leave its parsing to Guardium, if Guardium is already familiar with that data source type. 

The filtered Event, with the attached GuardRecord now, is moved to processing by Guardium universal connector in the output stage, which is responsible for communicating with Guardium and inserting this Guardium Record. 

## Not supported/Future
* Embedded documents as inner objects(?)
* Fields hierarchy inside a Sentence.

## Contribute
See [CONTRIBUTING.md](CONTRIBUTING.md) for ways to raise issues, fix them, discuss enhancements, and just sending us feedback.

<!-- links -->
[logstash-java-plugin-dev]: https://www.elastic.co/guide/en/logstash/current/java-filter-plugin.html

[github-uc-commons]: https://www.github.com/IBM/guardium-universalconnector-commons

[KC-UC-main]: https://www.ibm.com/support/knowledgecenter/SSMPHH_11.3.0/com.ibm.guardium.doc.stap/guc/g_universal_connector.html

[KC-UC-plugin-dev]: https://www.ibm.com/support/knowledgecenter/SSMPHH_11.3.0/com.ibm.guardium.doc.stap/guc/plugin_dev_guide.html

[KC-UC-staging]: https://www.ibm.com/support/knowledgecenter/SSMPHH_11.3.0/com.ibm.guardium.doc.stap/guc/test_filter_guardium.html

