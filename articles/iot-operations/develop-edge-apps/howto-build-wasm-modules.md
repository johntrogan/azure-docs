---
title: Build WASM modules for data flows
description: Learn how to build WebAssembly (WASM) modules for data flows using the Azure IoT Operations Data Flow extension for VS Code or the aio-dataflow CLI.
author: dominicbetts 
ms.author: dobett 
ms.topic: how-to
ms.date: 03/30/2026
ms.service: azure-iot-operations

# CustomerIntent: As a developer, I want to understand how to use the VS Code extension or the aio-dataflow CLI to build and test WASM modules to use in data flow graphs or the HTTP/REST connector.
---

# Build WASM modules for data flows

The custom WebAssembly (WASM) data processing feature in Azure IoT Operations enables real time telemetry data processing within your Azure IoT Operations cluster. By deploying custom WASM modules, you can define and execute data transformations as part of your data flow graph, the HTTP/REST connector, or the MQTT connector.

This article describes how to use the **Azure IoT Operations Data Flow** VS Code extension or the `aio-dataflow` CLI to develop and test your WASM modules locally before you deploy them to your Azure IoT Operations cluster. The VS Code extension provides a debugging experience. You'll learn how to:

- Run a graph application locally by executing a prebuilt graph with sample data to understand the basic workflow.
- Create custom WASM modules by building new operators in Python and Rust with map and filter functionality.
- Use the state store to maintain state across message processing.
- Use schema registry validation to validate message formats using JSON schemas before processing.
- Debug WASM modules in VS Code by using breakpoints and step-through debugging for local development.

The extension and CLI tool are supported on the following platforms:

- Linux
- Windows Subsystem for Linux (WSL)
- Windows (be sure to use a Windows shell such as PowerShell or Command Prompt when you run the extension commands on Windows)

> [!IMPORTANT]
> Data flow graphs currently only support MQTT, Kafka, and OpenTelemetry endpoints. Other endpoint types like Azure Data Lake, Microsoft Fabric OneLake, Azure Data Explorer, and local storage aren't supported. For more information, see [Known issues](../troubleshoot/known-issues.md#data-flow-graphs-only-support-specific-endpoint-types).

To learn more about graphs and WASM in Azure IoT Operations, see:

- [Use a data flow graph with WebAssembly modules](../connect-to-cloud/howto-dataflow-graph-wasm.md)
- [Transform incoming data with WebAssembly modules](../discover-manage-assets/howto-use-http-connector.md#transform-incoming-data)

## Prerequisites

# [VS Code extension](#tab/vscode)

Development environment:

- [Visual Studio Code](https://code.visualstudio.com/)
- (Optional) [RedHat YAML extension](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-yaml) for VS Code
- [Azure IoT Operations Data Flow extension](https://marketplace.visualstudio.com/items?itemName=ms-azureiotoperations.azure-iot-operations-data-flow-vscode) for VS Code.
- [CodeLLDB extension](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb) for VS Code to enable debugging of WASM modules
- Docker

# [aio-dataflow CLI](#tab/cli)

- [aio-dataflow CLI](TODO TODO add link to aio-dataflow CLI installation instructions)
- [Azure CLI](/cli/azure/install-azure-cli)
- [ORAS CLI](https://oras.land/docs/installation/)
- Docker

---

Docker images:

```bash
docker pull mcr.microsoft.com/azureiotoperations/processor-app:1.1.5
docker tag mcr.microsoft.com/azureiotoperations/processor-app:1.1.5 host-app

docker pull mcr.microsoft.com/azureiotoperations/devx-runtime:0.1.8
docker tag mcr.microsoft.com/azureiotoperations/devx-runtime:0.1.8 devx

docker pull mcr.microsoft.com/azureiotoperations/statestore-cli:0.0.2
docker tag mcr.microsoft.com/azureiotoperations/statestore-cli:0.0.2 statestore-cli

docker pull eclipse-mosquitto
```

## Run a graph application locally

# [VS Code extension](#tab/vscode)

This example uses a sample workspace that contains all the necessary resources to build and run a graph application locally using the VS Code extension.

### Open the sample workspace in VS Code

Clone the [Explore IoT Operations](https://github.com/Azure-Samples/explore-iot-operations) repository if you haven't already.

Open the `samples/wasm` folder in Visual Studio Code by selecting **File > Open Folder** and navigating to the `samples/wasm` folder.

### Build the operators

Press `Ctrl+Shift+P` to open the command palette and search for **Azure IoT Operations: Build All Data Flow Operators**. Select **release** as the build mode.

This command builds all the operators in the workspace and creates `.wasm` files in the `operators` folder. You use the `.wasm` files to run the graph application locally.

### Run the graph application locally

To start the local execution environment, press `Ctrl+Shift+P` to open the command palette and search for **Azure IoT Operations: Start Development Environment**. Select **release** as the run mode.

When the local execution environment is running, press `Ctrl+Shift+P` to open the command palette and search for **Azure IoT Operations: Run Application Graph**. Select **release** as the run mode. This command runs the graph application locally by using the local execution environment with the `graph.dataflow.yaml` file in the workspace.

It also reads from `hostapp.env.list` to set the environment variable `TK_CONFIGURATION_PARAMETERS` for data flow operator configuration parameters.

When you're prompted for input data, select the `data-and-images` folder in the workspace. This folder contains the input data files for the graph application including temperature and humidity data, and some images for the snapshot module.

Wait until you see a VS Code notification that the logs are ready: `Log files for the run can be found at ...\wasm\data-and-images\output\logs`.

The output is located in the `output` folder under the `data-and-images` folder. You can open the `output` folder in the workspace to see the output files. The `.txt` file with the date and time for filename contains the processed data, and it looks like the following example:

```output
{"tst":"2025-09-19T04:19:13.530381+0000","topic":"sensors","qos":0,"retain":0,"payloadlen":312,"properties":{"payload-format-indicator":1,"message-expiry-interval":10,"correlation-data":"...","user-properties":{"__ts":"001758255553528:00000:...","__protVer":"1.0","__srcId":"mqtt-source"},"content-type":"application/json"},"payload":{"temperature":[{"count":2,"max":653.888888888889,"min":204.44444444444449,"average":429.16666666666669,"last":204.44444444444449,"unit":"C","overtemp":true}],"humidity":[{"count":3,"max":85,"min":45,"average":69.666666666666671,"last":79}],"object":[{"result":"notebook, notebook computer; sliding door"}]}}
```

The output shows that the graph application processed the input data and generated the output. The output includes temperature and humidity data, and the objects detected in the images.

# [aio-dataflow CLI](#tab/cli)

Clone the [Explore IoT Operations](https://github.com/Azure-Samples/explore-iot-operations) repository if you haven't already:

```bash
git clone https://github.com/Azure-Samples/explore-iot-operations.git
```

### Build the operators

Navigate to your graph application directory and build all operators:

```bash
cd explore-iot-operations/samples/wasm
aio-dataflow build --app .
```

The `--app` flag specifies the directory that contains the `graph.dataflow.yaml` file and the `operators` folder. The CLI builds all operators in the workspace and creates `.wasm` files.

### Run the graph application locally

Start the local Docker execution environment, then run the graph:

```bash
aio-dataflow run start
aio-dataflow test --app . test-runner/tests/t03-complex-full-pipeline
aio-dataflow run stop
```

> [!TIP]
> If you plan to run multiple tests, you can start the environment once with `aio-dataflow run start`, then run multiple test commands, and stop the environment at the end with `aio-dataflow run stop`.

The `test` command runs the graph defined in the test case's `.test.yaml` file, feeds the input data, and compares the output against expected results. The sample repository includes multiple test scenarios such as:

| Test | Scenario | Description |
|---|---|---|
| `t01-simple-temp-conversion` | Simple | Basic Fahrenheit to Celsius map transform |
| `t02-complex-temp-pipeline` | Complex | Branch → map → filter → accumulate → enrichment |
| `t03-complex-full-pipeline` | Complex | Temperature, humidity, machine learning inference |
| `t08-schema-valid` | Schema | Valid payload passes through schema validation |
| `t11-statestore-enrichment` | State store | State store key-value lookup enrichment |

To run all tests:

```bash
aio-dataflow run start
aio-dataflow test --app . test-runner/tests
aio-dataflow run stop
```

---

## Create a new graph with custom WASM modules

This scenario shows you how to create a new graph application with custom WASM modules. The graph application consists of two operators: a `map` operator that converts temperature values from Fahrenheit to Celsius, and a `filter` operator that filters out messages with temperature values above 500°C.

Currently, don't use hyphens (`-`) or underscores (`_`) in operator names. The VS Code extension enforces this requirement, but if you create or rename modules manually it causes issues. Use simple alphanumeric names for modules like `filter`, `map`, `stateenrich`, or `schemafilter`.

# [VS Code extension](#tab/vscode)

Instead of using an existing sample workspace, you create a new workspace from scratch. This process lets you learn how to create a new graph application and program the operators in Python and Rust.

### Create a new graph application project in Python

Press `Ctrl+Shift+P` to open the VS Code command palette and search for **Azure IoT Operations: Create a new Data Flow Application**:

1. For the folder, select a folder where you want to create the project. You can create a new folder for this project.
1. Enter `my-graph` as the name.
1. Select **Python** as the language.
1. Select **Map** as the type.
1. Enter `map` as the name.

You now have a new VS Code workspace with the basic project structure and starter files. The starter files include the `graph.dataflow.yaml` file and the map operator template source code.

> [!IMPORTANT]
> To use a Python module in a deployed Azure IoT Operations instance, you must deploy the instance with the broker memory profile set to **Medium** or **High**. If you set the memory profile to **Low** or **Tiny**, the instance can't pull the Python module.

### Add Python code for the map operator module

Open the `operators/map/map.py` file and replace the contents with the following code to convert an incoming temperature value from Fahrenheit to Celsius:

```python
import json
from map_impl import exports
from map_impl import imports
from map_impl.imports import types

class Map(exports.Map):
    def init(self, configuration) -> bool:
        imports.logger.log(imports.logger.Level.INFO, "module4/map", "Init invoked")
        return True

    def process(self, message: types.DataModel) -> types.DataModel:
        # TODO: implement custom logic for map operator
        imports.logger.log(imports.logger.Level.INFO, "module4/map", "processing from python")

        # Ensure the input is of the expected type
        if not isinstance(message, types.DataModel_Message):
            raise ValueError("Unexpected input type: Expected DataModel_Message")

        # Extract and decode the payload
        payload_variant = message.value.payload
        if isinstance(payload_variant, types.BufferOrBytes_Buffer):
            # It's a Buffer handle - read from host
            imports.logger.log(imports.logger.Level.INFO, "module4/map", "Reading payload from Buffer")
            payload = payload_variant.value.read()
        elif isinstance(payload_variant, types.BufferOrBytes_Bytes):
            # It's already bytes
            imports.logger.log(imports.logger.Level.INFO, "module4/map", "Reading payload from Bytes")
            payload = payload_variant.value
        else:
            raise ValueError("Unexpected payload type")

        decoded = payload.decode("utf-8")

        # Parse the JSON data
        json_data = json.loads(decoded)

        # Check and update the temperature value
        if "temperature" in json_data and "value" in json_data["temperature"]:
            temp_f = json_data["temperature"]["value"]
            if isinstance(temp_f, int):
                # Convert Fahrenheit to Celsius
                temp_c = round((temp_f - 32) * 5.0 / 9.0)

                # Update the JSON data
                json_data["temperature"]["value"] = temp_c
                json_data["temperature"]["unit"] = "C"

                # Serialize the updated JSON back to bytes
                updated_payload = json.dumps(json_data).encode("utf-8")

                # Update the message payload
                message.value.payload = types.BufferOrBytes_Bytes(value=updated_payload)

        return message
```

Make sure Docker is running. Then, press `Ctrl+Shift+P` to open the command palette and search for **Azure IoT Operations: Build All Data Flow Operators**. Create a **release** module.

The build process places the `map.wasm` file for the `map` operator in the `operators/map/bin/release` folder.

### Add Rust code for the filter operator module

Create a new operator by pressing `Ctrl+Shift+P` to open the command palette and search for **Azure IoT Operations: Create a new Data Flow Operator**:

1. Select **Rust** as the language.
1. Select **Filter** as the operator type.
1. Enter `filter` as the name.

Open the `operators/filter/src/lib.rs` file and replace the contents with the following code to filter out values where the temperature is above 500°C:

```rust
mod filter_operator {
    use wasm_graph_sdk::macros::filter_operator;
    use serde_json::Value;

    fn filter_init(_configuration: ModuleConfiguration) -> bool {
        // Add code here to process the module init properties and module schemas from the configuration

        true
    }

    #[filter_operator(init = "filter_init")]
    fn filter(input: DataModel) -> Result<bool, Error> {

        // Extract payload from input to process
        let payload = match input {
            DataModel::Message(Message {
                payload: BufferOrBytes::Buffer(buffer),
                ..
            }) => buffer.read(),
            DataModel::Message(Message {
                payload: BufferOrBytes::Bytes(bytes),
                ..
            }) => bytes,
            _ => return Err(Error { message: "Unexpected input type".to_string() }),
        };

        // ... perform filtering logic here and return boolean
        if let Ok(payload_str) = std::str::from_utf8(&payload) {
            if let Ok(json) = serde_json::from_str::<Value>(payload_str) {
                if let Some(temp_c) = json["temperature"]["value"].as_i64() {
                    // Return true if temperature is above 500°C
                    return Ok(temp_c > 500);
                }
            }
        }

        Ok(false)
   }
}
```

Open the `operators/filter/Cargo.toml` file and add the following dependencies:

```toml
[dependencies]
# ...
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

Make sure Docker is running. Then, press `Ctrl+Shift+P` to open the command palette and search for **Azure IoT Operations: Build All Data Flow Operators**. Create a **release** module.

The build process places the `filter.wasm` file for the `filter` operator in the `operators/filter/bin/release` folder.

# [aio-dataflow CLI](#tab/cli)

To create and build new operators with the aio-dataflow CLI, use one of the sample projects as a template and create new operator source files in the `operators` folder. Then, use the `aio-dataflow build` command to build the operators and generate the `.wasm` files:

- For Rust projects, see the examples in the `explore-iot-operations/samples/wasm` folder. 
- For Python projects, see the examples in the `explore-iot-operations/samples/wasm-python` folder.

---

### Run the graph application locally with sample data

# [VS Code extension](#tab/vscode)

Open the `graph.dataflow.yaml` file and replace the contents with the following code:

```yaml
metadata:
    $schema: "https://www.schemastore.org/aio-wasm-graph-config-1.0.0.json"
    name: "Temperature Monitoring"
    description: "A graph that converts temperature from Fahrenheit to Celsius, if temperature is above 500°C, then sends the processed data to the sink."
    version: "1.0.0"
    vendor: "Microsoft"
moduleRequirements:
    apiVersion: "1.1.0"
    runtimeVersion: "1.1.0"
operations:
  - operationType: source
    name: source
  - operationType: map
    name: map
    module: map
  - operationType: filter
    name: filter
    module: filter
  - operationType: sink
    name: sink
connections:
  - from:
      name: source
    to:
      name: map
  - from:
      name: map
    to:
      name: filter
  - from:
      name: filter
    to:
      name: sink
```

Copy the `data` folder that contains the sample data from the cloned samples repository `explore-iot-operations\samples\wasm\data` to the current workspace. The `data` folder contains three JSON files with sample input temperature data.

If you previously stopped the local execution environment, press `Ctrl+Shift+P` to open the command palette and search for **Azure IoT Operations: Start Development Environment**. Select **release** as the run mode.

Press `Ctrl+Shift+P` to open the command palette and search for **Azure IoT Operations: Run Application Graph**:

1. Select the `graph.dataflow.yaml` graph file.
1. Select **release** as the run mode.
1. Select the `data` folder you copied to the workspace.

The DevX container launches to run the graph. The processed result is saved in the `data/output` folder. The text file in the `output` folder contains the processed data with the temperature converted to Celsius and filtered based on the threshold.

To learn how to deploy your custom WASM modules and graph to your Azure IoT Operations instance, see [Deploy WASM modules and data flow graphs](../connect-to-cloud/howto-dataflow-graph-wasm.md).

# [aio-dataflow CLI](#tab/cli)

To run your graph application locally with the aio-dataflow CLI, use the `aio-dataflow test` command with a test case that points to your graph configuration and input data. You can create a new test case YAML file in the `test-runner/tests` folder that references your `graph.dataflow.yaml` and input data files. Then, run the test with the following command:

```bash
aio-dataflow run start
aio-dataflow test --app . test-runner/tests/my-test-case
aio-dataflow run stop
```

Look at the example test case files in the `test-runner/tests` folder for reference on how to structure your test case YAML file. You can test both Python and Rust modules with the CLI tool by pointing to the appropriate graph configuration and input data in your test case file.

---

## State store support for WASM operators

This example shows how to use the state store with WASM operators. The state store lets operators persist and retrieve data across message processing, and enables stateful operations in your data flow graphs.

### Open the state store sample workspace

Clone the [Explore IoT Operations](https://github.com/Azure-Samples/explore-iot-operations) repository if you haven't already.

Open the `explore-iot-operations/samples/wasm/statestore-scenario` folder. This folder contains the following resources:

- `graph.dataflow.yaml` - The data flow graph configuration with a state store enabled operator.
- `statestore.json` - State store configuration with key-value pairs.
- `data/` - Sample input data for testing.
- `operators/` - Source code for the otel-enrich and filter operators.

### Configure the state store

1. Open the `statestore.json` file to view the current state store configuration.

1. You can modify the values of `factoryId` or `machineId` for testing. The enrichment parameter references these keys.

1. Open `graph.dataflow.yaml` and review the enrichment configuration. The significant sections of this file are shown in the following snippet:

    ```yaml
    moduleConfigurations:
      - name: module-otel-enrich/map
        parameters:
          enrichKeys:
            name: enrichKeys
            description: Comma separated list of DSS keys which will be fetched and added as attributes
            default: factoryId,machineId
    operations:
      - operationType: "source"
        name: "source"
      - operationType: "map"
        name: "module-otel-enrich/map"
        module: "otel-enrich"
      - operationType: "sink"
        name: "sink"
    ```

    The `enrichKeys` parameter's default value (`factoryId,machineId`) determines which keys are fetched from the state store and added as attributes.

1. Ensure each key listed in `enrichKeys` exists in `statestore.json`. If you add or remove keys in `statestore.json`, update the `default` value or override it by using the `TK_CONFIGURATION_PARAMETERS` environment variable.

### Update test data (optional)

You can modify the test data in the `data/` folder to experiment with different input values. The sample data includes temperature readings.

### Build and run the state store scenario

# [VS Code extension](#tab/vscode)

If you previously stopped the local execution environment, press `Ctrl+Shift+P` to open the command palette and search for **Azure IoT Operations: Start Development Environment**. Select **release** as the run mode.

1. Press `Ctrl+Shift+P` to open the command palette and search for **Azure IoT Operations: Build All Data Flow Operators**. Select **release** as the build mode. Wait for the build to complete.

1. Press `Ctrl+Shift+P` again and search for **Azure IoT Operations: Run Application Graph**. Select **release** as the run mode.

1. Select the `data` folder in your VS Code workspace for your input data. The DevX container launches to run the graph with the sample input.

### Verify state store functionality

After the graph execution completes:

1. View the results in the `data/output/` folder.

1. Open the generated `.txt` file to see the processed data. The `user properties` section of the output messages includes the `factoryId` and `machineId` values retrieved from the state store.

1. Check the logs in `data/output/logs/host-app.log` to verify that the `otel-enrich` operator retrieved values from the state store and added them as user properties to the messages.

# [aio-dataflow CLI](#tab/cli)

Review the test run configuration file: `explore-iot-operations/samples/wasm/test-runner/tests/t11-statestore-enrichment/t11-statestore-enrichment.test.yaml`.

If you previously stopped the local execution environment, run the following command to start it again:

```bash
aio-dataflow run start
```

Then, build and run the state store test with the following commands:

```bash
aio-dataflow build --app ./statestore-scenario
aio-dataflow test --app . test-runner/tests/t11-statestore-enrichment/t11-statestore-enrichment.test.yaml
```

Check the command output shows that the tests passed.

If you've finished using the local execution environment, stop it with the following command:

```bash
aio-dataflow run stop
```

Verify the test outputs and logs to confirm that state store enrichment is working as expected, with messages being enriched with values from the state store based on the specified keys.

---

## Schema registry support for WASM modules

This example shows you how to use the schema registry with WASM modules. The schema registry lets you validate message formats and ensure data consistency in your data flow processing.

### Open the schema registry sample workspace

Clone the [Explore IoT Operations](https://github.com/Azure-Samples/explore-iot-operations) repository if you haven't already.

Open the `samples/wasm/schema-registry-scenario` folder. This folder contains the following resources:

- `graph.dataflow.yaml` - The data flow graph configuration.
- `tk_schema_config.json` - The JSON schema that the host app uses locally to validate incoming message payloads before they reach downstream operators. Keep this file in sync with any schema you publish to your Microsoft Azure environment.
- `data/` - Sample input data with different message formats for testing.
- `operators/filter/` - Source code for the filter operator.
- (Optional) `hostapp.env.list` - The VS Code extension autogenerates one at run time adding `TK_SCHEMA_CONFIG_PATH=tk_schema_config.json`. If you supply your own schema file, make sure that the variable points to it.

### Understand the schema configuration

Open the `tk_schema_config.json` file to view the schema definition:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "humidity": {
      "type": "integer"
    },
    "temperature": {
      "type": "number"
    }
  }
}
```

This schema defines a top-level JSON object that can contain two numeric properties: `humidity` (integer) and `temperature` (number). Add a `"required": ["humidity", "temperature"]` array if you need to make them both required, or extend the `properties` section as your payload format evolves.

The following limitations apply to the local schema file (`tk_schema_config.json`). The schema file:
- Uses standard JSON Schema draft-07 syntax, the same draft supported by the Azure IoT Operations schema registry.
- Is functionally the same content you register in the cloud schema registry. You can copy and paste between the two to stay consistent.
- Is referenced by the host app through the environment variable `TK_SCHEMA_CONFIG_PATH`.

Currently, the following limitations apply in the local development runtime:
- Only one schema config file is loaded. No multi-file include or directory scanning is supported.
- `$ref` to external files or URLs isn't supported locally. Keep the schema self‑contained. You can use internal JSON pointer references like `{"$ref":"#/components/..."}`.
- Commonly used draft-07 keywords like `type`, `properties`, `required`, `enum`, `minimum`, `maximum`, `allOf`, `anyOf`, `oneOf`, `not`, and `items` all work. The underlying validator might ignore less common or advanced features like `contentEncoding` and `contentMediaType`.
- Keep the size of your schemas to less than ~1 MB for fast cold starts.
- Versioning and schema evolution policies aren't enforced locally. You're responsible for staying aligned with the cloud registry.

If you rely on advanced constructs that fail validation locally, validate the same schema against the cloud registry after publishing to ensure parity.

To learn more, see [Azure IoT Operations schema registry concepts](../connect-to-cloud/concept-schema-registry.md).

### Review the test data

The `data/` folder contains three test files:

- `temperature_humidity_payload_1.json` - Contains both temperature and humidity data. However, the humidity value (175.1) isn't an integer as specified in the schema, so the data fails schema validation and is filtered out.
- `temperature_humidity_payload_2.json` - Contains only humidity data and is filtered out.
- `temperature_humidity_payload_3.json` - Contains both temperature and humidity data and passes validation.

### Build and run the schema registry scenario

# [VS Code extension](#tab/vscode)

If you previously stopped the local execution environment, press `Ctrl+Shift+P` to open the command palette and search for **Azure IoT Operations: Start Development Environment**. Select **release** as the run mode.

1. Press `Ctrl+Shift+P` to open the command palette and search for **Azure IoT Operations: Build All Data Flow Operators**.

1. Select **release** as the build mode. Wait for the build to complete.

1. Press `Ctrl+Shift+P` again and search for **Azure IoT Operations: Run Application Graph**.

1. Select the `graph.dataflow.yaml` graph file.

1. Select **release** as the run mode.

1. Select the `data` folder in your VS Code workspace for your input data. The DevX container launches to run the graph with the sample input.

### Verify schema validation

After processing completes:

1. Check the `data/output/` folder for the results.

1. The output contains only the processed version of the `temperature_humidity_payload_3.json` message because it conforms to the schema.

1. The `data/output/logs/host-app.log` file contains log entries indicating which messages are accepted or rejected based on schema validation.

This example shows how the schema registry validates incoming messages and filters out messages that don't conform to the defined schema.

Stop the local execution environment when you're done testing in release mode by pressing `Ctrl+Shift+P` and searching for **Azure IoT Operations: Stop Development Environment**.

# [aio-dataflow CLI](#tab/cli)

Review the test run configuration files:

- `explore-iot-operations/samples/wasm/test-runner/tests/t08-schema-valid/t08-schema-valid.test.yaml`.
- `explore-iot-operations/samples/wasm/test-runner/tests/t09-schema-invalid/t09-schema-invalid.test.yaml`.
- `explore-iot-operations/samples/wasm/test-runner/tests/t10-schema-mixed/t10-schema-mixed.test.yaml`.

If you previously stopped the local execution environment, run the following command to start it again:

```bash
aio-dataflow run start
```

Then, build and run the state store test with the following commands:

```bash
aio-dataflow build --app ./schema-registry-scenario
aio-dataflow test --app . test-runner/tests/t08-schema-valid/t08-schema-valid.test.yaml
aio-dataflow test --app . test-runner/tests/t09-schema-invalid/t09-schema-invalid.test.yaml
aio-dataflow test --app . test-runner/tests/t10-schema-mixed/t10-schema-mixed.test.yaml
```

If you've finished using the local execution environment, stop it with the following command:

```bash
aio-dataflow run stop
```

Verify the test outputs and logs to confirm that schema validation is working as expected, with valid messages being processed and invalid messages being filtered out.

---


## Debug WASM modules in VS Code

This example shows you how to debug WASM modules locally using breakpoints and the integrated debugger in VS Code.

Run the [Schema registry support for WASM modules](#schema-registry-support-for-wasm-modules) example to set up the sample workspace.

### Set up debugging

1. Open the file `operators/filter/src/lib.rs` in the `schema-registry-scenario` workspace.

1. Locate the `filter` function and set a breakpoint by clicking in the margin next to the line number or by pressing `F9`.

    ```rust
    fn filter(input: DataModel) -> Result<bool, Error> {
        let DataModel::Message(message) = input else {
            return Err(Error {message: "Unexpected input type.".to_string()});
        };
        // ... rest of function
    }
    ```

### Build for debugging

1. Press `Ctrl+Shift+P` to open the command palette and search for **Azure IoT Operations: Build All Data Flow Operators**.

1. Select **debug** as the build mode. Wait for the build to complete.

### Run with debugging enabled

Press `Ctrl+Shift+P` to open the command palette and search for **Azure IoT Operations: Start Development Environment**. Select **debug** as the run mode.

1. Press `Ctrl+Shift+P` and search for **Azure IoT Operations: Run Application Graph**.

1. Select the `lldb-debug.graph.dataflow.yaml` graph file.

1. Select **debug** as the run mode.

1. Select the `data` folder in your VS Code workspace for your input data. The DevX container launches to run the graph with the sample input.

1. After the DevX container launches, you see the host-app container start with an `lldb-server` for debugging.

### Debug the WASM module

1. The execution automatically stops at the breakpoint you set in the `filter` function.

1. Use the VS Code debugging interface to:
   - Inspect variable values in the **Variables** panel.
   - Step through code using `F10` or `F11`.
   - View the call stack in the **Call Stack** panel.
   - Add watches for specific variables or expressions.

1. Continue execution by pressing `F5` or selecting the **Continue** button.

1. The debugger stops at the breakpoint for each message being processed, letting you inspect the data flow.

### Debugging tips

- Use the **Debug Console** to evaluate expressions and inspect runtime state.
- Set conditional breakpoints by right-clicking on a breakpoint and adding conditions.
- Use `F9` to toggle breakpoints on and off without removing them.
- The **Variables** panel shows the current state of local variables and function parameters.

This debugging capability enables you to troubleshoot issues, understand data flow, and validate your WASM module logic before deploying to production.

## Test your modules

### Unit testing

Extract your core logic into plain functions that you can test without WASM:

# [Rust](#tab/rust)

```rust
// In src/lib.rs - extract the conversion logic
pub fn fahrenheit_to_celsius(f: f64) -> f64 {
    (f - 32.0) * 5.0 / 9.0
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_boiling_point() {
        assert!((fahrenheit_to_celsius(212.0) - 100.0).abs() < 0.001);
    }

    #[test]
    fn test_freezing_point() {
        assert!((fahrenheit_to_celsius(32.0) - 0.0).abs() < 0.001);
    }

    #[test]
    fn test_body_temperature() {
        assert!((fahrenheit_to_celsius(98.6) - 37.0).abs() < 0.001);
    }
}
```

```bash
cargo test  # Runs tests without WASM target
```

# [Python](#tab/python)

```python
# test_converter.py
def fahrenheit_to_celsius(f):
    return (f - 32) * 5.0 / 9.0

def test_boiling_point():
    assert abs(fahrenheit_to_celsius(212) - 100.0) < 0.001

def test_freezing_point():
    assert abs(fahrenheit_to_celsius(32) - 0.0) < 0.001

def test_body_temperature():
    assert abs(fahrenheit_to_celsius(98.6) - 37.0) < 0.001
```

```bash
pytest test_converter.py
```

---

### Inspect WASM output

Verify your module exports the expected interfaces before pushing to a registry:

```bash
wasm-tools component wit your-module.wasm
```

This shows the WIT interfaces your module implements. Verify you see the expected `map`, `filter`, or `branch` export.

### Local testing with the aio-dataflow CLI

The `aio-dataflow` CLI lets you test your WASM modules locally against a graph definition without deploying to a cluster. The CLI uses Docker containers to simulate the dataflow runtime.

To create a test case, create a directory with the following structure:

```
my-test/
  my-test.test.yaml     # Test descriptor
  input/                 # Input data files
    temperature.json
  expected/              # Expected output
    expected.json
```

Define the test in the `.test.yaml` file:

```yaml
name: "Temperature F to C conversion"
graph: "../graph-simple.yaml"
input: "./input"
expected: "./expected/expected.json"
timeout: 90000
select: ["payload"]
```

The `graph` field points to the graph definition, `input` contains the test data, and `expected` has the output to compare against. The `select` field specifies which fields of the output to compare.

Run the test:

```bash
aio-dataflow run start
aio-dataflow build --app .
aio-dataflow test --app . my-test
aio-dataflow run stop
```

For a complete set of test examples, see the [test-runner/tests](https://github.com/Azure-Samples/explore-iot-operations/tree/main/samples/wasm/test-runner/tests) directory in the samples repository.

### End-to-end testing on a cluster

For integration testing, deploy your module to a development cluster and use MQTT to send test data:

1. Push the module to a test registry.
2. Deploy a DataflowGraph pointing at the test registry.
3. Subscribe to the output topic: `mosquitto_sub -h localhost -t "output/topic" -v`
4. Publish test messages: `mosquitto_pub -h localhost -t "input/topic" -m '{"temperature": {"value": 72}}'`
5. Verify the output matches expectations.
6. Check pod logs for errors: `kubectl logs -l app=aio-dataflow -n azure-iot-operations --tail=50`

## Troubleshoot

### Build errors

| Error | Cause | Fix |
|---|---|---|
| `error[E0463]: can't find crate for std` | Missing WASM target | Run `rustup target add wasm32-wasip2` |
| `error: no matching package found` for `wasm_graph_sdk` | Missing cargo registry | Add the `[registries]` block to `.cargo/config.toml` as shown in [Run a graph application locally](#run-a-graph-application-locally) |
| `componentize-py` can't find WIT files | Schema path wrong | Use `-d` flag with the full path to the schema directory. All `.wit` files must be present because they reference each other. |
| `componentize-py` version mismatch | Bindings generated with different version | Delete the generated bindings directory and regenerate with the same `componentize-py` version |
| `wasm-tools` component check fails | Wrong target or missing component adapter | Ensure you're using `wasm32-wasip2` (not `wasm32-wasi` or `wasm32-unknown-unknown`) |

### Runtime errors

| Symptom | Cause | Fix |
|---|---|---|
| Operator crashes with WASM backtrace | Missing or invalid configuration parameters | Add defensive parsing in `init` with defaults. See [Module configuration parameters](concepts-wasm-modules.md#module-configuration-parameters). |
| `init` returns `false`, dataflow won't start | Configuration validation failed | Check dataflow logs for error messages. Verify `moduleConfigurations` names match your code. |
| Module loads but produces no output | `process` returning errors or filter dropping everything | Add logging in `process` to trace data flow. |
| `Unexpected input type` | Module received wrong `data-model` variant | Add a type check at the start of `process` and handle unexpected variants. |
| Module works alone but crashes in complex graph | Missing config when reused across nodes | Each graph node needs its own `moduleConfigurations` entry. |

### Common pitfalls

- **Forgetting `--artifact-type` on ORAS push.** Without it, the operations experience UI won't display your module correctly.
- **Mismatched `name` in `moduleConfigurations`.** The name must be `<module>/<operator>` (for example, `module-temperature/filter`), matching the graph definition's `operations` section.
- **Using `wasm32-wasi` instead of `wasm32-wasip2`.** Azure IoT Operations requires the WASI Preview 2 target.
- **Python: working outside the samples repo without copying the schema directory.** All `.wit` files must be co-located because they reference each other.

### Known issues

- **Boolean values in YAML**: Boolean values must be quoted as strings to avoid validation errors. For example, use `"True"` and `"False"` instead of `true` and `false`.

  Example error when using unquoted booleans:

  ```output
  * spec.connections[2].from.arm: Invalid value: "boolean": spec.connections[2].from.arm in body must be of type string: "boolean"
  * spec.connections[2].from.arm: Unsupported value: false: supported values: "False", "True"
  ```

- **Python module requirements**: To use Python modules, Azure IoT Operations must be deployed with the MQTT broker configured to use the **Medium** or **High** memory profile. Python modules can't be pulled when the memory profile is set to **Low** or **Tiny**.

- **Module deployment timing**: Pulling and applying WASM modules might take some time, typically around a minute, depending on network conditions and module size.

- **Build error details**: When a build fails, the error message in the pop-up notification might not provide sufficient detail. Check the terminal output for more specific error information.

- **Windows compatibility**: On Windows, the first time you run a graph application, you might encounter an error "command failed with exit code 1." If this error occurs, retry the operation and it should work correctly.

- **Host app stability**: The local execution environment might occasionally stop working and require restarting to recover.

- **Remote debugging limitations**: Currently, you can't remotely debug WASM modules running in Azure Linux 3.0 due to incompatible LLDB versions.

### Recovery procedures

**VS Code extension reset**: If the VS Code extension behaves unexpectedly, try uninstalling and reinstalling it, then restart VS Code.

## Related content

- [Understand WebAssembly (WASM) modules and graph definitions for data flow graphs](concepts-wasm-modules.md)
- [Configure graph definitions](howto-configure-wasm-graph-definitions.md)
- [Deploy graph definitions](howto-deploy-wasm-graph-definitions.md)
- [ONNX inference in WASM modules](howto-wasm-onnx-inference.md)
- [Use WASM in dataflow graphs](../connect-to-cloud/howto-dataflow-graph-wasm.md)
