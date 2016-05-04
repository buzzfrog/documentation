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

*Insert screenshots and caption for the different views*

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

Cirrus is the API provided by Yanzi which carries out the necessary information about the sensors using websockets.

#### Essentials <a name="yanziessentials"></a>

Access and other necessities regarding Cirrus is provided by [Yanzi](http://www.yanzinetworks.com/).

#### Further Readings <a name="yanzifurtherreadings"></a>

[Technical documentation](http://www.yanzinetworks.com/pdf/product-brief/product-brief-890-07065-cirrus-iot-solution-smart-building-ibm-v01pa3.pdf)

### Adapter <a name="adapter"></a>

#### Overview

Takes the data from the web socket provided by Cirrus and commits it to the Event Hub without
processing it.

#### In depth

#### Setup guide

### Microsoft Azure Event hub <a name="eventhub"></a>

#### Overview

#### In depth

#### Setup Guide

*Insert configuration tutorial [Emma]*

#### Further Readings

[Technical documentation](https://azure.microsoft.com/documentation/services/events-hubs/)

### Microsoft Azure Stream Analytics <a name="streamanalytics"></a>

#### Overview <a name="streamanalyticsoverview"></a>

Stream Analytics processes the data and adjust the format in a way that is compatible with Yanzi
Smart Map, making sure that the data contains the necessary information.

When receiving data from the sensors, it is important that the information is displayed in real
time as well as being saved in the database.

Since the solution processes different types of data, it requires two output streams from Stream
Analytics, one for each data type.

#### Real Time Data  <a name="streamanalyticsrtdata"></a>

When Stream Analytics receives real time data it is transmitted via the Event Hub to a websocket.

#### Historical Data <a name="streamanalyticshdata"></a>

Historical data is saved in a database. Queries for the average values will later be made using the historic data,
displayed as graphs in the graphic user interface.

#### Setup Guide <a name="streamanalyticssetup"></a>

*Insert configuration tutorial [Emma]*

#### Further Readings <a name="streamanalyticsfurtherreadings"></a>

[Technical documentation](https://azure.microsoft.com/sv-se/documentation/services/stream-analytics/)

### ASP.NET <a name="aspnet"></a>

#### Overview

ASP.NET takes information provided from the data layer and makes it available for the graphic user
interface.

#### Real Time Data  <a name="aspnetrtdata"></a>

In order to push new data in real time a websocket is used between the ASP.NET
application and the GUI.

#### Historical Data <a name="aspnethdata"></a>

The historic data is assembled using queries on the database and after
that transfered to the GUI via http-protocol.

#### Setup Guide <a name="aspnetsetupguide"></a>

*Insert configuration tutorial [Mauritz]*

#### Further Readings <a name="aspnetfurtherreadings"></a>

[Technical documentation](https://docs.asp.net/)

### GUI (Graphic User Interface) <a name="gui"></a>

#### Overview

A map of the building in question is displayed, showing every available room and the conditions
for that particular room.

#### In depth

#### Setup Guide