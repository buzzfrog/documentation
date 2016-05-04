*(insert logotype)*

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
- [Users Guide](#id-usersguide)
- [Developers Guide](#id-developersguide)
    - [Solution Overview](#id-solutionsoverview)
    - [Components](#id-components)
        - [Yanzi Cirrus API](#id-cirrus)
            - [Overview](#id-cirrusoverview)
            - [Essentials](#id-yanziessentials)
            - [Further Readings](#id-yanzifurtherreadings)
        - [Adapter](#id-adapter)
        - [Microsoft Azure Event hub](#id-eventhub)
        - [Microsoft Azure Stream Analytics](#id-streamanalytics)
            - [Overview](#id-streamanalyticsoverview)
            - [Real Time Data](#id-streamanalyticsrtdata)
            - [Historical Data](#id-streamanalyticshdata)
            - [Setup Guide](#id-streamanalyticssetupguide)
            - [Further Readings](#id-streamanalyticsfurtherreadings)
        - [asp.NET](#id-aspnet)
            - [Overview](#id-aspnetoverview)
            - [Real Time Data](#id-aspnetrtdata)
            - [Historical Data](#id-aspnethdata)
            - [Setup Guide](#id-aspnetsetupguide)
            - [Further Readings](#id-aspnetfurtherreadings)
        - [GUI](#gui)

# User's Guide <a name="usersguide"></a>

![Image](images/smartmap-1.png?raw=true)
*All rooms on a floor are displayed on the map as seen in the screenshot above.*

![Image](images/smartmap-2.png?raw=true)

*When a user clicks on a room its historical data is shown in a graph together with room details.*

# Developer's Guide <a name="developersguide"></a>

## Solution Overview <a name="solutionoverview"></a>

*Insert component schema*

**In short, data flows from left to right in the system schema pictured above.**

All sensor data is retrieved from *Yanzi Cirrus API* via an adapter listening in on data pushed over the WebSocket used by Cirrus. All incoming sensor data, regardless of type, is directly forwarded to *Azure Event Hub*.
The Event Hub buffers data until it is processed by the next component in the pipeline.

An *Azure Stream Analytics* job receives the data from the Event Hub. In order for the data
provided from Yanzi to be able to get processed by Yanzi Smart Map, it needs to be converted to a
suitable format. This is taken care of by Stream Analytics.

After data is converted it is fed into one of the two outputs. Every data point is transmitted to an Azure Service Bus Queue (for real time updates to the frontend) and contributes to an hourly average value which is stored in the SQL database. This gives the solution both real time data updates and historical average values.

An *ASP.NET* web application handles the task of making both real time and historical data available in such a format that it can be queried or listened to using HTTP and WebSockets.

It provides an HTTP JSON RESTful API, backed by SQL, where static map data (buildings, floors, rooms) and historical averages can be queried. It also allows for WebSocket connections where every data point put in the Service Bus Queue is broadcasted to all connected clients.

## Components <a name="components"></a>

### Yanzi Cirrus API <a name="cirrus"></a>

#### Overview <a name="cirrusoverview"></a>

Cirrus is the API provided by Yanzi which provides sensor and event data using WebSockets.

#### Essentials <a name="yanziessentials"></a>

Access and other necessities regarding Cirrus is provided by [Yanzi](http://www.yanzinetworks.com/).
This solution uses the simplified Yanzi Cirrus API known as *Shrek* with basic authentication instead of certificates.

#### Further Readings <a name="yanzifurtherreadings"></a>

For technical documentation contact [Yanzi](http://www.yanzinetworks.com/).

<hr>

### Adapter <a name="adapter"></a>

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

### Azure Event hub <a name="eventhub"></a>

#### Overview

Azure Event Hub buffers events until they are processed by a consumer. For more information about Event Hub
[Microsoft Azure Technical documentation](https://azure.microsoft.com/documentation/services/events-hubs/).

#### Essentials

Setting up the Event Hub is done in a few steps in Azure. Create a Service Bus with an Event Hub. Make sure your
Event Hub has a connection string with permission to send events into the hub.

#### Setup Guide

*Insert configuration tutorial [Emma]*

#### Further Readings

[Technical documentation](https://azure.microsoft.com/documentation/services/events-hubs/)

<hr>

### Azure Stream Analytics <a name="streamanalytics"></a>

#### Overview <a name="streamanalyticsoverview"></a>

Stream Analytics processes incoming data and transforms it form Cirrus's message format to a custom data model.

When receiving data from the sensors, it is important that the information is displayed in real
time as well as being saved in the database.

Since the solution processes both real time and historical data, it requires two output streams from Stream
Analytics.

#### Essentials

##### Real Time Data  <a name="streamanalyticsrtdata"></a>

When Stream Analytics receives data it is transformed to the custom data model and then pushed to a Service Bus
Queue (later used for real time updates of the GUI).

##### Historical Data <a name="streamanalyticshdata"></a>

Every data point received by Stream Analytics contributes to an hourly average data value for each sensor
(TumblingWindow). Each calculated average value is stored in SQL.

#### Stream Analytics Query <a name="streamanalyticssetup"></a>

*Insert configuration tutorial [Emma]*

#### Further Readings <a name="streamanalyticsfurtherreadings"></a>

[Technical documentation](https://azure.microsoft.com/sv-se/documentation/services/stream-analytics/)

<hr>

### ASP.NET <a name="aspnet"></a>

#### Overview

ASP.NET takes information provided from the data layer and makes it available for the graphic user
interface using HTTP and WebSockets.

#### Essentials

##### Real Time Data  <a name="aspnetrtdata"></a>

In order to push new data in real time a WebSocket is used between the ASP.NET
application and the GUI. Every real time data point received from the data layer is broadcasted to all connected
clients.

##### Historical Data <a name="aspnethdata"></a>

Both historical sensor data (hourly averages) and static map data (available buildings, floors and rooms) can be
fetched using an HTTP JSON RESTful API, backed by SQL. A [detailed API reference can be read at the root URL](
http://api20160426022719.azurewebsites.net/) of a deployed version of the application.

#### Setup Guide <a name="aspnetsetupguide"></a>

The application is an ASP.NET Core (also known as ASP.NET 5) web application. For setup instructions please reference the [ASP.NET Core documentation](https://docs.asp.net/en/latest/getting-started/).

Configure the application by creating a static class `Config.cs` in the `src/api` directory of the solution. This file must contain two strings: `SqlConnectionString` and `QueueConnectionString`, containing your SQL Server connection string and a Service Bus Queue connection string with read permission. Run the application using `dnx web`.

#### Further Readings <a name="aspnetfurtherreadings"></a>

* [WebSockets in ASP.NET Core](https://docs.asp.net/en/latest/fundamentals/owin.html#run-asp-net-5-on-an-owin-based-server-and-use-its-websockets-support)
* [Entity Framework Core](http://docs.efproject.net/en/latest/)
* [Dependency Injection ](https://docs.asp.net/en/latest/fundamentals/dependency-injection.html#registering-your-own-services)(the real time component is a Service registered with `AddInstance to allow for continuous operation)

<hr>

### Frontend (Graphical User Interface) <a name="gui"></a>

#### Overview

Presents an interactive indoor room map, color coded based on sensor data.

#### Essentials

The frontend which end-users interact with is a React web application that renders a map of a building floor plan.
All rooms on the map are color coded based on current sensor data values, and can be interacted with by the user.

Data updates are pushed to the frontend using a WebSocket connection. This allows for real time color updates of the rooms on the map. When a user interacts with a given room the historical data endpoint of the ASP.NET API is queried. Historical data is presented in a graph.

#### Setup Guide

Setup and run the frontend using `npm install` and `npm start`.
