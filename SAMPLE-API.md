# SchemaRegistry configuration

Configuration file is located at conf/registry-dev.yaml. By default it uses inmemory storage manager. It can be changed to use MySQL as storage by providing configuration like below. You should configure respective dataSourceUrl

```
# MySQL based jdbc provider configuration is:
storageProviderConfiguration:
  providerClass: "com.hortonworks.iotas.storage.impl.jdbc.JdbcStorageManager"
  properties:
    db.type: "mysql"
    queryTimeoutInSecs: 30
    db.properties:
      dataSourceClassName: "com.mysql.jdbc.jdbc2.optional.MysqlDataSource"
      dataSource.url: "jdbc:mysql://localhost:3307/schema_registry"

```

Before starting the SchemaRegistry server, below script should be run with configured database names.

```mysql
CREATE DATABASE IF NOT EXISTS schema_registry;
USE schema_registry;

CREATE TABLE IF NOT EXISTS schema_info (
  id            BIGINT AUTO_INCREMENT NOT NULL,
  type          VARCHAR(256)          NOT NULL,
  schemaGroup   VARCHAR(256)          NOT NULL,
  name          VARCHAR(256)          NOT NULL,
  compatibility VARCHAR(256)          NOT NULL,
  description   TEXT,
  timestamp     BIGINT                NOT NULL,
  PRIMARY KEY (type, schemaGroup, name),
  UNIQUE KEY (id),
  UNIQUE KEY `UK_TYPE_GROUP_NAME` (type, schemaGroup, name)
);

CREATE TABLE IF NOT EXISTS schema_version_info (
  id               BIGINT AUTO_INCREMENT NOT NULL,
  description      TEXT,
  schemaText       TEXT                  NOT NULL,
  fingerprint      TEXT                  NOT NULL,
  version          INT                   NOT NULL,
  schemaMetadataId BIGINT                NOT NULL,
  timestamp        BIGINT                NOT NULL,
  type             VARCHAR(256)          NOT NULL,
  schemaGroup      VARCHAR(256)          NOT NULL,
  name             VARCHAR(256)          NOT NULL,
  UNIQUE KEY (id),
  UNIQUE KEY `UK_METADATA_ID_VERSION_FK` (schemaMetadataId, version),
  PRIMARY KEY (version, type, schemaGroup, name),
  FOREIGN KEY (schemaMetadataId, type, schemaGroup, name) REFERENCES schema_info (id, type, schemaGroup, name)
);

CREATE TABLE IF NOT EXISTS schema_field_info (
  id               BIGINT AUTO_INCREMENT NOT NULL,
  schemaInstanceId BIGINT                NOT NULL,
  timestamp        BIGINT                NOT NULL,
  name             VARCHAR(256)          NOT NULL,
  fieldNamespace   VARCHAR(256),
  type             VARCHAR(256)          NOT NULL,
  PRIMARY KEY (id),
  FOREIGN KEY (schemaInstanceId) REFERENCES schema_version_info (id)
);

CREATE TABLE IF NOT EXISTS schema_serdes_info (
  id           BIGINT AUTO_INCREMENT NOT NULL,
  description  TEXT,
  name         TEXT                  NOT NULL,
  fileId       TEXT                  NOT NULL,
  className    TEXT                  NOT NULL,
  isSerializer BOOLEAN               NOT NULL,
  timestamp    BIGINT                NOT NULL,
  PRIMARY KEY (id)
);

CREATE TABLE IF NOT EXISTS schema_serdes_mapping (
  schemaMetadataId BIGINT NOT NULL,
  serDesId         BIGINT NOT NULL,

  UNIQUE KEY `UK_IDS` (schemaMetadataId, serdesId)
);

```

# API examples

Below set of code snippets explain how SchemaRegistryClient can be used for
 - registering new versions of schemas
 - fetching registered schema versions
 - registering serializers/deserializers
 - fetching serializer/deserializer for a given schema
 
## Using schema related APIs
 
```java

Map<String, Object> config = new HashMap<>();
config.put(SchemaRegistryClient.Options.SCHEMA_REGISTRY_URL, "http://localhost:8080/api/v1");
config.put(SchemaRegistryClient.Options.CLASSLOADER_CACHE_SIZE, 10);
config.put(SchemaRegistryClient.Options.CLASSLOADER_CACHE_EXPIRY_INTERVAL_MILLISECS, 5000L);

schemaRegistryClient = new SchemaRegistryClient(config);

String schemaFileName = "/device.avsc";
String schema1 = getSchema(schemaFileName);
SchemaInfo schemaInfo = createSchemaInfo("com.hwx.schemas.sample-"+System.currentTimeMillis());
SchemaKey schemaKey = schemaInfo.getSchemaKey();

// registering a new schema
Integer v1 = schemaRegistryClient.addSchemaVersion(schemaInfo, new SchemaVersion(schema1, "Initial version of the schema"));
LOG.info("Registered schema [{}] and returned version [{}]", schema1, v1);

// adding a new version of the schema
String schema2 = getSchema("/device-next.avsc");
SchemaVersion schemaInfo2 = new SchemaVersion(schema2, "second version");
Integer v2 = schemaRegistryClient.addSchemaVersion(schemaKey, schemaInfo2);
LOG.info("Registered schema [{}] and returned version [{}]", schema2, v2);

//adding same schema returns the earlier registered version
Integer version = schemaRegistryClient.addSchemaVersion(schemaKey, schemaInfo2);
LOG.info("");

// get a specific version of the schema
SchemaVersionInfo schemaVersionInfo = schemaRegistryClient.getSchemaVersionInfo(new SchemaVersionKey(schemaKey, v2));

// get latest version of the schema
SchemaVersionInfo latest = schemaRegistryClient.getLatestSchemaVersionInfo(schemaKey);
LOG.info("Latest schema with schema key [{}] is : [{}]", schemaKey, latest);

// get all versions of the schema
Collection<SchemaVersionInfo> allVersions = schemaRegistryClient.getAllVersions(schemaKey);
LOG.info("All versions of schema key [{}] is : [{}]", schemaKey, allVersions);

// finding schemas containing a specific field
SchemaFieldQuery md5FieldQuery = new SchemaFieldQuery.Builder().name("md5").build();
Collection<SchemaVersionKey> md5SchemaVersionKeys = schemaRegistryClient.findSchemasByFields(md5FieldQuery);
LOG.info("Schemas containing field query [{}] : [{}]", md5FieldQuery, md5SchemaVersionKeys);

SchemaFieldQuery txidFieldQuery = new SchemaFieldQuery.Builder().name("txid").build();
Collection<SchemaVersionKey> txidSchemaVersionKeys = schemaRegistryClient.findSchemasByFields(txidFieldQuery);
LOG.info("Schemas containing field query [{}] : [{}]", txidFieldQuery, txidSchemaVersionKeys);

```

### Using serializer and deserializer related APIs
Registering serializer and deserializer is donw with the below steps
 - Upload jar file which contains serializer and deserializer classes and its dependencies
 - Register serializer/deserializer
 - Map serializer/deserializer with a registered schema.
 - Fetch Serializer/Deserializer and use it to marshal/unmarshal payloads.
 
##### Uploading jar file

```java

String serdesJarName = "/samples-serdes.jar";
InputStream serdesJarInputStream = SampleSchemaRegistryApplication.class.getResourceAsStream(serdesJarName);
if (serdesJarInputStream == null) {
    throw new RuntimeException("Jar " + serdesJarName + " could not be loaded");
}

String fileId = schemaRegistryClient.uploadFile(serdesJarInputStream);

```

##### Register serializer and deserializer

```java

String simpleSerializerClassName = "com.hortonworks.schemaregistry.samples.serdes.SimpleSerializer";
SerDesInfo serializerInfo = new SerDesInfo.Builder()
                                            .name("simple-serializer")
                                            .description("simple serializer")
                                            .fileId(fileId)
                                            .className(simpleSerializerClassName)
                                            .buildSerializerInfo();
Long serializerId = schemaRegistryClient.addSerializer(serializerInfo);

String simpleDeserializerClassName = "com.hortonworks.schemaregistry.samples.serdes.SimpleDeserializer";
SerDesInfo deserializerInfo = new SerDesInfo.Builder()
        .name("simple-deserializer")
        .description("simple deserializer")
        .fileId(fileId)
        .className(simpleDeserializerClassName)
        .buildDeserializerInfo();
Long deserializerId = schemaRegistryClient.addDeserializer(deserializerInfo);


```

##### Map serializer/deserializer with a schema

```java

// map serializer and deserializer with schemakey
// for each schema, one serializer/deserializer is sufficient unless someone want to maintain multiple implementations of serializers/deserializers
SchemaKey schemaKey = schemaInfo.getSchemaKey();
schemaRegistryClient.mapSchemaWithSerDes(schemaKey, serializerId);
schemaRegistryClient.mapSchemaWithSerDes(schemaKey, deserializerId);

```

##### Marshal and unmarshal using the registered serializer and deserializer for a schema

```java
SnapshotSerializer<Object, byte[], SchemaInfo> snapshotSerializer = getSnapshotSerializer(schemaKey);
String payload = "Random text: " + new Random().nextLong();
byte[] serializedBytes = snapshotSerializer.serialize(payload, schemaInfo);

SnapshotDeserializer<byte[], Object, SchemaInfo, SchemaInfo> snapshotdeserializer = getSnapshotDeserializer(schemaKey);
Object deserializedObject = snapshotdeserializer.deserialize(serializedBytes, schemaInfo, schemaInfo);

LOG.info("Given payload and deserialized object are equal: "+ payload.equals(deserializedObject));

```

## Using inbuilt Kafka Avro serializer and deserializer

Below Serializer and Deserializer can be used for avro records as respective Kafka avro serializer and deserializer respectively.
`com.hortonworks.registries.schemaregistry.avro.kafka.KafkaAvroSerializer`
`com.hortonworks.registries.schemaregistry.avro.kafka.KafkaAvroDeserializer`

Following properties can be configured for producer/consumer

```java
// producer configuration
props.put(SchemaRegistryClient.Options.SCHEMA_REGISTRY_URL, schemaRegistryUrl);
props.put(SchemaRegistryClient.Options.SCHEMA_CACHE_SIZE, 1000);
props.put(SchemaRegistryClient.Options.SCHEMA_CACHE_EXPIRY_INTERVAL_MILLISECS, 60*60*1000L);
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, KafkaAvroDeserializer.class.getName());

// consumer configuration
props.put(SchemaRegistryClient.Options.SCHEMA_REGISTRY_URL, schemaRegistryUrl);
props.put(SchemaRegistryClient.Options.SCHEMA_CACHE_SIZE, 1000);
props.put(SchemaRegistryClient.Options.SCHEMA_CACHE_EXPIRY_INTERVAL_MILLISECS, 60*60*1000L);
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, KafkaAvroSerializer.class.getName());

```