Brainstorm

- NewRelic APM Apdex alerting: works, but is not specific to each endpoint, so it takes a bigger change to make the whole app alert
- NewRelic Synthetics: Requires you to do work for every endpoint. Also, I don't think it will work when auth or any state (cookie/header) is required.
- NewRelic Key Transactions: Requires you to do work for every endpoint, but should otherwise work well. Alert based on endpoint specific Apdex or a custom defined response time.
- NewRelic Web transactions report, plot by avg time. Would have to check regularly to see issues. No alerting.
- NewRelic custom dashboard to make it easy to review this stuff manually instead of alerting?
- Custom logstash alerting: Poll logs. Assemble recent average response time per endpoint. Tricky to do endpoint aggregation? Need to store prev avg in a data store for comparison. How to alert? NewRelic? Email? OpsGenie?
- Custom anomaly detection may be quite complicated: http://www.coscale.com/blog/anomaly-detection-and-alerting-for-web-performance-monitoring
- But that is trying to find anomalies in arbitrary metrics. My needs are arguably more static. Threshold is 100% change in the average for 5 minutes in a row. Or is it? Performance may get worse during peak traffic times, but not worse enough to care? Now we're getting into the shape of your scale curve. Requiring that it not only stay at a low response time, but also that the shape is flat or nearly flat.

Recommendation

Combination of NewRelic Apdex alerting and Key Transaction alerting. KT is best, but lots of work to set up for every endpoint. So just draw the line where you see fit. Start with your most important transactions and go from there. Some less important endpoints will have to rely on app level Apdex alerting.
