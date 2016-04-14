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

# Graphic interface

*Insert screenshots and caption for the different views*

# Architecture

## Overview

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

## Components

### Yanzi Cirrus API

Cirrus is the API provided by Yanzi which carries out the necessary information about the sensors.

[Technical documentation](insert link here)

### Adapter

Takes the data from the web socket provided by Cirrus and commits it to the Event Hub without
processing it.

### Microsoft Azure Event hub

*Insert configuration tutorial?*

[Technical documentation](https://azure.microsoft.com/documentation/services/events-hubs/)


### Microsoft Azure Stream Analytics

Stream Analytics processes the data and adjust the format in a way that is compatible with Yanzi
Smart Map, making sure that the data contains the necessary information.

When receiving data from the sensors, it is important that the information is displayed in real
time as well as being saved in the database. Queries for the average values will later be made
using the historic data, displayed as graphs in the graphic user interface.

[Technical documentation](https://azure.microsoft.com/sv-se/documentation/services/stream-analytics/)


### ASP.NET

ASP.NET takes information provided from the data layer and makes it available for the graphic user
interface. In order to push new data in real time a web socket is used between the ASP.NET
application and the GUI. The historic data is assembled using queries on the database and after
that transfered to the GUI via http-protocol.

[Technical documentation](https://docs.asp.net/)

### GUI (Graphic User Interface)

A map of the building in question is displayed, showing every available room and the conditions
for that particular room.