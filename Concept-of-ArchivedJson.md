## ArchivedJson example
```json
{
    "archive": [
        {
            "path": "/jobs/overview",
			"json": "..." },
        {
            "path": "/jobs/da12f990aba5bcdff710e96c5a409123/config",
			"json": "..."
		},
		{
            "path": "/jobs/da12f990aba5bcdff710e96c5a409123/checkpoints",
			"json": "..."
		}
	]
}
```

## TaskManager vertex model of data structure
`org.apache.flink.runtime.rest.messages.JobVertexTaskManagersInfo.java`
```java
        @JsonCreator
        public TaskManagersInfo(
                @JsonProperty(TASK_MANAGERS_FIELD_HOST) String host,
                @JsonProperty(TASK_MANAGERS_FIELD_STATUS) ExecutionState status,
                @JsonProperty(TASK_MANAGERS_FIELD_START_TIME) long startTime,
                @JsonProperty(TASK_MANAGERS_FIELD_END_TIME) long endTime,
                @JsonProperty(TASK_MANAGERS_FIELD_DURATION) long duration,
                @JsonProperty(TASK_MANAGERS_FIELD_METRICS) IOMetricsInfo metrics,
                @JsonProperty(TASK_MANAGERS_FIELD_STATUS_COUNTS)
                        Map<ExecutionState, Integer> statusCounts,
                @JsonProperty(TASK_MANAGERS_FIELD_TASKMANAGER_ID) String taskmanagerId,
                @JsonProperty(TASK_MANAGERS_FIELD_AGGREGATED)
                        AggregatedTaskDetailsInfo aggregated) {
            this.host = checkNotNull(host);
            this.status = checkNotNull(status);
            this.startTime = startTime;
            this.endTime = endTime;
            this.duration = duration;
            this.metrics = checkNotNull(metrics);
            this.statusCounts = checkNotNull(statusCounts);
            this.taskmanagerId = taskmanagerId;
            this.aggregated = aggregated;
        }
```

Sample:
```json
{
    "id": "cbc357ccb763df2852fee8c4fc7d55f2",
    "name": "Source: Custom Source -> Process -> Map",
    "now": 1731465012643,
    "taskmanagers": [
        {
            "host": "kka136119:40079",
            "status": "FAILED",
            "taskmanager-id": "container_e87_1724243239726_2160_01_000003"
        },
        {
            "host": "kka129134:40853",
            "status": "CANCELED",
            "taskmanager-id": "container_e87_1724243239726_2160_01_000002"
         }
        ...
    ]
}
```
The vertex id in this sample is `"cbc357ccb763df2852fee8c4fc7d55f2"`, we can collect metrics from the given json.
