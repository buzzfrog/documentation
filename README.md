*(insert logotype)*

**Yanzi Smart Map** is a solution for visualizing Yanzi sensor data. It colorizes individual
rooms in a floor plan and allows interaction with the rooms.

Yanzi Smart Map collects, stores and interpret the provided sensor data and presents the data
graphically. The data model links individual sensors to a room and to components in that room
(e.g. chairs or tables). The sensors provides either climate data (temperature and humidity) or
utilization data (usage of components in the room).

Depending on what kind of data is to be presented, the graphical presentation is in form of
diagrams (e.g. long term use of a conference room) or a map over the facilities (e.g. air quality
in different rooms). Real time usage of components in the room as well ass current temperature
and humidity is displayed as well as diagrams over historical conditions.

# Table of content
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
        - [GUI](#id-gui)

# Users guide <a name="usersguide"></a>

*Insert screenshots and caption for the different views*

# Developers guide <a name="developersguide"></a>

## Solution Overview <a name="solutionoverview"></a>

*Insert component schema*

The data flow travels from left to right in the visualization of the various system components.
All sensor data is sent from **Yanzi Cirrus API** via an adapter receiving data from the web
socket used by Cirrus, in order to send it forward into the **Microsoft Azure Event Hub**. The
Event Hub receives the data and passes it forward.

**Microsoft Azure Stream Analytics** receives the data from the Event Hub. In order for the data
provided from Yanzi to be able to get processed by Yanzi Smart Map, it needs to be converted to a
suiting format. This will be taken care of by Stream Analytics.

Yanzi Smart Map processes both real-time data and historical data. Therefore there are two output
streams from Stream Analytics where the information is sent via the Event Hub as real-time
information and also stored in a database in order to contribute to the historical overview.

The **ASP.NET** application takes both real-time and historical data and makes it available for the
user to see through the graphical interface.

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