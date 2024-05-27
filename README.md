
# Monitoring CouchDB with Zabbix: Efficient Data Fetching and Preprocessing

## Introduction

Monitoring CouchDB with Zabbix can be optimized by minimizing the number of HTTP requests. Instead of fetching each metric individually, we can fetch all necessary statistics in a single request and use Zabbix's preprocessing capabilities to extract the relevant data for individual metrics. This approach reduces the load on both the monitoring system and CouchDB.

Here is an example on how you can explore the CouchDB metrics using the `_stats` endpoint:

```json
curl -s http://USER:PASSWORD@127.0.0.1:5984/_node/couchdb@127.0.0.1/_stats | jq .couchdb.httpd_request_methods.GET

{
  "value": 6116349,
  "type": "counter",
  "desc": "number of HTTP GET requests"
}

curl -s http://USER:PASSWORD@127.0.0.1:5984/_node/couchdb@127.0.0.1/_stats | jq .couchdb.httpd_request_methods.GET.value

6117319

```

## Template Overview

The provided YAML template for Zabbix demonstrates how to achieve this efficiency. The key idea is to fetch the CouchDB statistics once using the `couchdb.get.stats` item and then preprocess this data to obtain specific metrics.
If you want to know more about Couchdb Metrics you can read [this documentation](https://docs.couchdb.org/en/stable/api/server/common.html#node-node-name-stats).

### Key Components of the Template

1. **Zabbix Agent Item**: Fetches the CouchDB statistics.
2. **Preprocessing**: Extracts individual metrics from the fetched data.
3. **Dependent Items**: Use the preprocessed data to create specific metrics.

### Zabbix Agent Item

The `couchdb.get.stats` item is configured to make a single request to CouchDB's `_stats` endpoint through the host's userparameter. This item collects all the necessary statistics in one go.

```yaml
   name: 'Couchdb Get Stats'
   key: couchdb.get.stats
   trends: '0'
   value_type: TEXT
```

### Preprocessing

The collected data from `couchdb.get.stats` is then preprocessed to extract individual metrics. Zabbix's preprocessing step uses JSONPath to filter out specific values.

```yaml
preprocessing:
  - step:
      type: "JSONPath"
      parameters:
        - "$.couchdb.httpd_request_methods.GET"
```

### Dependent Items

Dependent items are created based on the preprocessed data. These items do not make additional requests; instead, they use the data fetched by `couchdb.get.stats`.

```yaml
        - uuid: b5a3e08d18424ac78dc1442c2d244fc4
          name: 'CouchDB: httpd_request_methods.GET'
          type: DEPENDENT
          key: couchdb.httpd_request_methods.GET
          delay: '0'
          trends: 90d
          preprocessing:
            - type: JSONPATH
              parameters:
                - $.couchdb.httpd_request_methods.GET.value
            - type: CHANGE_PER_SECOND
              parameters:
                - ''
          master_item:
            key: couchdb.get.stats
          tags:
            - tag: Application
              value: CouchDB
```

## Using UserParameter for Monitoring

In addition to the template configuration, a `UserParameter` file is required on the monitored server. This file allows Zabbix Agent to execute custom commands and scripts to fetch CouchDB metrics. Below is an example configuration for the `userparameter_couchdb.conf` file:

```ini
UserParameter=couchdb.get.stats, curl -s http://USER:PASSWORD@127.0.0.1:5984/_node/couchdb@127.0.0.1/_stats                                                                      
UserParameter=couchdb.min_doc_opened, grep YOUR_DATABASE /var/log/couchdb/current | cut -d " " -f 14 | grep -E '[0-9]+' | awk '!/\./' | awk 'NR==1 || $1<min {min=$1} END {print min}'             
UserParameter=couchdb.max_doc_opened, grep YOUR_DATABASE /var/log/couchdb/current | cut -d " " -f 14 | grep -E '[0-9]+' | awk '!/\./' | awk 'NR==1 || $1>max {max=$1} END {print max}'             
UserParameter=couchdb.avg_doc_opened, grep YOUR_DATABASE /var/log/couchdb/current | cut -d " " -f 14 | grep -E '[0-9]+' | awk '!/\./' | awk '{sum+=$1; count++} END {avg=sum/count; print avg}'

```

### Steps to Configure UserParameter

1. **Create the UserParameter File**: Save the above content to a file named `userparameter_couchdb.conf` on your monitored server.

2. **Place the File in the Correct Directory**: Move `userparameter_couchdb.conf` to the Zabbix Agent configuration directory, typically `/etc/zabbix/zabbix_agentd.d/`.

3. **Restart Zabbix Agent**: Restart the Zabbix Agent to apply the new configuration:
   ```sh
   sudo systemctl restart zabbix-agent
   ```

4. **Verify Configuration**: Ensure the Zabbix Agent is running correctly and is able to execute the custom commands defined in the `userparameter_couchdb.conf` file.

## Benefits of This Approach

1. **Reduced HTTP Requests**: By fetching all required statistics in a single request, we minimize the number of HTTP requests to CouchDB, reducing network overhead and load on the database server.
2. **Efficiency in Data Handling**: Zabbix's preprocessing allows efficient extraction and transformation of data, enabling the creation of multiple metrics from a single data source.
3. **Simplified Configuration**: Managing a single agent item and multiple dependent items simplifies the configuration and maintenance of the monitoring setup.

## Conclusion

Using a single request to fetch CouchDB statistics and leveraging Zabbix's preprocessing capabilities is an efficient way to monitor CouchDB. This method reduces the load on both the monitoring system and the CouchDB server, ensuring a more streamlined and effective monitoring setup.

For further details, refer to the provided YAML template and customize it according to your specific monitoring needs.
