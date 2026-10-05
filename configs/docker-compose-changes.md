# Changes made to Shuffle's `docker-compose.yml`

Shuffle's bundled OpenSearch was defaulting to settings meant for a dedicated host. Two edits were needed
to run it next to the Wazuh indexer on one VM. Only the `opensearch` service was changed.

## 1. Published port (avoids the clash with Wazuh's indexer on 9200)

```yaml
    ports:
      - 9201:9200      # was 9200:9200
```

The container still listens on 9200 internally; only the host port changed.

## 2. JVM heap (avoids out-of-memory kills)

```yaml
    environment:
      - "OPENSEARCH_JAVA_OPTS=-Xms1024m -Xmx1024m"   # default was -Xms3072m -Xmx3072m
```

The default is half of system RAM, which assumes Shuffle has the machine to itself.
