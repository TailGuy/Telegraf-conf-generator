# Telegraf Configuration Generator for OPC UA

## Description

This Python script (`telegraf_conf_generator.py`) reads OPC UA node information from a specified CSV file and generates a Telegraf configuration file (`.conf`). The generated configuration instructs Telegraf to:

1.  Read data from the specified OPC UA nodes on a defined server.
2.  Write the collected data to an InfluxDB v2 instance.
3.  Publish the data for *each individual node* to a unique MQTT topic, using tag filtering to separate the streams.

The script includes validation and sanitization for generated MQTT topic names to avoid using characters restricted by MQTT specifications.

## Requirements

* Python 3.x
* Required Python libraries:
    * `csv`
    * `logging`
    * `os`

## Input CSV Format

The script requires an input CSV file specified by the `HARDCODED_CSV_FILE` variable. This CSV file **must** contain at least the following columns:

* `NodeId`: The full OPC UA Node ID in the format `ns=<namespace_index>;s=<identifier_string>` (e.g., `ns=2;s=MyDevice.TankLevel`). The script currently assumes the identifier type is a string (`s=`).
* `CustomName`: Will be used as the measurement name in InfluxDB and will be used as the topic for mqtt, for example: `Device1/Pressure`.

Example `input_nodes.csv`:

| NodeId              | BrowseName       | CustomName              | DataType | DisplayName          | Description                      |
|---------------------|------------------|-------------------------|----------|----------------------|----------------------------------|
| ns=2;s=Device1.Temp | Device1/Temp     | Sensor_Device1_Temp     | Float    | Temperature Sensor 1 | Reads temperature from Device 1  |
| ns=2;s=Device1.Pressure | Device1/Pressure | Sensor_Device1_Pressure | Double   | Pressure Sensor 1    | Reads pressure from Device 1     |
| ns=2;s=Device2.Status | Device2/Status   | State_Device2_Status    | String   | Status Device 2      | Operating status of Device 2     |

*(Note: `BrowseName`, `DataType`, `DisplayName`, `Description` columns are shown for context but are not used by this generator script)*
## Configuration

Before running the script, you **must** modify the hardcoded configuration values within the `main` function in the `telegraf_conf_generator.py` file:

1.  **`HARDCODED_CSV_FILE`**: Set this to the path of your input CSV file containing the node details (e.g., `"input_nodes.csv"`).
2.  **`HARDCODED_OUTPUT_FILE`**: Specify the desired path and filename for the generated Telegraf configuration file (e.g., `"telegraf_opcua.conf"`).
3.  **`HARDCODED_MQTT_BROKER`**: Set this to the URL of your MQTT broker, including the protocol (e.g., `"tcp://mqtt.example.com:1883"`).
4.  **`HARDCODED_OPCUA_ENDPOINT`**: Set this to the endpoint URL of the OPC UA server that Telegraf should connect to (e.g., `"opc.tcp://opcua.example.com:4840"`). This should match the server from which the nodes in the CSV originate.
5.  **`HARDCODED_INFLUXDB_URL`**: Set this to the URL of your InfluxDB v2 instance (e.g., `"http://influxdb.example.com:8086"`).
6.  **`HARDCODED_LOGLEVEL`**: (Optional) Change the script's logging verbosity. Options include `'DEBUG'`, `'INFO'`, `'WARNING'`, `'ERROR'`, `'CRITICAL'`. The default is `'INFO'`.

## Usage

1.  Modify the hardcoded configuration values in `telegraf_conf_generator.py` as described above.
2.  Run the script from your terminal:

    ```bash
    python telegraf_conf_generator.py
    ```
3.  The script will read the CSV, generate the Telegraf configuration content, perform MQTT topic validation/sanitization, and write the result to the specified output file (`HARDCODED_OUTPUT_FILE`). Progress and summary information will be printed to the console.
4.  Place the generated `.conf` file in your Telegraf configuration directory (e.g., `/etc/telegraf/telegraf.d/`) or specify it when running Telegraf.
5.  Ensure Telegraf has the necessary environment variables set for InfluxDB authentication (`$DOCKER_INFLUXDB_INIT_ADMIN_TOKEN`, `$DOCKER_INFLUXDB_INIT_ORG`) or replace these placeholders in the generated config file with actual values.
6.  Start or reload the Telegraf service.

## Output Configuration Details

The generated Telegraf configuration file (`HARDCODED_OUTPUT_FILE`) will contain:

* **Agent Settings**: Basic Telegraf agent settings like interval, buffer sizes, flush interval.
* **OPCUA Input Plugin (`[[inputs.opcua]]`)**:
    * Configured with the `HARDCODED_OPCUA_ENDPOINT`.
    * Security set to "None" by default (modify the script or the output file if needed).
    * Authentication set to "Anonymous" by default.
    * Includes a `[[inputs.opcua.nodes]]` section for each node listed in the input CSV, using the `CustomName`, namespace, identifier type (`s`), and identifier extracted from the `NodeId`.
* **InfluxDB v2 Output Plugin (`[[outputs.influxdb_v2]]`)**:
    * Configured with the `HARDCODED_INFLUXDB_URL`.
    * Uses environment variables (`$DOCKER_INFLUXDB_INIT_ADMIN_TOKEN`, `$DOCKER_INFLUXDB_INIT_ORG`) for token and organization – **these must be set in Telegraf's environment or replaced manually in the config file**.
    * Targets a bucket named `"OPC UA"`.
* **MQTT Output Plugins (`[[outputs.mqtt]]`)**:
    * A separate `[[outputs.mqtt]]` section is generated for *each node* from the CSV.
    * Configured with the `HARDCODED_MQTT_BROKER`.
    * The `topic` is generated as `telegraf/opcua/<identifier>`, where `<identifier>` is extracted from the `NodeId`. Restricted characters (`+`, `#`, `*`, `>`, leading `$`) in the identifier are replaced with `_`. Warnings are logged if sanitization occurs.
    * Uses `tagpass = { id = ["<full_node_id>"] }` to ensure only data for the specific node is sent to its corresponding topic.
    * Uses JSON data format, QoS 0, and retain set to false.
