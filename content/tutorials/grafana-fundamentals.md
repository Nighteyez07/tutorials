
---
title: Grafana fundamentals
summary: Get familiar with Grafana
id: grafana-fundamentals
categories: ["fundamentals"]
tags: beginner
status: Published
authors: Grafana Labs
Feedback Link: https://github.com/grafana/tutorials/issues/new
---

{{% tutorials/step duration="1" title="Introduction" %}}

In this tutorial, you'll learn how to use Grafana to set up a monitoring solution for your application.

### What you'll learn

- How to explore metrics and logs
- How to build dashboards
- How to annotate dashboards
- How to set up alerts

### What you'll need

- [Docker](https://docs.docker.com/install/)
- [Docker Compose](https://docs.docker.com/compose/)

{{% /tutorials/step %}}
{{% tutorials/step title="Set up the sample application" %}}

This tutorial uses a sample application to demonstrate some of the features in Grafana. To complete the exercises in this tutorial, you need to download the files to your local machine.

- Using [Git](https://git-scm.com/), clone the example code:

```
git clone https://github.com/grafana/monitoring-intro-workshop.git
```

- Go to the directory where you cloned this repository:

```
cd monitoring-intro-workshop
```

- Start the sample application:

```
docker-compose up -d
```

The first time you run `docker-compose up -d`, Docker downloads all the necessary resources for this tutorial. This might take a few minutes, depending on your internet connection.

- Ensure all services are up-and-running:

```
docker-compose ps
```

All services should report their state as "Up".

- Browse to [localhost:8081](http://localhost:8081) to start using the sample application.

{{% /tutorials/step %}}
{{% tutorials/step title="Add a Prometheus data source" %}}

- In the side bar, click **Configuration** -> **Data Sources**.
- Click **Add data source**, and select "Prometheus" from the list of available data sources.
- In the URL box, type "http://prometheus:9090".
- Click **Save & Test** to save your changes.

{{% /tutorials/step %}}
{{% tutorials/step title="Explore time series" %}}

Grafana lets you explore and compare metrics from your data sources.

- In the side bar, click **Explore**.
- In the **Query** box, type the following, and press Enter:

```
tns_request_duration_seconds_count
```

- In the top right corner, click the drop down on the **Run Query** button, and select "5s" to have Grafana run your query every 5 seconds.

PromQL is a powerful query language that lets you apply transformations to your queries. Since `tns_request_duration_seconds_count` is a counter, it will only increase. For something more interesting, try the rate and sum functions:

- Add the `rate` function to your query to visualize the rate of requests per second:

```
irate(tns_request_duration_seconds_count[5m])
```

- Add the `sum` function to your query to group time series by route:

```
sum(irate(tns_request_duration_seconds_count[5m])) by(route)
```

- Go back to the sample application and generate some traffic by adding new links, voting, or just refresh the browser.

> If you don't want to manually refresh the browser, hey is a great tool to generate traffic.

```
hey -n 10 -c 10 -m GET http://localhost:8081
hey -n 10 -c 10 -m POST -H "Content-Type: application/x-www-form-urlencoded" -d "title=Example&url=http://example.com" http://localhost:8081/post
```

{{% /tutorials/step %}}
{{% tutorials/step title="Add a Loki data source" %}}

- In the side bar, click **Configuration** -> **Data Sources**.
- Click **Add data source**, and select "Loki" from the list of available data sources.
- In the URL box, type [http://loki:3100](http://loki:3100).
- Click **Save & Test** to save your changes.

{{% /tutorials/step %}}
{{% tutorials/step title="Explore logs" %}}


- In the side bar, click **Configuration** -> **Data Sources**.

- Click **Add data source**, and select **Loki** from the list of available data sources.

- In the **URL** box, type `http://loki:3100`.

- Click **Save & Test** to save your changes.

### Explore logs

- In the side bar, click **Explore**.
- In the dropdown at the top, select the "Loki" data source.
- In the **Query** box, type the following, and press Enter to display all logs within the log file of the sample application:

```
{filename="/var/log/tns-app.log"}
```

- Filter log lines based on a substring:

```
{filename="/var/log/tns-app.log"} |= "error"
```

Grafana only shows logs within the current time interval. This lets you narrow down your logs to a certain time.

- Click and drag over a log occurrence in the graph to filter logs based on time.

{{% /tutorials/step %}}
{{% tutorials/step title="Build a dashboard" %}}

Panels are the building blocks of Grafana dashboards. Every panel consists by a _query_ and a _visualization_.

- In the side bar, click **Create** to create a new dashboard.
- Click **Add query**, and enter the query from earlier:

```
sum(rate(tns_request_duration_seconds_count[5m])) by(route)
```

- In the **Legend** box, enter "{{route}}" to rename the time series in the legend.
- Click the **Visualization** panel tab to the left to go to the visualization settings for the panel.
- Enable the **Stack** option to stack the series on top of each other. This makes it easier to see the total traffic across all routes.
- Click the **General** panel tab, and change the title to "Traffic".
- Click the arrow in the top-left corner to go back to the dashboard view.
- Click the **Save dashboard** icon at the top of the dashboard, to save your dashboard.

{{% /tutorials/step %}}
{{% tutorials/step title="Annotate events" %}}

Whenever things go bad, it can be invaluable to understand the context in which the system failed. Time of last deploy, or database migration can offer insight into what might have caused an outage. Annotations lets you add custom events to your graphs.

- To manually add an annotation, left-click anywhere in your graph, and click **Add annotation**.
- Describe what you did, and optionally add tags for more context.

Let your team know that you did some testing for a while, by clicking and dragging an interval, while pressing Ctrl (or Cmd on macOS).

Instead of manually annotating your dashboards, you can tell Grafana to get annotations from a data source.

- Select **Dashboard settings** from the top of the dashboard view.
- Click **Annotations**, then **New Annotation Query**.
- In the **Name** box, type "Errors".
- Select "Loki" from the **Data source** drop down.
- In the **Query** box, type a LogQL query:

```
{filename="/var/log/tns-app.log"} |= "error"
```

- Click **Add** and go back to your dashboard.

The log lines returned by your query are now displayed as annotations in the graph.

{{% /tutorials/step %}}
{{% tutorials/step title="Build a RED dashboard" %}}

RED, or Rate, Errors, and Duration, is a method for monitoring services. Let's create a RED dashboard for our sample application.

In the last exercise, you created a panel to visualize the _Rate_ of requests, or traffic.

Next, we'll add one for _Errors_, and _Duration_.

- Add another Graph panel for _Errors_:

```
sum(irate(tns_request_duration_seconds_count{status_code!~"2.."}[5m]))
```

- Add a third Graph panel to display _Duration_:

```
histogram_quantile(0.99, sum(irate(tns_request_duration_seconds_bucket[5m])) by(le))
```

To be able to troubleshoot any errors, let's add a logs panel to our dashboard:

- Create another panel with a Logs visualization.
- In the **Query** settings, select the "Loki" data source, and enter the query:

```
{filename="/var/log/tns-app.log"}
```

- Go back to you dashboard. With the current dashboard, we can quickly see when an error occurred, and what may have caused it.

{{% /tutorials/step %}}
{{% tutorials/step title="Alerting" %}}

For this exercise, you'll be sending alerts using Webhooks. Before you can do that, you need to set up a _request bin_:

- Browse to [requestbin.com](https://requestbin.com)
- Unselect **Private (requires log in)**, and click **Create Request Bin**.
- You request bin is now waiting for the first request.
- Copy the endpoint URL.

Next, you'll configure a _notification channel_.

- In the side bar, click **Alerting** -> **Notification channels**.
- Click **Add channel**.
- In the **Name** box, type RequestBin.
- In the **Type** box, select "webhook".
- In the **Url** box, paste the endpoint to your request bin.
- Click **Send Test** to send a test alert to your request bin.

Now that Grafana knows how to notify you, it's time to set up an alert.

- Go to the dashboard you just created, and **Edit** the Traffic panel.
- Click the bell icon to the left of the panel editor to access the settings for alerting.
- In the **Name** box, type "My alert".
- In the **Evaluate every** box, type "5s".
- In the **For** box, type "0m".
- Change the alert condition to:

```
WHEN last() OF query(A, 1m, now) IS ABOVE 0.2
```

This condition will evaluate whether the last value of any of the time series from one minute back is more than 0.2 requests per second.

- Under **Notification**, click the plus sign (+) next to **Send to**.
- Select the notification channel you configured earlier.
- Go back to the dashboard, and save the changes you've made.

You now have an alert set up to send a notification using a web hook. See if you can trigger it by generating some traffic on the sample application.

- Browse to [localhost:8081](http://localhost:8081), and refresh the page repeatedly to generate traffic.

When Grafana triggers the alert, it sends a request to the web hook you set up earlier.

{{% /tutorials/step %}}
{{% tutorials/step title="Congratulations" %}}

Congratulations, you made it to the end of this tutorial!

{{% /tutorials/step %}}