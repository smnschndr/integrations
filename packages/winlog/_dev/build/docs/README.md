# Custom Windows event log package

The custom Windows event log package allows you to ingest events from any [Windows event log](https://docs.microsoft.com/en-us/windows/win32/wes/windows-event-log) channel.
You can get a list of available event log channels by running [`Get-WinEvent -ListLog * | Format-List -Property LogName`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent) in PowerShell on Windows Vista or newer.
If `Get-WinEvent` is not available, [`Get-EventLog *`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-eventlog) may be used.
Custom ingest pipelines may be added by setting one up in [Ingest Node Pipelines](/app/management/ingest/ingest_pipelines/).

## Choosing the right integration for Windows event logs

Use the Custom Windows event logs integration when you need to collect events from Windows event log channels that are not covered by the prebuilt integrations. This integration provides flexibility, but does not include specialized ingest pipelines—the data is collected in its raw form.

**Before using this integration**, check if one of these alternatives better fits your use case:

- **[System integration](https://www.elastic.co/docs/reference/integrations/system)**: Collects from the `Application`, `System`, and `Security` channels with specialized ingest pipelines that enrich the data for observability dashboards and alerting.
- **[Windows integration](https://www.elastic.co/docs/reference/integrations/windows)**: Collects from PowerShell, Sysmon, Windows Defender, AppLocker, and ForwardedEvents channels with security-focused ingest pipelines.

Using the System or Windows integrations for their supported channels provides better out-of-the-box value because their ingest pipelines parse and enrich the event data, making it more useful for dashboards, searches, and security detections.

## Configuration

### Collecting multiple channels

The `winlog` input reads exactly one channel per configuration, so the **Channel Name** option accepts a single
channel. There are two ways to collect from more than one channel:

1. **One integration policy per channel.** Each policy gets its own reader, its own checkpoint and, if you want,
   its own dataset. This is the recommended approach when you need per-channel filtering, routing or ingest
   pipelines.
2. **A single policy with a custom XML query.** Leave **Channel Name** empty and set **XML Query** to a
   `QueryList` that selects from several channels. The input then queries all listed channels through one
   reader. Because the query is mutually exclusive with the simple filter options, it also has to carry an
   identifier, which is supplied through the **Custom Configurations** option:

   *XML Query*

   ```xml
   <QueryList><Query Id="0"><Select Path="Application">*</Select><Select Path="System">*</Select><Select Path="Microsoft-Windows-PowerShell/Operational">*</Select></Query></QueryList>
   ```

   *Custom Configurations*

   ```yaml
   id: my-multi-channel-query
   ```

   You can build such a query in Event Viewer: create a custom view, switch to the **XML** tab and copy the
   generated `QueryList`. Put the query on a single line as shown above: on older Kibana versions a multi-line
   value produces an invalid agent policy, so a single-line query is the portable form.

   Note the trade-offs of this approach:

   - **XML Query** is mutually exclusive with **Channel Name**, **Event ID**, **Ignore events older than**,
     **Level** and **Providers**. All filtering has to be expressed inside the query itself.
   - All matched events are written to the single dataset configured in the policy, so events from different
     channels cannot be routed to different indices.
   - All channels share one reader and one checkpoint, so collection resumes for the query as a whole rather
     than per channel.

### Windows Event ID clause limit

If you specify more than 22 query conditions (event IDs or event ID ranges), some
versions of Windows will prevent the integration from reading the event log due to
limits in the query system. If this occurs, a similar warning as shown below:

```
The specified query is invalid.
```

In some cases, the limit may be lower than 22 conditions. For instance, using a
mixture of ranges and single event IDs, along with an additional parameter such
as `ignore older`, results in a limit of 21 conditions.

If you have more than 22 conditions, you can work around this Windows limitation
by using a drop_event processor to do the filtering after filebeat has received
the events from Windows. The filter shown below is equivalent to
`event_id: 903, 1024, 2000-2004, 4624` but can be expanded beyond 22 event IDs.

```yaml
- drop_event.when.not.or:
  - equals.winlog.event_id: "903"
  - equals.winlog.event_id: "1024"
  - equals.winlog.event_id: "4624"
  - range:
      winlog.event_id.gte: 2000
      winlog.event_id.lte: 2004
```

## Fields Mapping

In addition to the fields specified below, this integration includes the ECS Dynamic Template. Any field that follow the ECS Schema will get assigned the correct index field mapping and does not need to be added manually.

{{ fields }}