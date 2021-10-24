# Final project in OS Workshop, adding ELK Logging and Testing to eVSSIM 

# Overview

**eVSSIM Simulator** simulates software-based SSD. One of it's purposes is it to research SSD behavior under different use-cases. 

To enable this research we want to be able to observe its inner action, we need to log its actions. 
Currently eVSSIM contains Real Time analyzer which is able to shows the current status of the simulator from various perspectives. 

The next level of visibility into the simulator operation is aggregating the logs and conveniently compose dashboards for asking questions about the specific operations. Another important goal is to save data over time in order to be able to do time-series analysis. 

The project aims to write logs in JSON format, describing main actions in the simulator, index them to Elastic Search and visualize the logs with Kibana.

This will allow us to ask questions about specific actions and over-time behavior and observe basic dashboards that we will provide. 

## Project components:
* In-line simulator code that write logs in JSON format of each actions. Log types are defined [here](https://github.com/davidsaOpenu/simulator/blob/master/eVSSIM/LOG_MGR/logging_parser.h). 
*  Index logs to ELK using Filebeat
*  Built-in dashboards
*  Standalone log generator for testing
*  Testing framework to validate dashboard computations, for example: aggregations over time

## Writing Logs to file
TODO

## Event Types:
Defined [here](https://github.com/davidsaOpenu/simulator/blob/master/eVSSIM/LOG_MGR/logging_parser.h)
1. **PhysicalCellReadLog** - Physical read of cell. Parameters are channel, block, page(amount of pages read) and the time. 
2. **PhysicalCellProgramLog** - Physical write of cell. Parameters are channel, block, page(amount of pages writing) and the time. 
3. **LogicalCellProgramLog** - Logical log, which sums all actions need to be done for writing(switching channel , actual write). This log is probably unnecessary and would be probably deprecated. 

	Currently used for calculating Write Amplification which represents the ratio between logical and physical writes. 

	When the SSD is more efficient this ration would be smaller and there would be less physical writes for each logical write. Parameters are the same as former logs. 
4. **RegisterReadLog** - Change register to read mode. Parameters are channel, die(the flash we are writing), reg(index of the register) and the time. 
5. **RegisterWriteLog** - Change register to read mode, same parameters as former log. 
6. **BlockEraseLog** - Erasing block from disk, same parameters as former log. 
7. **ChannelSwitchToReadLog** - Change channel state to read, parameters are channel index and time. 
8. **ChannelSwitchToWriteLog** - Change channel state to write, parameters are the as former log. 

### Disk structure 
flash is a die, each flash contains channels which allow parallel access to planes (in hardware).
  each channel contains blocks and pages. each register and channel can be in read or write mode. 

as can be seen in the following diagram: 
![Simulator Overview](images/ssd_scheme.png)

[Reference](https://www.researchgate.net/publication/261391212_VSSIM_Virtual_machine_based_SSD_simulator)

In all of the logs we are using time in resolution of milliseconds. 

Every action causes configurable delay to the simulator in order to be realistic, example configuration is [here, lines 98-193](https://github.com/davidsaOpenu/simulator/blob/master/eVSSIM/tests/host/base_emulator_tests.h)

## ELK Stack
First things first, let's install all the related projects that we will later use. 

### Installation
First things first, small technical intro of our usage of the The ELK stack. It consists of 4 main projects: 

1. **ElasticSearch** - Text DB - Based on indexing text, not structured data as other SQL, enables text and structured search. Uses different indexes for storing data. 
2. **Kibana** - Visualization framework, enables to easily build dashboards to visualize our data 
3. **Logstash** - internal framework for indexing logs to ElasticSearch. 
4. **Filebeat** - Monitor directory for log files and upload logs to ElasticSearch. 

We will install everything possible as container, for ELK we'll use (this project)[https://github.com/deviantony/docker-elk], we won't change anything in the configuration except Memory allocated for the containers(512mb instead of 256mb). 

Kibana configuration would be exported JSON file which contains the dashboards, which were built using Kibana's' web interface. 

I had many problems with the configurations of the containers, if something needs to be changed, do it carefully. Mostly on network related configurations, because all the containers need to connect each other. 

In the containers we are using the Volume feature, which gives as the option to map host directory to fixed location in the container, so the project could run from every directory on the host. 

One thing to note in Filebeat configurations is the time field. We want ElasticSearch index our logging_time field as a native time. All of the aggregations and searches are time based and it's critical for it be indexed as native time. Also pay attention to the TimeZone, i've used Israels' TZ, change it if needed. 

### Dashboards
Those are the dashboards we currently built with Kibana. 

To see it live, please browse to [http://localhost:5601](http://localhost:5601).
Adding or changing dashboards is very easy, Kibana is well documented and there are plenty of features. 

The dashboards were built as standard dashboards, mostly using built-in  [aggregations features](https://www.elastic.co/guide/en/kibana/current/add-aggregation-based-visualization-panels.html) and [Timelion dashboards](https://www.elastic.co/guide/en/kibana/current/timelion.html), which are more compatible to visualize data over time.

The dashboards themselves are self-explanatory, the example data was generated from the simulator with some simple read and write traffic. 

![](images/cell_rw_dashboard.png)

![](images/blocks_dashboard.png)

![](images/channels_switch_dashboard.png)

![](images/blocks_dashboard.png)

![](images/write_amplification_disk_utilization.png)

The user can filter specific channels and blocks in this panel: 

![](images/user_selection.png)

In order to monitor this system, there is a monitoring dashboard that shows how many logs were indexed over time, and the amount of logs in total and from each type, in the selected time: 

![](images/general_dashboard.png)

## Testing Framework

After indexing and visualizing all this data, we need to verify all of the computations the Kibana made for us are correct. Example for a computation would be sum of writes in each minute in each channel. 

Probably not many from the ELK developers intended that someone will **actually** verify the dashboards correctness, and theres' no simple API to download dashboard data. 
So we've developed a framework to do that. 

I've reached to elastic developers and they said the the process of exporting data from dashboard is fully client-side, so it's not that simple to develop API for that task. But still theres' [Feature request](https://github.com/elastic/kibana/issues/30982) for that from 2019, so it will definitely come up in the following versions of Kibana. Until then, you can use this framework :)

### General purposes 
In order to verify the aggregation we will need to calculate in separately from Kibana, on the original log file, and compare the results. This calculation would be made with [Pandas](https://pandas.pydata.org/docs/getting_started/index.html) python module. 

Pandas is pretty good with tabular data and would make the aggregations for us. 

To get the results from kibana we will scrape it using [Selenium](https://www.selenium.dev/selenium-ide/)(Framework for browser automation and user simulation, mostly used for automated Web testing).

We will use container of selenium and Firefox browser, reasons are detailed in the technical issues section. 

We will use selenium for scraping Kibana web page.
 
Disadvantage of that is the we are using hard-coded HTML tags, which can **vary between versions and are not supposed to be compatible.** 

Our main testing script will run in a container named `automated_testing`.

This scraping can made only to standard Kibana dashboard, that have "Export Dashboard as CSV" option, not for Timelion dashboards. There we will need to do a different solution. 

I've checked the last 3 major versions of Kibana and they didn't changed. 


**So our whole testing process would be**:

```wrap
Installing ELK Stack and filebeat.
	Using the custom configuration for each project. 
	We will wait Kibana will finish its setup.
2. Index the log to ElasticSearch using Filebeat. 
	Wait until the log file is indexed. 
	By comparing number of logs in Kibana from this file, by it's name, 
	to number of lines in the files.  
3. Computing the same aggregations on the original log file with Pandas. 
4. Download the dashboard data as CSV, copy to the container runs the test itself. 
5. Execute our tests (in a separate container)
6. Compare the results and exit the script if one of them isn't equal
	   (using python assert). 
```

### Kibana automation Framework

**The internal process for downloading dashboard as CSV would be:**


1. Assuming ELK installed and the log is indexed,
   we will browse to kibana and login, using default credentials. 
2. For each dashboard,We will do the following process: 
	1. find the options button, the 3 dots button, by this HTML Tag: 
```wrap
	button[@aria-label='Panel options for'  + dashboard_name
```
3. We would locate the "More" button to get more options, 
   by this HTML Tag: button[@class='euiContextMenuItem'][.='More']
4. We would click the Download as CSV, by clicking the button by this HTML Tag: 
```wrap
   button[@data-test-subj='embeddablePanelAction-ACTION_EXPORT_CSV']
``` 


The functionality of exporting a Dashboard as CSV is contained in the class `DashboardExporter`, which is part of the main `SeleniumUtils` which contains all the Selenium related tasks, which are: 

1. Creating remote selenium driver with the desired permissions to download files
2. Login to Kibana and the download of dashboard as CSV and copying this CSV file to the host. 

**Prior assumptions hard-coded in the code:**

1. Dashboard names inside kibana and all of the parameters names and time interval(currently 100ms)
2. Time format of the log `2021-07-04T15:14:47.000Z`
2. Kibana is running at localhost:5601
3. Kibana and Elasticsearch use default username and password(elastic:chanemge)
3. Log names(in the Type field)
4. URL of the dashboard(times can be changed)

The tests were implemented for the first 3 dashboards: Block Writes, Page Reads, and Page Writes.

## Demo time

Let's see it all in action!

Every `Passed` print is a successful test 

![Demo](images/Demo.gif)

## Technical issues encountered in developing the Testing framework:

* Files download automation - First i've tried to used chrome browser with remote selenium driver. Chrome browser has security features that blocks downloading files by automation frameworks, as kibana. There were some configuration that we're supposed to disable this feature, but it's not officially documented. Finally it ended switching to Firefox.

* Files download permissions - The browser needs to have permissions to save files in the download directory. From strange reasons, Firefox didn't had permissions to save files in the default download directory. Using some brute-force, i've found it had permissions to save files to the default user directory. Because it's a container of selenium, the user is `seluser`. But, the browser was able to download there files, just went it wasn't mounted to the host as a volume, so i've needed to find a way to copy the files to the host in order to pass them to the other container.  

* Copying files between containers - there are 2 components of the project - the Kibana container, Selenium container and a container which runs the testing comparison script. So the problem is to copy a file created in the Selenium container to the container which runs the testing script, in order it to use it for comparing the results. The standard solution for this it to use a [docker volume](https://docs.docker.com/storage/volumes/), which is a shared directory between the host and the container, but from unknown reasons it didn't worked with the Selenium container. The solution was to use docker `cp` command to copy the files from the container to host machine, but with default security premissions it won't work, becuse the copy command works only from the host. So we need to give a container permission to run shell commands on it's host. That's a bad practice from security perspective, but the only solution worked for me in this issue. If a container runs with `docker run  --privileged --pid=host` arguments, it will have permissions to run shell commands on the host running the container. 
	Inside the container we will use:
 `nsenter -t 1 -m -u -n -i sh -c "docker cp " + Selenium_container_id + ":/home/seluser " + Host_CSV_Exports_Directory "`. [credit](https://medium.com/lucjuggery/a-container-to-access-the-shell-of-the-host-2c7c227c64e9).
 
	We will find `Selenium_container_id ` in the same method, parsing `docker ps` output and looking for Selenium-firefox container. 

* HTML elements dynamic loading - Some of the HTML Elements in the Kibana web UI are loading dynamically, according the user's click and dashboard types. The `Options` and `More` buttons are loaded dynamically according to user's click on them and not automatically. Because of that we can't use Selenium best practice which is `WebDriverWait(self.driver,10).until(EC.presence_of_element_located((By.XPATH,'Button_XPATH')))` and [wait until it loaded after some time](https://www.selenium.dev/documentation/webdriver/waits/). 

To wait for those elements to load we will use `Sleep` until they load. It could be a problem if you're loading dashboard with lot's of Data(More than 200K Logs), depends on your setup performance, the default sleep would be enough and the test would file just because the UI didn't finish loading, and not because the data isn't correct. The default sleep time is 10 seconds. 
* Firefox scrolling - Firefox Selenium is different from Chrome driver, by the fact that it's not searching the full page, it's searches just the current view of the user. So, if the button you're searching isn't in the current view and you need to scroll up or down to find it, Firefox Selenium driver doesn't scrolls automatically. So we've implemented small function, `scroll_shim` ,  that runs JS script in the browser context the scrolls to the location of the button that we wanted to click.

## Future work 
* Research questions: the provided dashboards here are just the tip of the iceberg, you can easily build much more dashboards and ask interesting questions regarding the logs and explain phenomenons in SSD.
* Dynamic inputs - It can be useful to see how a certain configurations or traffic type affects the results, so you can expand this logging framework to do so. 
* Testing framework - The provided framework can be expanded to more dashboards' and logics that you can validate using it. 
* Add more logs - new logs can be easily added, keep the general format of type and time and it would be simple. 
* Visualization framework - For this project i've choose Kibana, but also checked Grafana. I haven't found major differences between them for this project's purposes. Kibana has features like text-search which isn't possible in Grafana, but is not needed for the current analytics. [Grafana has more visualizations features](https://www.metricfire.com/blog/grafana-vs-kibana/), but for the current visualization needs, Kibana has all the necessary features.  Grafana can be easily connected to the same ElasticSearch and run as container. I've used `docker run -d -p 3000:3000 --name grafana grafana/grafana:6.5.0`, configured Elastic data source, this is an [example dashboard](images/grafana.png).
 