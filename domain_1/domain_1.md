## CloudWatch

### CloudWatch Metrics

Every AWS service publishes metrics to CloudWatch automatically. These are called standard metrics and they're free. The granularity is 1 minute by default for most services, 5 minutes for some older ones like basic EC2 monitoring.

High-resolution custom metrics can go down to 1-second granularity. You publish them via the PutMetricData API. The cost is $0.30 per custom metric per month, plus $0.01 per 1,000 API calls.

The metric data model is: namespace → metric name → dimensions. Namespace is just a grouping string, like AWS/EC2 or your own custom namespace. Dimensions are key-value pairs that further identify the metric — for EC2, the InstanceId dimension identifies which instance the CPUUtilization metric belongs to. You can have up to 30 dimensions per metric.

Retention schedule matters for the exam. Data points with a period of less than 60 seconds are retained for 3 hours. One-minute data points are retained for 15 days. Five-minute data points for 63 days. One-hour data points for 15 months. Data is automatically aggregated upward as it ages — your 1-minute data rolls into 5-minute data, then into 1-hour data.

Statistics you can compute: Sum, Average, Minimum, Maximum, SampleCount, and percentiles (p50, p90, p99, etc.). Percentiles are important for latency metrics because an average hides tail latency. A p99 latency of 5 seconds means 1% of your requests are taking at least 5 seconds — the average might look fine.

Metric Math lets you combine and transform metrics using expressions. You can write RATE(m1) to compute a rate of change, SUM([m1,m2,m3]) to add metrics together, or m1/m2*100 to compute a percentage. Metric Math results can be used in alarms just like raw metrics. A common pattern is computing error rate as errors/requests and alarming on that ratio rather than raw error counts.

GetMetricStatistics is the older API. GetMetricData is the newer one — it supports multiple metrics and Metric Math in a single call, and it's what the console uses. Know that GetMetricData is preferred for programmatic access.

The EC2 metrics you get by default are: CPUUtilization, NetworkIn, NetworkOut, NetworkPacketsIn, NetworkPacketsOut, DiskReadOps, DiskWriteOps, DiskReadBytes, DiskWriteBytes, StatusCheckFailed (which combines instance status and system status checks), StatusCheckFailed_Instance, and StatusCheckFailed_System.

What you do NOT get by default: memory utilization, disk space utilization (the percentage of disk used, not just I/O operations), swap usage, and process-level data. These require the CloudWatch Agent because they're inside the guest OS where the hypervisor can't see.

### CloudWatch Agent

The CloudWatch Agent is software you install on EC2 instances (and on-premises servers). It collects two categories of data: OS-level metrics that the hypervisor can't see, and log files from the local filesystem.

For metrics, the agent can collect: mem_used_percent, disk_used_percent, swap_used_percent, processes (running, sleeping, dead), netstat (TCP connections by state), and more. You define exactly what to collect in the agent's JSON configuration file.

The configuration file is the most operationally important piece. You can write it manually, generate it with the configuration wizard on a reference instance, or store it in SSM Parameter Store and distribute it fleet-wide. Storing it in Parameter Store is the exam-preferred pattern because you can update the config in one place and push it to all instances via SSM.

The agent supports two collection protocols beyond its native config: StatsD and collectd. StatsD lets application code push custom metrics to the agent over UDP on port 8125. collectd is a Unix daemon that the agent can read from. These exist so application teams can instrument their code without needing to call the CloudWatch API directly.

For logs, you define log files or Windows Event Log sources in the agent config. Each log source maps to a Log Group and Log Stream in CloudWatch. You can set the log stream name to the instance ID, the hostname, or a custom string.

The agent requires an IAM instance profile with permissions to call cloudwatch:PutMetricData for metrics and logs:CreateLogGroup, logs:CreateLogStream, logs:PutLogEvents, and logs:DescribeLogStreams for logs. It also needs ssm:GetParameter if it's fetching its config from Parameter Store.

### CloudWatch Logs

Log Groups are the top-level container. You set retention on the Log Group — anywhere from 1 day to 10 years, or never expire. The default is never expire, which accumulates cost indefinitely. The exam tests this: if you need to control cost, set a retention policy.

Log Streams are sequences of log events within a Log Group. One stream per instance per log file is a common pattern, but the structure is flexible.

Log Insights is the query engine. It uses its own query language — not SQL, but similar. Key functions: fields (select specific fields), filter (where clause), stats (aggregations like count(), avg(), sum()), sort, limit, and parse (extract fields from unstructured text using regex or glob patterns). Results are limited to 10,000 rows and are not stored — each query re-scans the data. Queries can span multiple Log Groups.

Metric Filters convert log data into CloudWatch metrics. You define a filter pattern that matches log events, and each match increments (or sets) a metric value. For example, filter for the string "ERROR" in your application log and publish a count to a custom metric, which you then alarm on. Metric filters only process new data after the filter is created — they don't backfill.

Subscription Filters stream log data in near-real-time to destinations: Lambda (for processing/transformation), Kinesis Data Streams, Kinesis Data Firehose (for delivery to S3, Redshift, OpenSearch), or OpenSearch Service directly. You can have up to two subscription filters per Log Group. Cross-account log streaming requires a subscription filter pointing to a cross-account Kinesis stream with an appropriate resource policy.

Log export to S3 is a batch operation — you trigger it via the console or CLI. It is not real-time. If you need real-time delivery to S3, use a subscription filter to Kinesis Data Firehose, which buffers and delivers to S3.

Contributor Insights analyzes log data to identify the top contributors to a pattern — for example, the top 10 IP addresses generating 4xx errors. You create a rule specifying the Log Group and the field paths to analyze. It runs continuously and updates every minute. The use case is finding which specific resources (IPs, users, endpoints) are causing disproportionate load or errors.

Logs Insights vs Contributor Insights: use Logs Insights for ad-hoc investigation — you want to explore what happened during an incident. Use Contributor Insights for ongoing operational visibility — you want a continuously updated view of your top contributors.

### CloudWatch Alarms

An alarm monitors a single metric (or Metric Math expression) and transitions between three states: OK, ALARM, and INSUFFICIENT_DATA. INSUFFICIENT_DATA means there isn't enough data to evaluate the condition — this happens when a metric hasn't published any data points in the evaluation period.

The evaluation logic: you define a threshold, a comparison operator, an evaluation period (how many minutes per datapoint), and a datapoints-to-alarm value. The datapoints-to-alarm pattern (M out of N) is critical. If you set 3 out of 5, the alarm fires only when at least 3 of the last 5 evaluation periods breach the threshold. This prevents alarms from triggering on transient spikes. The exam tests this: "the alarm is firing too often on brief CPU spikes" — the fix is increasing the M-of-N datapoints-to-alarm setting, not necessarily changing the threshold.

Missing data treatment is a separate setting: treat missing data as breaching (conservative — assumes something is wrong if data stops), not breaching (lenient — assumes the metric is fine), ignore (maintains current state), or missing (transitions to INSUFFICIENT_DATA). The right choice depends on context: for an EC2 instance that should always be sending metrics, treating missing data as breaching is appropriate because no data likely means the instance is down.

Alarm actions: SNS notification (most common — fan out to email, HTTP endpoint, Lambda, SQS from there), EC2 instance action (stop, terminate, reboot, recover — for self-healing), and Auto Scaling policy. You can have different actions for ALARM and OK transitions.

Composite Alarms evaluate a boolean expression across multiple alarms. Example: ALARM("HighCPU") AND ALARM("HighLatency"). The composite alarm only fires when both underlying alarms are in ALARM state simultaneously. This is the mechanism for reducing alert noise — individual metrics might spike independently for innocuous reasons, but both spiking at once indicates a real problem.

You cannot set a Composite Alarm action directly on an Auto Scaling policy — composite alarm actions are limited to SNS and Systems Manager OpsCenter. If you need to scale based on a composite condition, you'd trigger a Lambda from SNS which then calls the scaling API.

Period-based alarms can use high-resolution metrics with a period as low as 10 or 30 seconds. Standard alarms have a minimum period of 60 seconds. This matters when you need faster detection of transient issues — a 10-second alarm fires much faster than a 60-second one, but costs more.

### CloudWatch Anomaly Detection

Anomaly Detection trains an ML model on the history of a metric and produces an expected value band — a range above and below the predicted value. The alarm fires when the metric exits the band.

The model automatically accounts for time-of-day patterns, day-of-week patterns, and seasonal trends. You don't need to configure any of this. If your request count is always higher on weekdays and lower on weekends, the band will be wider on weekdays and narrower on weekends.

You can exclude time periods from training — for example, exclude the window of a known maintenance event or traffic spike that doesn't represent normal behavior. This prevents the model from learning bad patterns.

You can also adjust the anomaly threshold by setting the number of standard deviations the band should span. A wider band is more tolerant; a narrower band is more sensitive.

When to use Anomaly Detection versus static thresholds: use static thresholds when the acceptable range is fixed and well-understood (disk at 90% is always bad regardless of day or time). Use Anomaly Detection when the metric has natural variation and you care about deviations from the pattern rather than absolute values.

## CloudTrail

CloudTrail records API calls made to AWS — who called what, when, from what IP, and with what parameters. Every AWS action is an API call, so CloudTrail is a complete audit trail of everything that happened in your account.
Event types, in detail:

Management events are control-plane operations: creating resources, modifying IAM policies, changing security groups, deleting buckets. These are on by default. The 90-day event history in the console is free and covers management events only. If you need longer retention or delivery to S3 for analysis, you create a trail.

Data events are data-plane operations: S3 GetObject, PutObject, DeleteObject; Lambda Invoke; DynamoDB GetItem, PutItem. These are off by default because the volume is enormous — every S3 read from a busy bucket would generate an event. You enable them selectively using advanced event selectors that let you filter to specific buckets, prefixes, or object sizes.

Network Activity events (newer) capture VPC endpoint traffic — API calls made through VPC endpoints. Relevant for auditing private API traffic.

Insight events are generated when CloudTrail detects anomalies in management event activity. It establishes a baseline of normal API call rates and error rates for each API, then fires an Insight event when a significant deviation occurs. For example, if your account normally calls AuthorizeSecurityGroupIngress five times per day and suddenly calls it 500 times in an hour, an Insight event fires. Insight events are written to a separate S3 prefix.

Trail configuration best practices: create a single multi-region trail that captures all regions, enable for all accounts in the Organization (org trail), write to a centralized S3 bucket in a dedicated security/logging account, enable log file integrity validation, enable SSE-KMS encryption, and enable MFA Delete on the S3 bucket. This covers the exam's "secure, tamper-resistant audit logging" pattern completely.

Log file integrity validation works by CloudTrail generating a digest file every hour. The digest file contains the SHA-256 hash of each log file delivered in that hour, plus the hash of the previous digest file. This creates a hash chain. You can validate the chain with the aws cloudtrail validate-logs command to prove no logs were deleted or altered.

CloudTrail Lake is the newer feature — it's a managed data store for CloudTrail events that you can query with SQL directly without needing to export to S3 and run Athena. Retention up to 7 years. Useful when you want to search events interactively without managing S3 lifecycle and Athena table schemas.

One important detail: CloudTrail is not real-time. Log files are delivered to S3 within about 15 minutes of the API call. If you need near-real-time response to API calls, the pattern is CloudTrail → EventBridge (which does get CloudTrail management events in near-real-time via a separate pipeline) → Lambda or SNS.

## X-Ray

X-Ray collects trace data from distributed applications. A trace represents a single request as it flows through your entire system. A segment represents the work done by one service during that request. Subsegments represent downstream calls made within that service — calls to DynamoDB, HTTP requests to another service, SQL queries, etc. Annotations are indexed key-value pairs you attach to segments for filtering. Metadata is non-indexed key-value data for additional context.

The X-Ray daemon is a process that runs alongside your application. Your application code (using the X-Ray SDK) sends UDP packets to the daemon on port 2000. The daemon buffers these and sends them to the X-Ray API in batches. The daemon needs IAM permissions: xray:PutTraceSegments and xray:PutTelemetryRecords.

For Lambda, the daemon is built in — you just enable active tracing on the function. For ECS on EC2, you run the daemon as a sidecar container. For ECS on Fargate, same — sidecar container. For Elastic Beanstalk, there's a configuration option to enable it. For EC2, you install and run the daemon yourself.

Sampling is how X-Ray avoids recording every single request when traffic is high. The default rule is: record the first request per second for each host (the reservoir), then sample 5% of additional requests. You can create custom sampling rules in the console with different rates for different paths, HTTP methods, or services. Rules are evaluated in priority order.

The most common exam trap: "X-Ray service map is missing some services" or "some requests aren't showing up in traces." First check sampling — the service might be sampling at too low a rate. Second check the IAM permissions on the daemon or role. Third check whether the daemon is actually running and reachable on UDP 2000.

The service map is a visual graph of all services that have sent trace data, with edges showing call relationships. Each node is colored by error rate — green is healthy, yellow is warning, red is error. You can click a node to filter to traces that passed through that service, then drill into individual traces to see the exact timeline of calls.

Groups in X-Ray are filters on traces — you define a filter expression and traces matching it form a group. Groups can have their own service maps and are used to segment visibility (e.g., a group for all traces with error=true, or all traces hitting a specific endpoint).

X-Ray Insights (not to be confused with CloudTrail Insights) automatically detects anomalies in trace data within a group — unusual error rates, latency spikes, fault rates. It generates an insight event and tracks it through start, active, and resolved states.

X-Ray Analytics lets you interactively query trace data — filter by annotation, compare response time distributions between different time windows, drill into specific error types. It's the analysis layer on top of raw traces.

## EventBridge

EventBridge is the evolution of CloudWatch Events. The two are the same service — CloudWatch Events is the legacy name for the default event bus. All the old CloudWatch Events rules still work; they're just surfaced in the EventBridge console now.

Event buses: the default bus receives events from AWS services automatically. You create custom buses for your own application events. Partner buses are pre-configured to receive events from SaaS partners (Zendesk, Datadog, Auth0, etc.) without needing to build an ingestion layer.

Events are JSON objects with a standard envelope: version, id, source, account, time, region, resources, and detail-type are standard fields. The detail field is a JSON object whose schema varies by source. When AWS services emit events, AWS defines the schema. When your application emits events, you define it.

Rules have two components: an event pattern (filter) and one or more targets. Event patterns match against the event JSON using a subset of JSON path logic. You can match on: exact string values, prefix matching, suffix matching, numeric ranges, anything-but (negation), null values, and existence of a field. You can combine these with AND (implicit — multiple field conditions must all match) and OR (use array syntax for a field's values). There is no explicit OR across different fields in a single pattern — for that, you'd create multiple rules.

Targets receive the matched event. A single rule can have up to five targets. Targets include Lambda, Step Functions, SQS, SNS, Kinesis Data Streams, Firehose, ECS tasks, CodeBuild, CodePipeline, EC2 instances, another event bus (cross-account or cross-region), API Gateway, and more.

Input transformation lets you reshape the event before delivering it to a target. You can use JSONPath expressions to extract fields from the event and compose a new JSON object. This way you don't need a Lambda just to reshape the payload — EventBridge can do it natively.

Cross-account event delivery: you send events from a source account to a target account's event bus. The target bus needs a resource-based policy that allows events:PutEvents from the source account. The event then triggers rules on the target bus. This is the architecture for centralized event processing in a multi-account organization.

Scheduled rules use cron or rate expressions. Rate expressions are simple: rate(5 minutes), rate(1 hour). Cron expressions give you more control: cron(0 12 * * ? *) runs at noon UTC every day. EventBridge schedules run in UTC. The minimum schedule interval is 1 minute.

EventBridge Pipes is a newer feature that provides point-to-point integration between a source and a target with optional filtering and enrichment in between. Sources: SQS, Kinesis, DynamoDB Streams, Kafka, RabbitMQ. Targets: same set as rules. Enrichment: Lambda or Step Functions runs between filter and target. Pipes simplify the pattern of "poll SQS → process with Lambda → send to target" into a single managed resource.

Schema Registry: EventBridge can automatically discover the schema of events flowing through a bus. It infers the schema from the JSON and stores it. You can use the schema to generate code bindings in Python, Java, or TypeScript. This reduces the friction of consuming events by giving you a typed object model instead of raw JSON parsing.

EventBridge vs SNS vs SQS — the exam loves this comparison. EventBridge: content-based routing, rich filtering, many AWS and SaaS sources, schema registry, up to 5 targets per rule. SNS: simple pub/sub fan-out, many subscriber types (email, SMS, HTTP, SQS, Lambda, mobile push), message filtering on message attributes (not message body), up to 12.5M subscriptions per topic. SQS: durable queue, decouples producer and consumer, consumer polls, messages retained up to 14 days, FIFO ordering available. The decision tree: need to react to AWS service events? EventBridge. Need to fan out to email/SMS subscribers? SNS. Need durable decoupled processing with a consumer that controls its own pace? SQS. These can be chained: EventBridge → SNS → SQS for complex pipelines.