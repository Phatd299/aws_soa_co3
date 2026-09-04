# CloudWatch Metrics, Alarms, and Log Filters for SOA-C03

Amazon CloudWatch is the monitoring and logging backbone of AWS operations: it collects metrics, ingests logs, and raises alarms for every workload you run. Task 1.1 of the SOA-C03 exam tests whether you can wire that telemetry correctly — choose between standard and detailed monitoring, publish custom metrics, install the CloudWatch agent for the metrics EC2 does not emit on its own, convert log patterns into alarmable metrics with metric filters, and connect alarms to Amazon SNS so someone actually gets paged. It also expects you to keep CloudWatch and AWS CloudTrail straight — performance telemetry versus an audit trail of API calls — and to build dashboards that span accounts and Regions. This lesson covers all five skills in the task, from metric fundamentals through alarm anatomy to notification wiring, including the trap answers the exam loves: missing memory metrics, alarms stuck in INSUFFICIENT_DATA, and notifications that never arrive.

## What you’ll learn

- Distinguish standard, detailed, and high-resolution metrics, and choose the right resolution for a workload
- Configure the CloudWatch agent to collect the memory, disk-space, and log data that EC2 does not publish by default
- Create metric filters and subscription filters that turn log patterns into metrics, alarms, and streamed exports
- Design alarms with evaluation periods, datapoints-to-alarm, missing-data handling, and composite suppression
- Wire alarm actions to SNS, Auto Scaling, EC2 actions, and Systems Manager — directly or through EventBridge
- Separate CloudTrail's API-audit role from CloudWatch's performance telemetry and know when the exam wants each

## CloudWatch metrics: namespaces, dimensions, and resolution

A CloudWatch metric is a time-ordered series of datapoints identified by three things: a namespace (a container such as AWS/EC2 or AWS/Lambda, or a custom one like MyApp/Orders), a metric name (such as CPUUtilization), and zero or more dimensions — key-value pairs like InstanceId=i-0abc123 that scope the metric to a specific resource. This matters operationally because CloudWatch treats every unique combination of namespace, name, and dimensions as a separate metric. If you publish a custom metric with dimension Env=prod and later query it with no dimensions, you get nothing — a classic reason a dashboard or alarm shows no data.

Resolution determines how often datapoints arrive. EC2 standard monitoring sends instance metrics every 5 minutes at no charge. Detailed monitoring raises that to 1-minute metrics for an extra cost, and you enable it per instance (or in a launch template). Detailed monitoring does not add any new metrics — it only makes the existing ones more frequent, which lets alarms and Auto Scaling react faster.

Custom metrics you publish yourself with PutMetricData come in two resolutions: standard resolution stores data at 1-minute granularity, while high-resolution metrics store data down to 1-second granularity. High-resolution metrics support high-resolution alarms that evaluate on 10-second or 30-second periods — the right choice when a spiky workload can fail faster than a 1-minute alarm could notice. Choose resolution by asking one operations question: how quickly must I detect and react? Faster resolution costs more, so reserve it for the metrics that gate paging or scaling decisions.

## What EC2 doesn't tell you: the CloudWatch agent

The metrics EC2 publishes by default come from the hypervisor, which can only see the instance from the outside: CPU, network bytes and packets, disk I/O operations, and status checks. The hypervisor cannot see inside the guest operating system, so the default metric set never includes memory utilization, disk-space utilization, or swap usage. This is the single most-tested fact in Task 1.1: when a question says memory or free-disk metrics are "missing" from CloudWatch, the answer is to install and configure the CloudWatch agent on the instance.

| Source | Frequency | What you get |
| --- | --- | --- |
| Standard monitoring | 5 minutes, free | CPU, network, disk I/O, status checks (hypervisor view) |
| Detailed monitoring | 1 minute, paid | Same metrics, higher frequency — nothing new |
| CloudWatch agent | Configurable | In-guest metrics: memory, disk space, swap, processes — plus log file collection |

The agent runs inside the OS, reads a JSON configuration that lists which metrics and log files to collect, and pushes them to CloudWatch. The operational best practice — and a frequent exam answer — is to store that configuration in Systems Manager Parameter Store so an entire fleet pulls one consistent config, and to install the agent at scale with Systems Manager rather than by hand.

For containers, Container Insights deploys the agent (or a sidecar) into Amazon ECS or EKS clusters to collect cluster-, service-, task-, and pod-level metrics and logs. Serverless services publish their own metrics automatically — Lambda emits Invocations, Errors, Duration, and Throttles with no agent — and AI workloads such as Amazon Bedrock likewise publish invocation and latency metrics to CloudWatch. If a team standardizes on Prometheus-format metrics for containers, Amazon Managed Service for Prometheus collects and stores them and Amazon Managed Grafana visualizes them, both without you operating the monitoring servers.

## CloudWatch Logs: groups, streams, retention, and Logs Insights

CloudWatch Logs organizes everything into log groups and log streams. A log group is the unit of management — retention, access control, metric filters, and subscription filters all attach at the group level. A log stream is one ordered sequence of events from a single source, such as one EC2 instance or one Lambda execution environment. Operationally you almost always work at the group level: one group per application or component, with each instance writing its own stream inside it.

Retention is set per log group and defaults to never expire. That default is an exam-relevant cost trap: log storage grows forever until someone sets a retention policy, which you can configure from 1 day up to 10 years. When a question describes runaway CloudWatch Logs costs, setting group retention (or exporting old data to S3) is the operations-first answer.

CloudWatch Logs Insights is the interactive query engine you reach for during an incident. You select one or more log groups, pick a time range, and run queries built from commands like fields, filter, stats, and sort — for example, counting errors per five-minute bin to see exactly when a failure started, or filtering for a request ID across services. Insights queries existing ingested data on demand; it does not create metrics or alarms, which is what separates it from metric filters in the next section.

Logs arrive from many producers: the CloudWatch agent ships OS and application log files, Lambda writes function logs automatically, VPC Flow Logs can target a log group, and CloudTrail can optionally deliver its events into CloudWatch Logs for alarming — a bridge you will see again shortly.

## Metric filters: turning log patterns into alarms

A metric filter watches a log group for a pattern and increments a CloudWatch metric every time the pattern matches. It is the exam's favorite move in this task because it converts unstructured log lines into something you can alarm on. Two rules to remember: metric filters apply only to log data ingested after the filter is created (they never backfill history), and the metric they publish behaves like any other custom metric — namespace, name, and value are yours to define.

Walk through the classic scenario. Your application writes lines containing ERROR to a log file. First, the CloudWatch agent ships that file to the log group /myapp/prod. Second, you create a metric filter on the group with the pattern ERROR, publishing metric ErrorCount in namespace MyApp with a metric value of 1 per match — and a default value of 0 so the metric still emits datapoints during quiet periods instead of going missing. Third, you create an alarm: ErrorCount sum ≥ 5 over a 5-minute period, 2 of 3 datapoints breaching, action = notify an SNS topic. Now a burst of application errors pages the on-call within minutes, with no code changes.

A subscription filter is the other kind of filter and answers a different question. Instead of counting matches into a metric, it delivers the matching log events themselves, in near real time, to a destination: Kinesis Data Streams, Amazon Data Firehose, or a Lambda function. Use metric filters when you need a number to alarm on; use subscription filters when you need the log data to flow somewhere — centralized analysis, transformation, or delivery to storage.

## CloudTrail for operators: audit trail, not telemetry

AWS CloudTrail records API calls: who made the call, from where, when, against which resource, and whether it succeeded. That makes it an audit service, and the exam repeatedly checks that you keep it distinct from CloudWatch. "Which user modified the security group?" is CloudTrail. "Why did CPU spike at 14:00?" is CloudWatch. When a question mixes the two, sort it by asking whether you need actions taken (CloudTrail) or how the system is performing (CloudWatch).

CloudTrail distinguishes two event classes. Management events are control-plane operations — launching instances, changing IAM policies, modifying security groups — and are recorded by default in the CloudTrail event history, which keeps 90 days of them for free. Data events are high-volume data-plane operations such as S3 object-level reads and writes or Lambda function invocations; they are not logged by default and must be enabled explicitly, with cost implications, on a trail.

To keep events beyond 90 days, you create a trail that delivers events continuously to an S3 bucket. At the concept level the exam expects, an organization trail extends this: one trail configured in the management account that logs API activity for every account in the organization to a single bucket — one audit pipeline instead of dozens.

The two services also connect. A trail can deliver its events into CloudWatch Logs, where a metric filter can count sensitive calls — say, ConsoleLogin failures or security-group changes — and an alarm can page on them. That pattern, CloudTrail for the record and CloudWatch for the reaction, is exactly the kind of integration Task 1.1 rewards.

## Alarm anatomy: thresholds, M out of N, and missing data

A CloudWatch alarm watches one metric (or metric expression) and sits in one of three states: OK (threshold not breached), ALARM (breached), or INSUFFICIENT_DATA (not enough datapoints to decide). New alarms start in INSUFFICIENT_DATA, and an alarm returns there when the metric stops arriving — a stopped instance, a broken agent, or a custom metric published with the wrong namespace or dimensions.

Three settings control when an alarm fires. The period is how long each evaluated datapoint covers. Evaluation periods (N) is how many recent datapoints the alarm examines. Datapoints to alarm (M) is how many of those must breach — the "M out of N" setting. An alarm on CPU > 80% with a 1-minute period and 3-out-of-3 datapoints requires three consecutive breaching minutes; setting 2-out-of-3 tolerates a single recovered blip while still catching sustained problems. M out of N is the tool for taming flapping alarms without dulling detection, and high-resolution alarms can evaluate on 10- or 30-second periods when a minute is too slow.

Treat missing data decides what a gap in the metric means, and it is a reliable exam discriminator. The four options: missing (the default — the alarm ignores the gap while it can, then goes to INSUFFICIENT_DATA), ignore (keep the current state), breaching (treat gaps as bad — right when silence itself is the failure, such as a heartbeat metric), and notBreaching (treat gaps as good — right for sparse metrics like an error count that only emits when errors occur, unless you gave the metric filter a default value of 0). Choosing this deliberately is the difference between an alarm you trust and one the team learns to ignore.

## Alarm actions, composite alarms, and SNS wiring

Alarm actions run on state transitions, and you attach them per target state — usually on entering ALARM, sometimes on returning to OK. A standard metric alarm can act directly in four ways: publish to an SNS topic, invoke an EC2 Auto Scaling policy, take an EC2 action on the instance behind the metric (stop, terminate, reboot, or recover), or create a Systems Manager OpsItem or incident so the event lands in your operations queue. Every alarm state change is also emitted as an event to Amazon EventBridge, which can route it to almost any target — that is the doorway to full automated-remediation flows, which belong to Task 1.2; here, the point is knowing the alarm's invokable actions.

A composite alarm combines other alarms with AND, OR, and NOT logic and exists to suppress noise. If a load balancer's 5xx alarm and its latency alarm both tend to fire during one incident, a composite alarm of 5xx AND latency pages once, on the real event, instead of twice on every wobble. Composite alarms notify through SNS (and can create OpsItems or incidents); they do not take EC2 or Auto Scaling actions — the child alarms do that.

Notification wiring trips up real teams and exam candidates alike. An alarm needs an SNS topic in the same Region, and the topic needs at least one confirmed subscription — an email endpoint receives a confirmation message that someone must click. "The alarm went to ALARM but nobody was notified" almost always resolves to one of: the subscription was never confirmed, the alarm points at the wrong (or empty) topic, or the alarm action was disabled. Check the topic and its subscriptions before touching the alarm itself.

## Dashboards across accounts and Regions

CloudWatch dashboards are the shareable, customizable home page for a workload's health, and Task 1.1 calls out one specific capability: displaying metrics and alarms across multiple accounts and Regions. Dashboards are global resources — a single dashboard can hold widgets that each pull metrics from a different Region, so one screen can show your us-east-1 API fleet next to its eu-west-1 replica. That multi-Region view is native; you simply choose the Region when adding each widget.

Crossing accounts uses CloudWatch cross-account observability: you designate a central monitoring account and link source accounts to it, after which the monitoring account can search, graph, dashboard, and alarm on metrics and logs from every linked account without anyone switching roles. For an operations team running many workload accounts, this is the expected answer to "view all accounts' metrics in one place" — one pane of glass, built once, instead of console-hopping during an incident.

Build dashboards around decisions, not decoration. A good workload dashboard pairs each headline metric with its alarm-status widget so the viewer sees both the value and whether it is currently breaching; text widgets can hold runbook links for the 2 a.m. responder. Dashboards can be shared beyond the console when stakeholders without AWS access need visibility. Two practical notes that surface in questions: dashboards display metrics but do not evaluate anything (alarms do that), and automatic dashboards exist per service to give you an instant, unedited view of a service's key metrics before you have built anything custom.

> 💡 Tip. Expect scenario questions that hide the answer in one trigger phrase. "Memory utilization is missing from CloudWatch" points to installing the CloudWatch agent; "metrics every minute instead of every 5 minutes" points to detailed monitoring; "alert when a specific pattern appears in the logs" points to a metric filter feeding an alarm. Alarm questions probe the mechanics — M-out-of-N datapoints for flapping alarms, treat-missing-data for sparse metrics, composite alarms to cut duplicate pages — and "the alarm fired but nobody was notified" resolves to an unconfirmed or missing SNS subscription. Finally, keep the audit-versus-telemetry line sharp: questions about who made an API change are CloudTrail, not CloudWatch.

## Key takeaways

- Default EC2 metrics never include memory or disk-space utilization — collecting those requires the CloudWatch agent.
- Standard EC2 monitoring is 5-minute metrics; detailed monitoring is 1-minute, costs extra, and adds frequency, not new metrics.
- Custom metrics are 1-minute standard resolution or 1-second high resolution; high-resolution alarms evaluate on 10- or 30-second periods.
- Metric filters turn log patterns into alarmable metrics but only match data ingested after creation; subscription filters stream the events themselves to Kinesis Data Streams, Firehose, or Lambda.
- An alarm has three states — OK, ALARM, INSUFFICIENT_DATA — and M-out-of-N datapoints plus a deliberate treat-missing-data choice keep it from flapping or lying.
- Composite alarms combine child alarms with AND/OR/NOT to suppress noise, and they act by notifying (SNS), not by taking EC2 or scaling actions.
- An alarm notification needs an SNS topic in the same Region and a confirmed subscription — "alarm fired, no email" means check the subscription first.
- CloudTrail answers who did what and when (API audit, management vs data events); CloudWatch answers how the system is performing.