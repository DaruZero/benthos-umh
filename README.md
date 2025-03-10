# benthos-umh

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache2.0-blue.svg)](https://www.apache.org/licenses/LICENSE-2.0)
[![GitHub Actions](https://github.com/united-manufacturing-hub/benthos-umh/workflows/main/badge.svg)](https://github.com/united-manufacturing-hub/benthos-umh/actions)

Welcome to the benthos-umh repository! This is a version of benthos maintained by the United Manufacturing Hub (UMH) to provide seamless OPC-UA integration with the [Unified Namespace](https://learn.umh.app/lesson/introduction-into-it-ot-unified-namespace/) (MQTT/Kafka). Our goal is to enhance the integration of IT and OT tools for engineers while avoiding vendor lock-in and streamlining data management processes.

## Description

`benthos-umh` is a Docker container designed to facilitate seamless OPC-UA integration with the Unified Namespace (MQTT/Kafka). It is part of the United Manufacturing Hub project and offers the following features:

- Simple deployment in Docker, docker-compose, and Kubernetes
- Connects to an OPC-UA server, browses selected nodes, and forwards all sub-nodes in 1-second intervals
- Supports a wide range of outputs, from the Unified Namespace (MQTT and Kafka) to HTTP, AMQP, Redis, NATS, SQL, MongoDB, Cassandra, or AWS S3. Check out the official [benthos output library](https://benthos.dev/docs/components/outputs/about)
- Fully customizable messages using the benthos processor library: implement Report-by-Exception (RBE) / message deduplication, modify payloads and add timestamps using bloblang, apply protobuf (and therefore SparkplugB), and explore many more options
- Integrates with modern IT landscape, providing metrics, logging, tracing, versionable configuration, and more
- Entirely open-source (Apache 2.0) and free-to-use

We encourage you to try out `benthos-umh` and explore the broader [United Manufacturing Hub](https://www.umh.app) project for a comprehensive solution to your industrial data integration needs.

## Usage

### Standalone

To use benthos-umh in standalone mode with Docker, follow the instructions in the main article provided.

1. Create a new file called benthos.yaml with the provided content

    ```yaml
    ---
    input:
      opcua:
        endpoint: 'opc.tcp://localhost:46010'
        nodeIDs: ['ns=2;s=IoTSensors']

    pipeline:
      processors:
        - bloblang: |
            root = {
              meta("opcua_path"): this,
              "timestamp_unix": timestamp_unix()
            }

    output:
      mqtt:
        urls:
          - 'localhost:1883'
        topic: 'ia/raw/opcuasimulator/${! meta("opcua_path") }'
        client_id: 'benthos-umh'
    ```

2. Execute the docker run command to start a new benthos-umh container
    `docker run --rm --network="host" -v '<absolute path to your file>/benthos.yaml:/benthos.yaml' ghcr.io/united-manufacturing-hub/benthos-umh:latest`

### With the United Manufacturing Hub (Kubernetes & Kafka)

To deploy benthos-umh with the United Manufacturing Hub and its OPC-UA simulator, use the provided Kubernetes manifests in UMHLens/OpenLens.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: benthos-1-config
  namespace: united-manufacturing-hub
  labels:
    app: benthos-1
data:
  benthos.yaml: |-
    input:
      umh_input_opcuasimulator: {}
    pipeline:
      processors:
        - bloblang: |
            root = {
              meta("opcua_path"): this,
              "timestamp_unix": timestamp_unix()
            }
    output:
      umh_output:
        topic: 'ia.raw.${! meta("opcua_path") }'
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: benthos-1-deployment
  namespace: united-manufacturing-hub
  labels:
    app: benthos-1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: benthos-1
  template:
    metadata:
      labels:
        app: benthos-1
    spec:
      containers:
        - name: benthos-1
          image: "ghcr.io/united-manufacturing-hub/benthos-umh:latest"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 4195
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /ping
              port: http
          readinessProbe:
            httpGet:
              path: /ready
              port: http
          volumeMounts:
            - name: config
              mountPath: "/benthos.yaml"
              subPath: "benthos.yaml"
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: benthos-1-config
```

### Capabilities
The plugin is designed to browse and subscribe to all child nodes within a folder for each configured NodeID, provided that the NodeID represents a folder. It features a recursion depth of up to 10 levels, enabling thorough exploration of nested folder structures. The browsing specifically targets nodes organized under the OPC UA 'Organizes' relationship type, intentionally excluding nodes under 'HasProperty' and 'HasComponent' relationships. Additionally, the plugin does not browse Objects represented by red, blue, or green cube icons in UAExpert.

Subscriptions are selectively managed, with tags having a DataType of null being excluded from subscription. Also, by default, the plugin does not subscribe to the properties of a tag, such as minimum and maximum values.

#### Datatypes
The plugin has been rigorously tested with an array of datatypes, both as single values and as arrays. The following datatypes have been verified for compatibility:

- `Boolean`
- `Byte`
- `DateTime`
- `Double`
- `Enumeration`
- `ExpandedNodeId`
- `Float`
- `Guid`
- `Int16`
- `Int32`
- `Int64`
- `Integer`
- `LocalizedText`
- `NodeId`
- `Number`
- `QualifiedName`
- `SByte`
- `StatusCode`
- `String`
- `UInt16`
- `UInt32`
- `UInt64`
- `UInteger`
- `ByteArray`
- `ByteString`
- `Duration`
- `LocaleId`
- `UtcTime`
- `Variant`
- `XmlElement`

There are specific datatypes which are currently not supported by the plugin and attempting to use them will result in errors. These include:

- Two-dimensional arrays
- UA Extension Objects
- Variant arrays (Arrays with multiple different datatypes)


### Authentication and Security

In benthos-umh, security and authentication are designed to be as robust as possible while maintaining flexibility. The software automates the process of selecting the highest level of security offered by an OPC-UA server for the selected Authentication Method, but the user can specify their own Security Policy / Security Mode if they want (see further below at Configuration options)

#### Supported Authentication Methods

- **Anonymous**: No extra information is needed. The connection uses the highest security level available for anonymous connections.
- **Username and Password**: Specify the username and password in the configuration. The client opts for the highest security level that supports these credentials.
- **Certificate (Future Release)**: Certificate-based authentication is planned for future releases.

### Configuration Options

The following options can be specified in the `benthos.yaml` configuration file:

```yaml
input:
  opcua:
    endpoint: 'opc.tcp://localhost:46010'
    nodeIDs: ['ns=2;s=IoTSensors']
    username: 'your-username'  # optional (default: unset)
    password: 'your-password'  # optional (default: unset)
    insecure: false | true # optional (default: false)
    securityMode: None | Sign | SignAndEncrypt # optional (default: unset)
    securityPolicy: None | Basic256Sha256 | Aes256Sha256RsaPss | Aes128Sha256RsaOaep # optional (default: unset)
    subscribeEnabled: false | true # optional (default: false)
```

#### Endpoint

You can specify the endpoint in the configuration file. Node endpoints are automatically discovered and selected based on the authentication method.

```yaml
input:
  opcua:
    endpoint: 'opc.tcp://localhost:46010'
    nodeIDs: ['ns=2;s=IoTSensors']
```

#### Node IDs

You can specify the node IDs in the configuration file (currently only namespaced node IDs are supported):

```yaml
input:
  opcua:
    endpoint: 'opc.tcp://localhost:46010'
    nodeIDs: ['ns=2;s=IoTSensors']
```

#### Username and Password

If you want to use username and password authentication, you can specify them in the configuration file:

```yaml
input:
  opcua:
    endpoint: 'opc.tcp://localhost:46010'
    nodeIDs: ['ns=2;s=IoTSensors']
    username: 'your-username'
    password: 'your-password'
```

#### Security Mode and Security Policy

Security Mode: This defines the level of security applied to the messages. The options are:
- None: No security is applied; messages are neither signed nor encrypted.
- Sign: Messages are signed for integrity and authenticity but not encrypted.
- SignAndEncrypt: Provides the highest security level where messages are both signed and encrypted.

Security Policy: Specifies the set of cryptographic algorithms used for securing messages. This includes algorithms for encryption, decryption, and signing of messages. Some common policies include Basic256Sha256, Aes256Sha256RsaPss, and Aes128Sha256RsaOaep.

While the security mode and policy are automatically selected based on the endpoint and authentication method, you have the option to override this by specifying them in the configuration file:

```yaml
input:
  opcua:
    endpoint: 'opc.tcp://localhost:46010'
    nodeIDs: ['ns=2;s=IoTSensors']
    securityMode: SignAndEncrypt
    securityPolicy: Basic256Sha256
```

#### Insecure Mode

Setting this to true will overwrite any configured securityMode and securityPolicy!

If the most secure endpoint selected by benthos-umh is not working or the server's security implementation is lacking, you can bypass encryption by setting `insecure: true`. This will use the Security Mode "None".

```yaml
input:
  opcua:
    endpoint: 'opc.tcp://localhost:46010'
    nodeIDs: ['ns=2;s=IoTSensors']
    insecure: true
```

#### Pull and Subscribe Methods

Benthus-umh supports two modes of operation: pull and subscribe. In pull mode, it pulls all nodes every second, regardless of changes. In subscribe mode, it only sends data when there's a change in value, reducing unnecessary data transfer.

| Method | Advantages | Disadvantages |
| --- | --- | --- |
| Pull | - Provides real-time data visibility, e.g., in MQTT Explorer. <br> - Clearly differentiates between 'no data received' and 'value did not change' scenarios, which can be crucial for documentation and proving the OPC-UA client's activity. | - Results in higher data throughput as it pulls all nodes every second, regardless of changes. |
| Subscribe | - Data is sent only when there's a change in value, reducing unnecessary data transfer. | - Less visibility into real-time data status, and it's harder to differentiate between no data and unchanged values. |

```yaml
input:
  opcua:
    endpoint: 'opc.tcp://localhost:46010'
    nodeIDs: ['ns=2;s=IoTSensors']
    subscribeEnabled: true
```

## Testing

We execute automated tests and verify that benthos-umh works:

- (WAGO PFC100, 750-8101) Connect Anonymously
- (WAGO PFC100, 750-8101) Connect Username / Password
- (WAGO PFC100, 750-8101) Connect and get one float number

These tests are executed with a local github runner called "hercules", which is connected to a isolated testing network.

## Development

### Quickstart

Follow the steps below to set up your development environment and run tests:

```
git clone https://github.com/united-manufacturing-hub/benthos-umh.git
cd serverless-stack
nvm install
npm install
sudo apt-get install zip
echo 'deb [trusted=yes] https://repo.goreleaser.com/apt/ /' | sudo tee /etc/apt/sources.list.d/goreleaser.list
sudo apt update
sudo apt install goreleaser
make
npm test
```

### Additional Checks and Commands

#### Gitpod and Tailscale

By default when opening the repo in Gitpod, everything that you need should start automatically. If you want to connect to our local PLCs in our office, you can use tailscale, which you will be prompted to install.
See also: <https://www.gitpod.io/docs/integrations/tailscale>

#### For Go Code

1. **Linting**: Run `make lint` to check for linting errors. If any are found, you can automatically fix them by running `make format`.

2. **Unit Tests**: Run `make test` to execute all Go unit tests.

#### For Other Code Types (Including Config Files)

1. **Benthos Tests**: Use `npm run test` to run all Benthos tests for configuration files. Note: We currently do not have these tests. [Learn more](https://www.benthos.dev/docs/configuration/unit_testing/).

2. **Linting**: Run `npm run lint` to check all files, including YAML files, for linting errors. To automatically fix these errors, run `npm run format`.

## License

All source code is distributed under the APACHE LICENSE, VERSION 2.0. See LICENSE for more information.

## Contact

Feel free to provide us feedback on our [Discord channel](https://discord.gg/F9mqkZnm9d).

For more information about the United Manufacturing Hub, visit [UMH Systems GmbH](https://www.umh.app). If you haven't worked with the United Manufacturing Hub before, [give it a try](https://umh.docs.umh.app/docs/getstarted/installation/)! Setting it up takes only a matter of minutes.
