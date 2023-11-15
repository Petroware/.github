# Tutorial I: Using JWitsml with ETP

ETP communication may be simple in theory, but when creating full-scale enterprise-ready 24/7-reliable real-time software, ETP soon becomes difficult to handle. The goal of the JWitsml project is to encapsulate all the complexity of ETP and provide a clean, simple and well documented API directly towards the back-end business model.

Energistics provides various *development kits* for ETP. One should be careful when using these as they expose far more of the ETP and web socket logic than necessary. They can act as useful starting blocks, but the elements of these kits are not equipped to be part of an enterprise application data model.

ETP communication is done through predefined [*Avro*](https://avro.apache.org/) encoded messages and data structures. The data structures contains bulk data in compressed XML form according to the WITSML version being used. All web socket communication is asynchronous. On top of the web socket protocol ETP defines a separate *session* model. Through its nine *sub protocols* ETP defines 54 different messages that can be sent between a server and a client. JWitsml has simplified this logic, and achieve full ETP coverage through only eleven simple methods. The result is a very narrow and simple to use API.

All ETP communication starts by creating an **EtpServer** instance that represents the ETP server in the client program:

```java
// Create ETP server endpoint
EtpServer etpServer = new EtpServer("wss://path/to/etp/server", "userName", "password");
```

The **EtpServer** is a factory for **EtpSession** instances that implements the main methods of the ETP protocols. The **EtpSession** is of *read* or *write* type depending on the ETP method that is to be called.

```java
// Create an ETP read session
EtpSession etpSession = etpServer.newReadSession();
```

The nine different sub protocols of ETP each defines a specific *appliance* of ETP. Each of these, and how they are accessed through JWitsml are discussed in detail below. In this discussion it is important to realize that due to the nature of web socket communication most calls to **EtpSession** will be *multi-threaded* in the back-end. This complexity is transparent for the JWitsml client, but as communication may both hang or fail it is important that client code always access **EtpSession** outside the main application thread.

## Protocol 0 - Core

The *Core* protocol defines several messages that manage ETP *sessions*. None of these are exposed to the JWitsml client; The back-end ETP sessions are automatically established when required and re-established when lost. In addition, protocol *roles* are automatically negotiated according to the nature of the client calls.

## Protocol 3 - Discovery

The *Discovery* protocol is used to navigate the remote store. Instances in the store are organized in a strict hierarchy, and each item is identified by a *resource*. The resources are easily navigated by sending *GetResources* messages to the ETP session:

```java      
// Get children resources for a given parent (null for root level)
List<EtpResource> resources = etpSession.getResources(parentResource);
```

This simple method is the typical basis for a GUI tree component that exposes the entire structure of a remote store.

## Protocol 1 - Channel Streaming

ETP log data are streamed by *channel* but are in JWitsml organized as *log sets* to be more consistent with industry well logging standards. A log set has an index curve and a number of value curves sampled along the index.

A log set is established by sending the *ChannelDescribe* message to the ETP server:

```java      
// Get log set for a specific log, curve, or channel set resource
EtpLogSet logSet = etpSession.channelDescribe(resource);
```

This log set will have no values, but will have the index and all its curve definitions in place.
Log set population is controlled through the *ChannelStreamingStart* and the *ChannelStreamingStop* messages:

```java       
// Start channel streaming to the given log set from the given index value or (if null) from top
etpSession.channelStreamingStart(logSet, indexValue);

// Stop streaming
etpSession.channelStreamingStop(logSet);
```
    
The **EtpLogSet** class is thread-safe, secured by separate read- and write locks in order to ensure protection and maximum performance for the client code.

The client listens for changes to the log set by adding an **EtpLogSetListener** to the log set:

```java   
// Listening interface for log set changes
public interface EtpLogSetListener
{
    void logSetChanged(EtpLogSet logSet);
}
```
   
The typical **logSetChanged** response for the client will be to refresh a GUI.


## Protocol 2 - Channel Data Frame

The *Channel Data Frame* protocol exists in order to be able to retrieve complete log sets of historic (non-streaming) data. This can easily be achieved through the protocol 1 approach, but it is even simpler using protocol 2; There is no need to listen for log set changes as the log set is populated on return:

```java
// Get log set for a specific log, curve, or channel set resource
EtpLogSet logSet = etpSession.requestChannelData(resource, fromIndex, toIndex);
```
    
Again: The implementation of the method is multi-threaded, and it blocks until the process is complete. It should consequently *not* be called from a main application thread.

## Protocol 4 - Store

The *Store* protocol is for CRUD access to store instances. ETP objects are communicated through Avro objects with embedded XML in compressed form. All this complexity is transparent to the JWitsml client however as the XML are conveniently uncompressed and converted into their equivalent Java classes as defined for each WITSML version 1.3, 1.4 or 2.0.

```java      
// Get object from data store
WitsmlObject witsmlObject = etpSession.getObject(resource);

// Create or update an object in data store
etpSession.putObject(witsmlObject);

// Delete object from data store
etpSession.deleteObject(witsmlObject);
```

## Protocol 5 - Store Notification

The Store Notification protocol is used to allow client programs to receive notifications of changes to data objects in the store. Notification subscription is done by sending a *NotificationRequest* message to the ETP server:

```java
// Get notifications on the object identified by the specified resource
etpSession.notificationRequest(resource, listener);

// Cancel notifications on the object
etpSession.cancelNotification(resource);
```
    
The listening interface is defined as follows:

```java
public interface EtpStoreNotificationListener {

    // Indicate that an object has been changed
    void objectChanged(EtpResource resource);

    // Indicate that an object has been deleted
    void objectDeleted(EtpResource resource);
}
```

## Protocol 6 - Growing Object

The *Growing Object* protocol is used to manage non-log index-based objects: *trajectories*, *wellbore geology intervals* and *wellbore geometry sections*. For some reason this protocol is *not* streaming, but responds to explicit calls from the client.

The growing object protocol defines several methods for accessing growing objects, but due to the push/pull nature of the calls JWitsml exposes only one of these, the *GrowingObjectGetRange*, conveniently renamed to *growingObjectGet* and always getting the full range of values:

```java      
// Get the specified growing object
WitsmlObject witsmlObject = etpSession.growingObjectGet(resource);
```

## Protocol 7 - Data Array
The *Data Array* protocol is used to transfer large, binary arrays of data between a client and an ETP server. The protocol is not used by WITSML, but as ETP is the foundation for PRODML and RESQML as well, it is included in the ETP module of JWistml for completeness:

```java
// Get a specific data array from the server
EtpDataArray dataArray = etpSession.getDataArray(uri);

// Get part of a data array from the server
EtpDataArray dataArray = etpSession.getDataArraySlice(uri, start, count);

// Send a data array to the server
etpSession.putDataArray(uri, dataArray);

// Send part of a data array to the server
etpSession.putDataArraySlice(uri, dataArray, start, count););
```

## Protocol 8 - WITSML SOAP
The *WITSML SOAP* protocol allows ETP clients to make WITSML 1.* server calls using ETP instead of SOAP. This can be thought of as a technology *migration*, but it is not included in JWitsml as JWitsml contains both the SOAP and the ETP technologies anyway.

# Tutorial II: Using JWitsml with HTTP/SOAP

Below is a complete example showing the few simple steps necessary to access data from a WITSML SOAP based server. The program lists the names of all wells in a given server.

```java
import java.net.MalformedURLException;
import java.net.URL;
import java.util.List;

import no.petroware.jwitsml.Capabilities;
import no.petroware.jwitsml.WitsmlQuery;
import no.petroware.jwitsml.WitsmlServer;
import no.petroware.jwitsml.WitsmlServerException;
import no.petroware.jwitsml.WitsmlVersion;
import no.petroware.jwitsml.model.WitsmlWell;

public class BasicExample {
    public static void main(String[] arguments) {
        try {
        // Identify ourself and our capabilities as WITSML client
        Capabilities clientCapabilities = new Capabilities(WitsmlVersion.VERSION_1_4,
                                                            "First And Last name",
                                                            "e-mail@some.org",
                                                            "+12 34 56 789",
                                                            "Description",
                                                            "Application Name",
                                                            "Vendor",
                                                            "Version Number");
        
        // Establish URL to the server
        URL url = new URL("http://path/to/witsml/server");
        
        // Create the WITSML server instance
        WitsmlServer witsmlServer = new WitsmlServer(url, "userName", "password",
                                                        clientCapabilities);
        
        // Retrieve all wells
        List<WitsmlWell> wells = witsmlServer.get(WitsmlWell.class,
                                                    new WitsmlQuery());
        
        // Write well names to the console
        for (WitsmlWell well : wells)
            System.out.println(well.getName());
        }
        catch (MalformedURLException exception) {
            exception.printStackTrace();
        }
        catch (WitsmlServerException exception) {
            exception.printStackTrace();
        }
    }
}
```
    
Unlike the previous ETP examples, all calls to the **WitsmlServer** instance are single-threaded.

## Getting WITSML data from the server

All data are retrieved through a [WitsmlServer](https://petroware.no/products/jwitsml/javadoc/no/petroware/jwitsml/WitsmlServer.html) instance which represents the WITSML server in the client program.
There are bascially three different ways to get data:

1. Get multiple children of a given type from a specific parent

```java 
// Get all wellbores from a given well instance
List<WitsmlWellbore> wellbores = witsmlServer.get(WitsmlWellbore.class,
                                                new WitsmlQuery(),
                                                well);


// Get all rigs from a given wellbore instance
List<WitsmlRig> rigs = witsmlServer.get(WitsmlRig.class,
                                        new WitsmlQuery(),
                                        wellbore);


// ... or through the wellbore ID
List<WitsmlRig> rigs = witsmlServer.get(WitsmlRig.class,
                                        new WitsmlQuery(),
                                        "ID-wellbore");


// Both parent and grandparent may be specified
List<WitsmlMudLog> mudLogs = witsmlServer.get(WitsmlMudLog.class,
                                            new WitsmlQuery(),
                                            "ID-wellbore",
                                            "ID-well");


// Wells doesn't have parents so we can skip the parent argument
List<WitsmlWell> wells = witsmlServer.get(WitsmlWell.class,
                                        new WitsmlQuery());


// Attempt to get all rigs! Some servers will accept this approach
// while others will require a valid wellbore parent.
// The WITSML standard is unclear on this issue.
List<WitsmlRig> rigs = witsmlServer.get(WitsmlRig.class,
                                        new WitsmlQuery());
```
 
2. Get a specific child of a given type from a specific parent

```java
// Get a specific log instance from a wellbore instance
WitsmlLog log = witsmlServer.getOne(WitsmlLog.class,
                                    new WitsmlQuery(),
                                    "ID-log",
                                    wellbore);


// ... or through the wellbore ID
WitsmlLog log = witsmlServer.getOne(WitsmlLog.class,
                                    new WitsmlQuery(),
                                    "ID-log",
                                    "ID-wellbore");


// Wells doesn't have parents so we can skip the parent argument
WitsmlWell well = witsmlServer.getOne(WitsmlWell.class,
                                    new WitsmlQuery(),
                                    "ID-well");
```

3. Refresh an already instantiated object

```java     
// Get all attributes from an already loaded wellbore instance
witsmlServer.refresh(wellbore, new WitsmlQuery());
```
  
Each JWitsml class representing WITSML instances has getter methods for all the individual properties defined by WITSML for that type.

Attribute names in the JWitsml library does not necessarily correspond exactly to the WITSML definition as the former is specified in a *human readable* form. Names like *dTimKickoff* is translated into kickoffTime etc.

     WitsmlMudLog mudLog = ...

     System.out.println("Name: " + mudLog.getName());
     System.out.println("Time: " + mudLog.getTime());
     System.out.println("Mud log company: " + mudLog.getMudLogCompany());
  
Floating point values that may have a *unit* associated are represented by the [Value](https://petroware.no/products/jwitsml/javadoc/no/petroware/jwitsml/Value.html) type which is a compound of a double and a unit symbol string.

## The WitsmlQuery class

The [WitsmlQuery](https://petroware.no/products/jwitsml/javadoc/no/petroware/jwitsml/WitsmlQuery.html) class is used to control *which properties* to extract for each object as well as for *object filtering*.

A default **WitsmlQuery** instance as shown in the above examples simply means *get all properties*.

Getting only a subset of the properties of a type is done by specifically including the requested WITSML elements in the **WitsmlQuery** object:

```java
// Construct a query consisting of the name property only
WitsmlQuery query = new WitsmlQuery();
query.includeElement("name");

// Get all wells (name only) from a given server
List<WitsmlWell> wells = WitsmlServer.get(WitsmlWell.class,
                                            query);
```
  
Note that when specifying elements and attributes for the **WitsmlQuery** class, the WITSML terms as specified in the [WITSML 1.3](http://w3.energistics.org/schema/witsml_v1.3.1_data/doc/WITSML_Schema_docu.htm) or [WITSML 1.4](http://w3.energistics.org/schema/witsml_v1.4.1_data/doc/witsml_schema_overview.html) schema respectively must be used.

Sometimes it is more convenient to get all elements *but* a few, perhaps elements representing bulk data. In the example below all trajectories of a wellbore are retrieved including all their properties except the trajectory stations:

```java
// Get all trajectories for a wellbore, but exclude the stations
WitsmlQuery query = new WitsmlQuery();
query.excludeElement("stations");

List<WitsmlTrajectory> trajectories = WitsmlServer.get(WitsmlTrajectory.class,
                                                       query,
                                                       wellbore);
```
  
When an object is initially instantiated with only a few properties, more properties can be loaded using the **refresh()** method:

```java
// Get wellbore with name property only
WitsmlQuery query = new WitsmlQuery();
query.includeElement("name");
WitsmlWellbore wellbore = WitsmlServer.getOne(WitsmlWellbore.class,
                                            query,
                                            "ID-wellbore");

// Get more properties 
query = new WitsmlQuery();
query.includeElement("statusWellbore");
query.includeElement("mdCurrent");
query.includeElement("tvdCurrent");
query.includeElement("mdPlanned");
query.includeElement("tvdPlanned");

witsmlServer.refresh(wellbore, query);
```
  
Note that calls to the **WitsmlServer.get()**, **getOne()** and **refresh()** methods might be time consuming, and in most cases it is more efficient to retrieve all attributes in a single server access.

## Filtering

The [WitsmlQuery](https://petroware.no/products/jwitsml/javadoc/no/petroware/jwitsml/WitsmlQuery.html) class can be used for object filtering as shown in the following example:

```java
// Get all active wellbores of a well
WitsmlQuery query = new WitsmlQuery();
query.addElementConstraint("statusWellbore", WitsmlWell.Status.ACTIVE);

List<WitsmlWellbore> wellbores = witsmlServer.get(WitsmlWellbore.class,
                                                  query,
                                                  well);
```

More than one filter may be specified in the same query, and the same filter may be set repeatedly in order to filter on more than one value:
     
     // Get log including a subset of the available curves and no bulk data
     WitsmlQuery query = new WitsmlQuery();
     query.addElementConstraint("mnemonic", "Depth");
     query.addElementConstraint("mnemonic", "GR");
     query.addElementConstraint("mnemonic", "Dens");
     query.excludeElement("logData");

     WitsmlLog log = witsmlServer.getOne(WitsmlWellbore.class,
                                         query,
                                         "ID-log",
                                         wellbore);
  
Another example:

```java
// Get list of wellbores with known IDs
WitsmlQuery query = new WitsmlQuery();
query.addAttributeConstraint("wellbore", "uid", "id1");
query.addAttributeConstraint("wellbore", "uid", "id2");
query.addAttributeConstraint("wellbore", "uid", "id3");

List<WitsmlWellbore> wellbores = witsmlServer.get(WitsmlWellbore.class,
                                                  query,
                                                  well);
```

The last example will not work if it is combined with other element or attribute constraints as JWitsml wouldn't know to which part of the query the additional constraints should be applied. In this case it is possible to use *sub-queries* to achieve finer control of the query. This example returns the same wellbores as in the example above, but in addition it set constraints for each of them:
     
```java
// Get list of wellbores with known IDs
WitsmlQuery query = new WitsmlQuery();

WitsmlQuery q1 = query.addSubQuery("wellbore");
q1.addAttributeConstraint("wellbore", "uid", "id1");
q1.includeElement("name");

WitsmlQuery q2 = query.addSubQuery("wellbore");
q2.addAttributeConstraint("wellbore", "uid", "id2");
q2.includeElement("name");
q2.includeElement("shape");

WitsmlQuery q3 = query.addSubQuery("wellbore");
q3.addAttributeConstraint("wellbore", "uid", "id3");
q3.addAttributeConstraint("mdCurrent", "uom", "ft");

List<WitsmlWellbore> wellbores = witsmlServer.get(WitsmlWellbore.class,
                                                  query,
                                                  well);
```

## Adding WITSML objects

Writing new data to the WITSML server requires the instantiation of new objects in the client program, specifying the necessary properties and submitting the instance to the server.

New root level objects (**Witsml<type>**) are created using the **WitsmlServer** instance as a *factory* class through the **WitsmlServer.newInstance()** method. Sub types are always created by using the parent object as a factory class, either through the **new<What>** or **add<What>** methods depending on if it has 1:1 or 1:n relationship respectively.

This example creates a WITSML rig instance with some of its defined sub-components:

    
```java
// Create empty rig instance using factory method
WitsmlRig rig = witsmlServer.newInstance(WitsmlRig.class,
                                            name,
                                            wellbore);

// Set rig properties
rig.setType(WitsmlRig.Type.FLOATER);
rig.setActive(true);
rig.setRigOwner("Subsea 7");
rig.setDerrickType(WitsmlRig.DerrickType.SLANT);
:

// Create rig BOP instance
WitsmlRig.Bop bop = rig.newBop();
bop.setModel("Single Ram");
bop.setManufacturer("Cameron");
bop.setConnectionSize(new Value(18.625, "in"));
bop.setPressureRating(new Value(5000.0, "psi"));
bop.setSize(new Value(new Value(12.0, "in"));
:

// Add a rig BOP component
WitsmlRig.Bop.Component component1 = bop.addComponent();
component1.setType(WitsmlRig.Bop.Component.Type.PIPE_RAM);
component1.setDescription("WP w/ stainless");
component1.setInnerPassThruDiameter(new Value(4.0, "in"));
component1.setWorkingPressure(new Value(20000, "psi"));
component1.setMinDiameter(new Value(4.0, "in"));
component1.setMaxDiameter(new Value(12.0, "in"));
component1.setNomenclature("S");
component1.setVarable(true);
:

// Add a second rig BOP component
WitsmlRig.Bop.Component component2 = bop.addComponent();
component2.setType(WitsmlRig.Bop.Component.Type.ANNULAR_PREVENTER);
component2.setDescription("Second WP w/ stainless");
component2.setInnerPassThruDiameter(new Value(4.0, "in"));
component2.setWorkingPressure(new Value(22000, "psi"));
component2.setMinDiameter(new Value(4.0, "in"));
component2.setMaxDiameter(new Value(12.0, "in"));
component2.setNomenclature("S");
component2.setVarable(true);
:

// Add rig pump instance
WitsmlRig.Pump pump1 = rig.addPump();
pump1.setPumpNumber(1);
pump.setManufacturer("Oilwell");
pump.setType(WitsmlRig.Pump.Type.TRIPLEX);
:

//
// Finally write the compound object to the server
// 
try {
    witsmlServer.add(rig);
}
catch (WitsmlServerException exception) {
    exception.printStackTrace();
}
```
  
Note that calling **WitsmlServer.newInstance()** only establish Java objects within the client code, it does not cause any remote access.

Unique IDs for new instances are created automatically by JWitsml. If the client needs to control this process, it may install an **IdGenerator** in the **WitsmlServer**:

```java
class MyIdGenerator implements IdGenerator
{
    public String generateId(String witsmlType, String name, WitsmlObject parent, String defaultId)
    {
        String id = ...;
        return id;
    }
}

witsmlServer.setIdGenerator(new MyIdGenerator());
```
  
Note that this is optional and that JWitsml by default will generate high quality [Leach-Salz](https://en.wikipedia.org/wiki/Universally_unique_identifier) IDs that are guaranteed to be universally unique.

## Updating WITSML objects

When the client program has a reference to an object, new properties can be updated in the server at any time. Use the object setter methods to specify changes and use the **WitsmlServer.update()** method to propagate the changes back to the server.

This example adds an additional section to an existing **WitsmlSurveyProgram** instance:

    
```java
// Create a new section for existing survey program
WitsmlSurveyProgram.Section section = surveyProgram.addSection();
section.setSequenceNo(8);
section.setName("section8");
section.setMdStart(new Value(250.0, "m"));
section.setMdEnd(new Value(260.0, "m"));
:

try {
    witsmlServer.update(surveyProgram);
}
catch (WitsmlServerException exception) {
    exception.printStackTrace();
}
```

## Deleting WITSML objects

Deleting a WITSML objects is done by the **WitsmlServer.delete()** call:
    
```java
// Delete a log from the server
try {
    witsmlServer.delete(log);
}
catch (WitsmlServerException exception) {
    exception.printStackTrace();
}
```
  
Note that servers *may* require children to be deleted first, if an object with children is to be deleted. Normally it should be sufficient to delete the root node however.

## Unit of Measure

WITSML defines an elaborate set of units and unit conversions. More than 2.300 units of more than 250 different petroleum related quantities are defined. JWitsml supports this information through a powerful yet simple API.

WITSML units are accessed through the [WitsmlUnitManager](https://petroware.no/products/jwitsml/javadoc/no/petroware/jwitsml/units/WitsmlUnitManager.html) singleton class. The unit manager provides an optional service for client programs that are doing computational operations on numerical WITSML data.

*Units* (such as "*meter*" or "*bar*") are organized by WITSML in *quantities* (such as "*length*" or "*pressure*"). Each quantity defines a *base unit* (typically the SI unit) and potentially a set of *derived units*.

The following piece of code lists all quantities and associated units available:

```java
import no.petroware.jwitsml.units.WitsmlUnitManager;
import no.petroware.jwitsml.units.WitsmlUnit;

:

// Get the unit manager
WitsmlUnitManager unitManager = WitsmlUnitManager.getInstance();

// Get all known quantities
List<String> quantities = unitManager.getQuantities();
for (String quantity : quantities) {
    System.out.println("Quantity: " + quantity);

// List all units of the quantity
List<WitsmlUnit> units = unitManager.getUnits(quantity);
for (WitsmlUnit unit : units)
    System.out.println("  " + unit);
}
```

Converting between units is done using the **convert()** method of the **WitsmlUnitManager**:

```java
// Get units for feet and centimeters respectively
WitsmlUnit ftUnit = unitManager.findUnit("ft");
WitsmlUnit cmUnit = unitManager.findUnit("cm");

// Convert between the two
double ftValue = 1.0;
double cmValue = unitManager.convert(ftUnit, cmUnit, ftValue);
```
  
When calling a WITSML server a client may specify the preferred return unit for a certain property. In fact JWitsml by default always requests values in their base units. As such requests are defined by the WITSML standard to be regarded as *hints* only, the client can unfortunately never rely on this feature. Until this is properly sorted out by the standard, the client is always forced to handle unit conversions locally:

```java
Value currentMd = wellbore.getCurrentMd();

WitsmlUnit unit = unitManager.findUnit(currentMd.getUnit());
WitsmlUnit baseUnit = unitManager.findBaseUnit(unit);

double currentMdInMeters = unitManager.convert(unit, baseUnit,
                                               currentMd.getValue());
```
  
As this is such a common operation it can be done using the **toBaseUnit()** convenience method:

```java
// Get current wellbore MD in meters
Value currentMd = unitManager.toBaseUnit(wellbore.getCurrentMd());
```
  
Note that using the JWitsml unit module never causes any remote WITSML server calls. The unit data base of WITSML is part of the JWitsml delivery and thus local to the client software, and it may be used as a unit conversion engine independently of WITSML.

JWitsml uses version 2.2 of the Energistics unit database which is compatible with WITSML 1.4 and PRODML 1.1. JWitsml has made a number of modifications to this database in order to tune incompatibilities and defects.

## Connection Logging

A client application may supervise the details of the ETP communication by adding a [EtpAccessListener](https://petroware.no/products/jwitsml/javadoc/no/petroware/jwitsml/etp/EtpAccessListener.html) to the **EtpServer** class:

```java
public class AccessListener implements EtpAccessListener {

    @Override
    public void access(EtpAccess access) {
        // Logging etc. goes here
    }
}
```
  
The **access()** method is called immedeately *before* an ETP message is sent from the client and immediately *after* an ETP message from the server has arrived at the client. The **EtpAccess** instance contains details about the message, its time stamp, and error codes and so on.

The methods may be used for simple client logging or for creating advanced performance statistics etc.

The equivalent API for WITSML 1 is through [WitsmlAccessListener](https://petroware.no/products/jwitsml/javadoc/no/petroware/jwitsml/WitsmlAccessListener.html) in the **WitsmlServer** class:

```java
public class ClientListener implements WitsmlAccessListener {

    @Override
    public void requestSent(WitsmlRequest request) {
        // Logging etc. goes here
    }

    @Override
    public void responseReceived(WitsmlResponse response) {
        // Logging etc. goes here
    }
}
```
  
The **requestSent()** method is called immediately *before* JWitsml access the remote WITSML server. The **responseReceived()** method is called immediately *after* the response is received from the server.

The [WitsmlRequest](https://petroware.no/products/jwitsml/javadoc/no/petroware/jwitsml/WitsmlRequest.html) instance contains information such as the time of request, the low level WSDL procedure name, the XML query, the WITSML type being queried etc. The [WitsmlResponse](https://petroware.no/products/jwitsml/javadoc/no/petroware/jwitsml/WitsmlResponse.html) instance contains a link to the corresponding request instance as well as the complete XML response, response time, status code, server message and exception if the access failed for some reason.
