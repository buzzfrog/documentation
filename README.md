![Image](images/logo.png?raw=true)

**Yanzi Smart Map is a solution for visualizing Yanzi sensor data. It colorizes individual
rooms in a floor plan and allows interaction with the rooms.**

Yanzi Smart Map collects, stores and interprets provided sensor data and presents it graphically.
The data model links individual sensors to a room. The sensors provide either climate data
(temperature and humidity) or utilization data (e.g. percentage of chairs occupied in a conference room).

Depending on what kind of data is to be presented, the graphical presentation is in form of
diagrams (e.g. long term use of a conference room) or a map over the facilities (e.g. air quality
in different rooms). Real time usage of components in the room as well as current temperature
and humidity is displayed together with diagrams over historical conditions.

# Table of contents
- [User's Guide](#usersguide)
- [Developer's Guide](#developersguide)
    - [Solution Overview](#solutionoverview)
    - [Components](#components)
        - [Yanzi Cirrus API](#cirrus)
        - [Adapter](#adapter)
        - [Azure Event Hub and Service Bus Queue](#eventhub)
        - [Azure Stream Analytics](#streamanalytics)
        - [ASP.NET (Backend)](#aspnet)
        - [Frontend](#gui)

<a name="usersguide"></a>
# User's Guide

![Image](images/smartmap-1.png?raw=true)
*All rooms on a floor are displayed on the map as seen in the screenshot above.*

![Image](images/smartmap-2.png?raw=true)

*When a user clicks on a room its historical data is shown in a graph together with room details.*

<a name="developersguide"></a>
# Developer's Guide

<a name="solutionoverview"></a>
## Solution Overview

![Image](images/system_map.png?raw=true)

**In short, data flows from left to right in the system schema pictured above.**

All sensor data is retrieved from *Yanzi Cirrus API* via an adapter listening in on data pushed over the WebSocket used by Cirrus. All incoming sensor data, regardless of type, is directly forwarded to *Azure Event Hub*.
The Event Hub buffers data until it is processed by the next component in the pipeline.

An *Azure Stream Analytics* job receives the data from the Event Hub. In order for the data
provided from Yanzi to be able to get processed by Yanzi Smart Map, it needs to be converted to a
suitable format. This is taken care of by Stream Analytics.

After data is converted it is fed into one of the two outputs. Every data point is transmitted to an Azure Service Bus Queue (for real time updates to the frontend) and contributes to an hourly average value which is stored in the SQL database. This gives the solution both real time data updates and historical average values.

An *ASP.NET* web application handles the task of making both real time and historical data available in such a format that it can be queried or listened to using HTTP and WebSockets.

It provides an HTTP JSON RESTful API, backed by SQL, where static map data (buildings, floors, rooms) and historical averages can be queried. It also allows for WebSocket connections where every data point put in the Service Bus Queue is broadcasted to all connected clients.

<a name="components"></a>
## Components

<a name="cirrus"></a>
### Yanzi Cirrus API

#### Overview

Cirrus is the API provided by Yanzi which provides sensor and event data using WebSockets.

#### Essentials

Access and other necessities regarding Cirrus is provided by [Yanzi](http://www.yanzinetworks.com/).
This solution uses the simplified Yanzi Cirrus API known as *Shrek* with basic authentication instead of certificates.

#### Further Readings

For technical documentation contact [Yanzi](http://www.yanzinetworks.com/).

<hr>

<a name="adapter"></a>
### Adapter

#### Overview

Pushes sensor and event data from Cirrus to an Azure Event Hub.

#### Essentials

The adapter takes the following actions:

1. Connect and authenticate to Cirrus
2. Subscribe to events from a given Cirrus location ID
3. Pass all incoming messages to an Azure Event Hub
4. Reconnect to Cirrus if connection drops

#### Setup guide

All necessary configuration needed to run the adapter is located in the file `Connection.config` located in the root
folder of the [adapter solution](https://github.com/mvk2016/adapter):

```xml
<?xml version="1.0" encoding="utf-8" ?>
<appSettings>
  <add key="YanziHost" value="wss://cirrusbeta.yanzi.se/cirrusAPI" />
  <add key="YanziUser" value="user@email.com" />
  <add key="YanziPass" value="password" />
  <add key="YanziLocation" value="871073" />
  <add key="SourceHubConnectionString" value="Endpoint=sb://myeventhub-ns.servicebus.windows.net/;SharedAccessKeyName=SendRule;SharedAccessKey={...}" />
  <add key="SourceHubName" value="myeventhub" />
</appSettings>
```
Note: Make sure `Connection.config` is copied to your output directory on compile.
Run the adapter as you would any C# solution in Visual Studio.

<hr>

<a name="eventhub"></a>
### Azure Event Hub and Service Bus Queue

#### Overview

Azure Event Hub and Service Bus Queue are connecting Azure services to each other and to outside consumers and producers. They both buffer events until they are processed by a consumer.

#### Essentials

An Event Hub is used as the connection between the Cirrus Adapter and Stream Analytics, and a Service Bus Queue forwards the Stream Analytics results to the ASP.NET web application.

#### Setup Guide

Setting up the Event Hub and Service Bus Queue is done in a few steps in Azure.
First quick create a Service Bus with a Queue and an Event Hub.
They are contained within a Service Bus namespace, and at this step a new namespace can be created.

To be able to send message to, and receive messages from the Event Hub and Queue, shared access policies need to be configured as the image below shows.

![Image](images/shared_access_policies.png?raw=true)

#### Further Readings

[Technical documentation on Event Hubs](https://azure.microsoft.com/en-us/documentation/services/event-hubs/)

[Technical documentation on Service Bus Queues](https://azure.microsoft.com/en-us/documentation/services/service-bus/)

<hr>

<a name="streamanalytics"></a>
### Azure Stream Analytics

#### Overview

Stream Analytics processes incoming data and transforms it from Cirrus's message format to a custom data model.

When receiving data from the sensors, it is important that the information is displayed in real
time as well as being saved in the database.

Since the solution processes both real time and historical data, it requires two output streams from Stream
Analytics.

#### Essentials

##### Real Time Data

When Stream Analytics receives data it is transformed to the custom data model and then pushed to a Service Bus
Queue (later used for real time updates of the GUI).

##### Historical Data

Every data point received by Stream Analytics contributes to an hourly average data value for each sensor
(TumblingWindow). Each calculated average value is stored in SQL.

#### Stream Analytics Query

```sql
with
-- Contains a timestamped sample list
SampleList as (
    select
        SampleList.ArrayValue as SampleList,
        -- Asset utilization data has an EventEnqueuedUtcTime
        -- which is a better timestamp than timeSent
        c.EventEnqueuedUtcTime as EventTime
    from Cirrus as C
    -- Convert from Unix to Microsoft time
    timestamp by DATEADD(millisecond, timeSent, '1970-01-01T00:00:00Z')
    cross apply GetArrayElements(C.list) as SampleList
),
-- Contains each individual sample
SampleInfo as (
    select
        Sample.SampleList.dataSourceAddress.did as SensorId,
        Sample.SampleList.dataSourceAddress.variableName.name as Type,
        GetArrayLength(Sample.SampleList.list) as SampleLength,
        Data.ArrayIndex as SampleIndex,
        Data.ArrayValue.value as Value,
        dateadd(millisecond, Data.ArrayValue.SampleTime, '1970-01-01T00:00:00Z') as SampleTime,
        Sample.EventTime,
        Rooms.RoomId

    from SampleList Sample
    left outer join Rooms

    -- on substring(Sample.SampleList.dataSourceAddress.did,1,len(R.SensorId)) = R.SensorId
    -- Because Stream Analytics wouldn't accept the line above
    -- using a case-statement is an alternative solution
    -- This is to remove the "-temp", "-humd" etc at the end of the unit address
    on (case substring(Sample.SampleList.dataSourceAddress.did,1,1)
            -- If did starts with 'E' it's a unit
            when 'E' then substring(Sample.SampleList.dataSourceAddress.did,1,22)
            -- If did starts with 'U' it's an asset
            when 'U' then Sample.SampleList.dataSourceAddress.did
            else ''
    end) = Rooms.SensorId
    cross apply GetArrayElements(Sample.SampleList.list) as Data
    -- To make sure data from unwanted locations are removed
    where Sample.SampleList.dataSourceAddress.locationId = '871073'
),
-- Contains only the samples of the Types we're interested in
Data as (
    select
        SensorId, RoomId,
        -- Rename some sample Types
        case Type
            when 'temperature' then 'temperature'
            when 'relativeHumidity' then 'humidity'
            when 'percentage' then 'utilization'
            else null
        end as Type,
        -- Convert temperature from Kelvin to Celsius
        case Type
            when 'temperature' then Value - 273.15
            else Value
        end as Value,
        SampleIndex, SampleLength,
        -- Use different timestamps depending on sample Type
        case Type
            when 'temperature' then SampleTime
            when 'relativeHumidity' then SampleTime
            when 'percentage' then EventTime
            else null
        end as Collected

    from SampleInfo
    -- Sort out sample Types of interest
    where (Type = 'temperature' or Type = 'relativeHumidity' or Type = 'percentage')
),
-- Contains samples only if it was the latest in a message from Cirrus
-- (i.e. the last in a sampleList in a message)
CurrentData as (
    select *
    from Data
    where SampleLength - 1  = SampleIndex
),
-- Aggregates samples in a tumbling window of one hour
AggregatedData as (
    select avg(Value) as Value, Type, SensorId, RoomId, System.TimeStamp as Collected
    from Data -- timestamp by Collected
    group by TumblingWindow(minute, 60), Type, SensorId, RoomId
)

select SensorId, Type, value, Collected, RoomId into backendqueue from CurrentData
select SensorId, Type, value, Collected, RoomId into mvkdatabase from AggregatedData


```

#### Further Readings

[Technical documentation](https://azure.microsoft.com/documentation/services/stream-analytics/)

<hr>

<a name="aspnet"></a>
### ASP.NET Backend

#### Overview

ASP.NET takes information provided from the data layer and makes it available for the graphic user
interface using HTTP and WebSockets.

#### Essentials

##### Real Time Data

In order to push new data in real time a WebSocket is used between the ASP.NET
application and the GUI. Every real time data point received from the data layer is broadcasted to all connected
clients.

##### Historical Data

Both historical sensor data (hourly averages) and static map data (available buildings, floors and rooms) can be
fetched using an HTTP JSON RESTful API, backed by SQL. A [detailed API reference can be read at the root URL](
http://api20160426022719.azurewebsites.net/) of a deployed version of the application.

#### Setup Guide

The application is an ASP.NET Core (also known as ASP.NET 5) web application. For setup instructions please reference the [ASP.NET Core documentation](https://docs.asp.net/en/latest/getting-started/).

Configure the application by creating a static class `Config.cs` in the `src/api` directory of the solution. This file must contain two strings: `SqlConnectionString` and `QueueConnectionString`, containing your SQL Server connection string and a Service Bus Queue connection string with read permission. Run the application using `dnx web`.

#### Further Readings

* [WebSockets in ASP.NET Core](https://docs.asp.net/en/latest/fundamentals/owin.html#run-asp-net-5-on-an-owin-based-server-and-use-its-websockets-support)
* [Entity Framework Core](http://docs.efproject.net/en/latest/)
* [Dependency Injection ](https://docs.asp.net/en/latest/fundamentals/dependency-injection.html#registering-your-own-services)(the real time component is a Service registered with `AddInstance to allow for continuous operation)

<hr>

<a name="gui"></a>
### Frontend (Graphical User Interface)

#### Overview

Presents an interactive indoor room map, color coded based on sensor data.

#### Essentials

The frontend which end-users interact with is a React web application that renders a map of a building floor plan.
All rooms on the map are color coded based on current sensor data values, and can be interacted with by the user.

Data updates are pushed to the frontend using a WebSocket connection. This allows for real time color updates of the rooms on the map. When a user interacts with a given room the historical data endpoint of the ASP.NET API is queried. Historical data is presented in a graph.

#### Setup Guide

Setup and run the frontend using `npm install` and `npm start`.
