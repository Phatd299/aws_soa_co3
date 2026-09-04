# Automated Remediation: EventBridge, Runbooks, and EC2 Recovery (SOA-C03)
Automated remediation is the practice of detecting an operational problem from metrics, alarms, or events and fixing it without a human logging in — the core of SOA-C03 Task 1.2. The exam assumes the metrics and alarms already exist (that was Task 1.1) and asks what happens next: how you isolate the failing component with metric math, anomaly detection, and CloudTrail correlation, and how you wire detection to action — an alarm's EC2 recover action for hardware failure, Auto Scaling replacing an unhealthy instance, EventBridge routing an event to Lambda, or a Systems Manager Automation runbook executing a documented fix. It also tests the unglamorous part: troubleshooting an EventBridge rule that never fires and verifying a remediation actually worked. This lesson walks the full detect-route-act-verify loop, compares the remediation patterns the exam makes you choose between, and covers the newly in-scope signal sources: the AWS Health Dashboard, Kiro, and AWS DevOps Agent.

## What you’ll learn

- Trace the remediation loop — detect, route, act, verify — and place each AWS service in it
- Analyze incidents with metric math, anomaly detection, and CloudTrail change correlation to isolate the failing component
- Choose between EC2 auto recovery, Auto Scaling replacement, Lambda, and Systems Manager runbooks for a given failure
- Build and troubleshoot EventBridge rules, event patterns, and input transformers that route events to remediation targets
- Run predefined and custom Systems Manager Automation runbooks and keep configuration in state with State Manager
- Recognize the AWS Health Dashboard, Kiro, and AWS DevOps Agent as remediation signal sources at exam level

## The remediation loop: detect, route, act, verify

Every automated-remediation design on this exam is the same four-step loop, and naming the steps makes unfamiliar scenarios easy to decompose. Detect: a signal says something is wrong — a CloudWatch alarm breaches, an EC2 status check fails, an AWS Health event announces degraded hardware, or an anomaly-detection band is exceeded. Route: the signal travels to whatever will act on it — an alarm's built-in action, an SNS topic fanning out to subscribers, or an EventBridge rule matching the event and forwarding it to targets. Act: something fixes the problem — the EC2 recover action, an Auto Scaling replacement, a Lambda function running custom logic, or a Systems Manager Automation runbook executing a documented procedure. Verify: confirm the fix worked — the alarm transitions back to OK, the runbook reports step success, or an OpsItem records the incident for review.

The verify step is the one candidates and junior engineers skip, and the exam notices. A Lambda that restarts a service but never checks the result is half a remediation; a design that closes the loop — act, then watch the same metric that detected the problem return to healthy — is the stronger answer.

Two scope boundaries keep this task tidy. Creating the metrics, filters, and alarms themselves belongs to Task 1.1; here they are inputs. And tuning the resource that keeps misbehaving — resizing an instance, changing an EBS volume type, adding RDS Proxy — is performance optimization, Task 1.3. Task 1.2 is the plumbing between a signal and a fix, and questions reward the design with the least ongoing human effort: managed loops over hand-run commands.

## Isolating the failing component: metric math, anomaly detection, and CloudTrail

Before you can remediate, you have to identify what is failing, and Task 1.2 names three analysis tools. Metric math builds expressions over existing metrics so you can alarm on the ratio or aggregate that actually matters. Raw error counts mislead — 50 errors is noise at 100,000 requests and an outage at 60 — so an expression like 100*(errors/requests) produces an error-rate metric, and the alarm watches the expression instead of either raw series. Metric math also aggregates across resources: summing a metric over a fleet, or subtracting healthy hosts from desired capacity to alarm on the gap.

Anomaly detection answers the question a static threshold cannot: what is normal right now? CloudWatch trains a model on the metric's history and draws an expected band around it, accounting for daily and weekly patterns; the alarm fires when the metric leaves the band. Choose it for metrics with strong cyclical behavior — request volume, queue depth — where any fixed threshold is either too tight at peak or too loose at night. Choose a plain threshold when the limit is genuinely absolute, like a disk filling up.

The third tool is correlation with CloudTrail. Incidents caused by change dominate real operations, so line the incident timeline up against recent API activity: latency spiked at 14:02, and CloudTrail shows a security group was modified at 14:01 — you have your suspect, and the remediation is to revert the change, not restart servers. When an exam scenario asks how to determine whether a configuration change caused an outage, CloudTrail is the answer; CloudWatch tells you when it broke, CloudTrail tells you what changed.

## EC2 status checks: recover the hardware, repair the OS

EC2 runs two automated status checks per instance, and telling them apart is one of the highest-yield facts in Domain 1 — because each maps to a different fix.

| Check | Metric | What it detects | The fix |
| --- | --- | --- | --- |
| System status check | StatusCheckFailed_System | Problems with AWS infrastructure under the instance: loss of power, loss of network connectivity, hardware or software failure on the physical host | The recover action (or stop/start) — move the instance to healthy hardware |
| Instance status check | StatusCheckFailed_Instance | Problems inside your instance: exhausted memory, corrupted filesystem, misconfigured network settings, failed kernel | OS-level intervention — reboot, fix the configuration, or rebuild |

The recover action is an alarm action on StatusCheckFailed_System: CloudWatch detects the host-level failure and migrates the instance to new underlying hardware, preserving its instance ID, private and Elastic IP addresses, and attached EBS volumes. In-memory data is lost — recovery is a migration and restart, not a live move. The trigger phrase to watch for is "underlying hardware failure" or "host-level issue": the answer is recover. A recover action can never fix a failed instance check, because new hardware does not cure a corrupted filesystem or a bad kernel — that failure needs a reboot alarm action or hands on the OS.

For fleets, Auto Scaling is the remediation layer: its health checks (EC2 status checks by default, optionally ELB health checks so "failing the load balancer" also counts as unhealthy) mark a bad instance unhealthy, terminate it, and launch a replacement. Auto Scaling replaces rather than repairs — the right model for stateless fleets, and the wrong one for a stateful single instance, which is exactly the distinction scenario questions probe.

## Choosing a remediation pattern

The exam's remediation questions are usually pattern-selection questions: given a failure and a constraint (least operational overhead, no code, multi-step fix), pick the right wiring. Learn the menu.

| Pattern | Best for | Watch out for |
| --- | --- | --- |
| Alarm → EC2 action (recover/reboot/stop) | Single-instance hardware or hang failures | Recover only helps system-check failures |
| Auto Scaling health replacement | Stateless fleets — terminate and replace unhealthy instances | Replacement, not repair; instance state is lost |
| Alarm → SNS → Lambda | Custom remediation logic plus human notification from one topic | You own the Lambda code and its error handling |
| Alarm state change → EventBridge → target | Flexible routing: Lambda, Systems Manager Automation, Step Functions, SQS; filtering and multiple targets | Rule patterns and target permissions must be right (next section) |
| EventBridge → Systems Manager runbook | Multi-step, documented, repeatable fixes — no custom code | Runbook needs IAM permissions for its steps |

Two habits resolve most choices. First, prefer managed capability over custom code: if a predefined runbook or a built-in alarm action does the job, it beats a Lambda you must write, test, and maintain — that is what "least operational overhead" means. Second, match the blast radius: instance-level failures get instance-level actions; fleet-level symptoms get Auto Scaling; anything needing sequenced steps with verification gets an Automation runbook. SNS stays in the picture even when automation acts, because remediation you were never told about is how small incidents hide until they become large ones — a topic can notify humans in parallel with the automated fix.

## EventBridge in depth: buses, patterns, transformers — and rules that don't fire

Amazon EventBridge is the routing layer of AWS operations. Every account has a default event bus that automatically receives events from AWS services — alarm state changes, EC2 instance state changes, AWS Health notifications — and you can add custom buses for your own applications or partner buses for SaaS sources. A rule attaches to one bus, matches events against an event pattern, and delivers matches to up to five targets. Patterns are JSON that must match the event's actual structure: every field you specify must be present in the event with one of the values you list, matching is exact and case-sensitive, and values sit in arrays ("state": {"value": ["ALARM"]}).

An input transformer reshapes the event before delivery: you extract fields with input paths and inject them into a template, so a target receives a clean payload — say, just the alarm name and instance ID — instead of the full event envelope. That is the exam's "enrich or shape events without adding a Lambda" answer. Rules can also run on a schedule (rate or cron expressions), which turns EventBridge into the trigger for periodic maintenance automation.

Troubleshooting a rule that never fires is an explicit skill (1.2.2). Work the checklist in order: the pattern doesn't match the real event (test it against a sample event — a typo'd field name or wrong case silently matches nothing); the rule is disabled; the target lacks permission (Lambda and SNS targets need a resource-based policy allowing EventBridge; targets like Systems Manager Automation need an IAM role on the rule); the event is arriving on a different bus than the rule watches; or the event occurs in a different Region — EventBridge is regional, and a rule in us-east-1 will never see a us-west-2 alarm. Cross-account delivery adds one more: the receiving bus needs a resource-based policy permitting the sender.

## Systems Manager: runbooks, Run Command, State Manager, OpsCenter

AWS Systems Manager is the act-and-verify half of the loop, and Task 1.2 names its pieces. Automation runbooks are step-by-step documents that execute operational procedures against AWS resources. AWS ships predefined runbooks with the AWS- prefix — AWS-RestartEC2Instance restarts an instance, and there are predefined documents for common tasks like stopping instances or creating images — while custom runbooks you author in YAML or JSON sequence your own steps: invoke AWS API actions, run scripts, branch on results, pause for approval. Runbooks can be started manually, from an EventBridge rule, on a schedule, or through the AWS SDKs, and they run under an IAM role that must permit each step's actions. When a scenario needs a multi-step, repeatable, documented fix without writing a Lambda, the answer is an Automation runbook.

Run Command executes commands on managed instances at scale with no SSH and no bastion — ad hoc or scripted operations like restarting a service across fifty instances, with output captured per instance. State Manager is the drift corrector: an association binds a document and its parameters to a set of instances on a schedule, and Systems Manager reapplies the configuration whenever it drifts. "Ensure this agent/setting stays configured on every instance" is a State Manager question, not a run-it-once question.

OpsCenter aggregates operational work items as OpsItems — alarms and EventBridge rules can create them automatically, giving each incident a record with related resources and suggested runbooks attached, which is how the verify-and-review step gets institutionalized. At mention level, Incident Manager layers response plans, escalation, and on-call engagement on top for the incidents that need humans immediately.

## Wider signals: AWS Health Dashboard and AI-assisted operations

Not every remediation signal starts in your own metrics. The AWS Health Dashboard shows events that affect your account specifically: operational issues impacting your resources, scheduled maintenance, and planned lifecycle changes such as hardware retirement or certificate expirations. That account-specific view is what distinguishes it from generic public status information — it tells you which of your instances is scheduled for maintenance, not just that a service has a problem somewhere.

Health events are also delivered to EventBridge, which upgrades the dashboard from something you remember to check into a remediation source. A rule matching aws.health events can notify the operations channel through SNS, open an OpsItem, or trigger automation — for example, reacting to a scheduled-maintenance notice for an EC2 instance by draining and replacing it during a change window instead of being surprised later. The exam treats Health as exactly this: an ops signal source that plugs into the same detect-route-act loop as any alarm.

## Kiro and AWS DevOps Agent

The SOA-C03 guide newly names two agentic AI tools, Kiro and AWS DevOps Agent, as examples alongside CloudWatch, Lambda, Systems Manager, and CloudTrail for analyzing performance metrics and automating remediation strategies. Know them at that level: AI-assisted tools an operations team can apply to metric analysis and remediation automation, and plausible correct-sounding names in a question about modern remediation options. The guide does not enumerate their features, so resist any answer choice that depends on invented specifics — the tested point is recognizing where they fit in the loop, not configuring them.

## Walkthrough: auto-healing a web tier

Put the loop together on a realistic setup: an Auto Scaling group of stateless web servers in front of one stateful reporting instance that cannot simply be replaced.

The fleet largely heals itself. The Auto Scaling group uses ELB health checks, so an instance that stops answering the load balancer — not just one that fails EC2 status checks — is marked unhealthy, terminated, and replaced. Your job is detection coverage on top: an alarm on the group's unhealthy-host count routed to SNS, so a pattern of replacements gets investigated rather than silently absorbed.

The stateful instance gets two alarms. First, StatusCheckFailed_System with the recover action — if the underlying host fails, the instance migrates to healthy hardware keeping its instance ID, IPs, and EBS volumes. Second, StatusCheckFailed_Instance, 2-out-of-2 datapoints, with no EC2 action; instead, an EventBridge rule matches this alarm's state change to ALARM and targets the AWS-RestartEC2Instance Automation runbook, with an input transformer extracting the instance ID from the event into the runbook's parameter. The rule's IAM role permits starting the automation; SNS notifies the on-call in parallel.

Verify, then troubleshoot. Success looks like: runbook execution succeeds, the status-check alarm returns to OK, and an OpsItem records the event. During the game-day test, the runbook never ran — so you walk the EventBridge checklist: the pattern had "state": ["ALARM"] instead of matching the event's actual nested state.value field, so it matched nothing. Fix the pattern, replay the test, watch the loop close. That debugging sequence — symptom, checklist, single root cause — is precisely how the exam frames these questions.

> 💡 Tip. Task 1.2 questions hand you a failure and make you pick the remediation wiring. "Underlying hardware failure" or a failed system status check points to the EC2 recover action; an OS-level symptom like a corrupted filesystem points to instance-check handling, not recovery; unhealthy instances in a fleet point to Auto Scaling replacement. "The rule/automation never ran" is an EventBridge troubleshooting question — pattern mismatch, disabled rule, missing target permissions, or wrong bus/Region. Phrases like "least operational overhead" and "no custom code" steer you toward predefined Systems Manager Automation runbooks over Lambda, and "configuration stays enforced" or "drift" points to State Manager.

## Key takeaways

- StatusCheckFailed_System means AWS infrastructure under the instance failed — the recover action migrates it to healthy hardware, preserving instance ID, IP addresses, and EBS volumes.
- StatusCheckFailed_Instance means an OS-level problem (exhausted memory, corrupted filesystem, bad network config, failed kernel) — recovery can't fix it; reboot or repair the OS.
- Auto Scaling remediates fleets by terminating and replacing unhealthy instances — replacement, not repair; add ELB health checks so failing the load balancer counts as unhealthy.
- An EventBridge rule that never fires: check the event pattern against a real sample event, confirm the rule is enabled, verify target permissions, and confirm the event reaches that bus in that Region.
- Input transformers reshape event JSON before delivery — enrich or trim the payload with no Lambda in the middle.
- AWS-prefixed Automation runbooks like AWS-RestartEC2Instance are predefined; author custom YAML/JSON runbooks for multi-step, workload-specific remediation, and prefer runbooks over custom code for least operational overhead.
- State Manager associations continuously reapply desired configuration — the drift-correction answer; Run Command is for ad hoc execution at scale without SSH.
- AWS Health Dashboard events (account-specific issues and planned maintenance) flow into EventBridge, turning upcoming hardware retirements into proactive remediation triggers.

