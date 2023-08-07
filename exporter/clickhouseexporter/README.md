This branch is created to change the type of attributes in ClickHouse exporter schema from Map[key]value to Array[keys] and Array[values]. The ultimate purpose is to do benchmarking to see which type choice performs better.

I just changed the types in the implementation, not the tests. So some tests is probably failing. However, I checked the change locally with a small trace generating experiment. The change worked.

I didn't change the hardcoded schema either. So to test this change, you would need to manually create the table in ClickHouse server side yourself:
```
CREATE TABLE IF NOT EXISTS otel_traces (
     Timestamp DateTime64(9) CODEC(Delta, ZSTD(1)),
     TraceId String CODEC(ZSTD(1)),
     SpanId String CODEC(ZSTD(1)),
     ParentSpanId String CODEC(ZSTD(1)),
     TraceState String CODEC(ZSTD(1)),
     SpanName LowCardinality(String) CODEC(ZSTD(1)),
     SpanKind LowCardinality(String) CODEC(ZSTD(1)),
     ServiceName LowCardinality(String) CODEC(ZSTD(1)),
     ResourceAttributes Nested
     (
        keys LowCardinality(String),
        values String
     ) CODEC (ZSTD(1)),
     ScopeName String CODEC(ZSTD(1)),
     ScopeVersion String CODEC(ZSTD(1)),
     SpanAttributes Nested
     (
        keys LowCardinality(String),
        values String
     ) CODEC (ZSTD(1)),
     Duration UInt64 CODEC(ZSTD(1)),
     StatusCode LowCardinality(String) CODEC(ZSTD(1)),
     StatusMessage String CODEC(ZSTD(1)),
     Events Nested (
         Timestamp DateTime64(9),
         Name LowCardinality(String),
         Attributes Map(LowCardinality(String), String)
     ) CODEC(ZSTD(1)),
     Links Nested (
         TraceId String,
         SpanId String,
         TraceState String,
         Attributes Map(LowCardinality(String), String)
     ) CODEC(ZSTD(1)),
     INDEX idx_trace_id TraceId TYPE bloom_filter(0.001) GRANULARITY 1,
     INDEX idx_res_attr_key ResourceAttributes.keys TYPE bloom_filter(0.01) GRANULARITY 1,
     INDEX idx_res_attr_value ResourceAttributes.values TYPE bloom_filter(0.01) GRANULARITY 1,
     INDEX idx_span_attr_key SpanAttributes.keys TYPE bloom_filter(0.01) GRANULARITY 1,
     INDEX idx_span_attr_value SpanAttributes.values TYPE bloom_filter(0.01) GRANULARITY 1,
     INDEX idx_duration Duration TYPE minmax GRANULARITY 1
) ENGINE MergeTree()

PARTITION BY toDate(Timestamp)
ORDER BY (ServiceName, SpanName, SpanAttributes.keys, SpanAttributes.values, toUnixTimestamp(Timestamp), Duration, TraceId)
SETTINGS index_granularity=8192, ttl_only_drop_parts = 1;
```

To build this change locally:
`make docker-otelcontribcol`

To run the Docker image of this change:
`docker run --network host --name otel -p 145.40.99.239:4317:4317 -p 145.40.99.239:8888:8888 -v $(pwd)/config.yaml:/etc/otel/config.yaml otelcontribcol`
