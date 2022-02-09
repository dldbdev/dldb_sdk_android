# DLDB android

Welcom to the DLDB SDK beta for android

- [Installation](#Installation)
- [Getting started](#getting-started)
- [Api and examples](#api-and-examples)
- [About DLDB](#about-dldb)
- [Support and contacts](#support-and-contacts)

## Installation

### In android studio

1. add `mavenCentral()` in the `repositories` section of your project `build.gradle`
2. add `implementation 'io.dldb.sdk:dldb-lib:0.9.8'` into dependencies section of your app `build.gradle`
    If your app is already integrating native components
    add `implementation 'io.dldb.sdk:dldb-lib-no-cpp-shared:0.9.8'` into dependencies section of your app `build.gradle`

## Getting started

```kotlin
    // create a unique instance in MainActivity.kt
    private var myDLDB : DLDB = DLDB()

    companion object {
        init {
            System.loadLibrary("dldb-lib")
        }
    }

    // start DLDB SDK after user consent
    myDLDB.init(applicationContext, 
            "11111111-1111-1111-1111-111111111111",
            "{\"button\" : \"t\",\"batteryLevel\" : \"i\"}")

    // once per day
    myDLDB.heartbeat();

    // on a regular basis, when app idle ?
    myDLDB.runQueriesIfAny();

    // wherever useful
    // events only
    myDLDB.addEvents(null, "{\"button\":\"log in\", \"batteryLevel\" : 55 }");

    // location and event
    Location location ...
    myDLDB.addEvents(
        loc 
        eventsAsJson : '{"batteryLevel" : 5 }');

    // location and time and event
    myDLDB.addEvents(0.0, 45.0, 30F, 1644229847, 3600, "{\"button\":\"log in\", \"batteryLevel\" : 55 }")

    // location only
    fusedLocationClient = LocationServices.getFusedLocationProviderClient(this)
    locationCallback = object : LocationCallback() {
        override fun onLocationResult(locationResult: LocationResult?) {
            locationResult ?: return
            for (location in locationResult.locations){
                myDLDB.addEvents(location, null)
            }
        }
    }
```

## Api and examples
  - [init](#init)
  - [heartbeat](#heartbeat)
  - [runQueriesIfAny](#runqueriesifany)
  - [addEvents](#addevents)
  - [queriesLog](#querieslog)
  - [locationsLog](#locationslog)
  - [close](#close)

### init

Initialize the DLDB SDK with the application context, the dldb API key and events dictionary.<br/>
The dldb API key is required and can be obtained from the  [DLDB dashboard](https://dashboard.dldb.io).<br/>

| parameter    | type     | description                               |
| -----------  |----------|------------------------------------------ |
| context         | Context   | application context     |
| dldbApiKey   | String   | api key                                   |
| dictionary  | String   | events dictionary with names and types |

The events dictionary is a json string containing one object per event. Each object defines the event name as a key and the values as the type of events : 't' for text, 'i' for numeric value.
A dictionary `{ "button" : "t", "batteryLevel": "i" }` defines 2 events : `button` with text value, and `batteryLevel` with numeric value.
The dictionary is meant to be a constant and can be changed only with new versions of your app.
The event names defined in this dictionary will be the only events accepted by the function [addEvents]


*Example:*

```kotlin
    private var myDLDB : DLDB = DLDB()

    companion object {
        init {
            System.loadLibrary("dldb-lib")
        }
    }
    ...
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
    ...
    myDLDB.init(applicationContext,
        "11111111-1111-1111-1111-111111111111",
        "{\"button\" : \"t\",\"batteryLevel\" : \"i\"}")
```
---

### heartbeat

to be called on a low frequency basis compatible with the app usage frequency, in order to provide accurate estimates of query response time to the dashboard.

*Example:*

```kotlin
    myDLDB.heartbeat()
```

### runQueriesIfAny

to be called on a regular basis compatible with the app usage frequency, in order to provide fast query responses to the dashboard. The more often it is called the faster the queries will be replied on dashboard

*Example:*

```kotlin
    myDLDB.runQueriesIfAny();
```

### addEvents

| parameter          | type       | description                                   |
| -----------------  |------------|------------------------------------------     |
| location           | Location | obtained from LocationManager, null if not available               |
| longitudeInDegrees | double     | in decimal degrees                            |
| latitudeInDegrees  | double     | in decimal degrees                            |
| horizontalAccuracyInMeters  | float | horizontal accuracy in meters aka the radius of the area, defaults to 100m |
| epochUTCInSeconds      | long  | UTC epoch in seconds                        |
| offsetFromUTCInSeconds | long  | offset from UTC in seconds                  |
| eventsAsJson           | String | The name and values of the events as json           |

Events `eventsAsJson` can be any metric or KPI relevant for better understanding of end-user behaviour. All events provided through this call will be attached to the same second. If you do not have access to any location information when this function is called, provide `nil` as the first parameter.
Event names must be present in the dictionary provided when calling [init](#init). Events with unknown names will be discarded.
The events will be attached to exactly the same instant, which will be the timestamp provided by `location`

*Example:*

```kotlin
    // only location
    myDLDB.addEvents(loc,
                    null)

    // Location and event
    Location loc = ...
    myDLDB.addEvents(loc,
                    eventsAsJson: "{\"button\":\"log in\", \"batteryLevel\" : 55 }"
                    )

    // only event
    myDLDB.addEvents(nil,
                    eventsAsJson: "{\"button\":\"log in\", \"batteryLevel\" : 55 }"
                    )

    // location and time and event
    myDLDB.addEvents(0.0, 45.0, 30F, 
                    1644229847, 3600, 
                    "{\"button\":\"log in\", \"batteryLevel\" : 55 }")

```

### queriesLog

| parameter    | type     | description                                   |
| -----------  |----------|------------------------------------------     |
| maxEntries   | number   | the max number of entries to include in the log |

return the log of `maxEntries` most recent queries ran on the device.

The log is a json array (most recent last) of json objects, each object representing a query ran on the device

| Property  | Description   |
| --------  | ------------- |
| id        | uuid of query  |
| type      | type of query  |
| fetched   | timestamp in ms when the query was fetched from server |
| answered  | timestamp in ms when the query was answered on the device  |
| finished  | timestamp in ms when the query was finished on the device |
| tries     | number of tries to run the query |
| json_in   | parameters of the query |
| json_out  | results of the query |

*Example:*

```kotlin
        var latestQueriesLog = myDLDB.queriesLog(5)
```

### locationsLog

| parameter    | type     | description                                   |
| -----------  |----------|------------------------------------------     |
| durationInSeconds | number   | how far back in the past to look for locations |
| maxEntries   | number   | the max number of entries to include in the log |
| resolution   | number   | resolution of returned locations |

return the log of at most `maxEntries` most recent unique locations stored on the device during the last `durationInSeconds`.

The log is a json array (most recent last) of [h3](https://h3geo.org) in hex string format, at resolution `resolution`.
A call to this function generates an entry into the queries log.

*Example:*

```kotlin
        var latestLocationsLog = myDLDB.locationsLog(3600*24, 100, 7)
```

### close

to be called when shutting down the app.

*Example:*

```kotlin
        myDLDB.close()  
```

## About DLDB

DLDB provides behavioural analytics for mobile applications with privacy by design.

DLDB architecture relies on an SDK to be integrated into your mobile application, and a dashboard https://dashboard.dldb.io/ to build, query, analyze the behaviour of your application users.

For your application, DLDB deploys a distributed database, where each database instance is inside the mobile application scope. All the analytics queries are run by the devices and no raw data ever leaves the devices. Only the statistical KPI-s are sent anonymously to the DLDB dashboard .

From the DLDB dashboard, developers, analysts and app owners can build their own queries and analyze the results. No need to have any additional storage or analytics platform: DLDB provides an end-to-end solution.

DLDB SDK is written in C and has bindings to most common languages - works natively on iOS, Android, React-Native and Flutter. We also have Python binding and C libraries for IoT devices.

### Highlights

- Seamless integration of DLDB SDK into your mobile app source base, in many flavours, on all major platforms
- Define your own schema of collected events and values
- Built-in GDPR compliance on the right to be forgottent: all data belonging to your user stays on the device, so delete the data whenever requested
- Built-in GDPR compliance on the traceability of data usage: all requests processed by the DLDB SDK are traced and can be shown on demand
- No additional online storage
- Rapid scaling

## Support and contacts

If you face any problems or have any questions, please contact us

support channel in Discord: https://discord.gg/TD4f6p6nUH

email: support@dldb.io 

web: https://dldb.io/

subscribe for product updates and news: https://dldb.io/#Beta
