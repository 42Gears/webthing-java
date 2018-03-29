# webthing

Implementation of an HTTP [Web Thing](https://iot.mozilla.org/wot/).

# Using

## Gradle

Add the following dependency to your project:

```xml
<dependencies>
    <dependency>
        <groupId>org.mozilla.iot</groupId>
        <artifactId>webthing</artifactId>
        <version>CURRENT_VERSION</version>
    </dependency>
</dependencies>
```

## Gradle

Add the following dependency to your project:

```
dependencies {
    runtime(
        [group: 'org.mozilla.iot', name: 'webthing', version: 'CURRENT_VERSION'],
    )
}
```

# Example

In this example we will set up a dimmable Light and a Humidity sensor. (Both using fake data of course)
Both working examples can be found in [here](https://github.com/mozilla-iot/webthing-java/tree/master/src/main/java/org/mozilla/iot/webthing/example) 

## Dimmable Light
Imagine we have a dimmable Light, you want to expose via the web of things API.
The Light can be turned on/off and the brightness can be set from 0% to 100%
Besides the name/description and type a dimmable light is required to expose two properties
  - on
    - the state of the light, whether it is turned on or off
    - setting this property via a PUT {"on":true/false} call to the Api switches on/off this light
  - level
    - the brightness level of the light from 0-100%
    - setting this property via a PUT call to the API sets the brightness level of this light 

First we create a new Thing
```java
        this.thing =  new Thing("My Lamp", "dimmableLight", "A web connected lamp");
```
Now we can add the required properties

**On** property reports and sets the on/off state of the light.
For this we need to have a `value` Object which holds the actual state and it also knows how to actually turn on/off the light.
For our purposes we just want to log the new state if the light is switched on/off
```java
Map<String, Object> onDescription = new HashMap<>();
onDescription.put("type", "boolean");
onDescription.put("description", "Whether the lamp is turned on");
Value<Boolean> on = new Value<>(
    //here you could send a signal to the GPIO that switches the lamp off
    isOn -> System.out.printf("On-State is now  %s\n",isOn),
    true);
thing.addProperty(new Property(thing, "on", onDescription, on));
```

**level** property reports the brightness level of the light and sets the level.
Same as before, instead of actually setting the level of a light, we just log the level to std::out
```java
Map<String, Object> levelDescription = new HashMap<>();
levelDescription.put("type", "number");
levelDescription.put("description", "The level of light from 0-100");
levelDescription.put("minimum", 0);
levelDescription.put("maximum", 100);

Value<Double> level = new Value<>(
    //here you could send a signal to the GPIO that controls the brightness
    l -> System.out.printf("New light level is %s",l),
    0.0);

thing.addProperty(new Property(
    thing,
    "level",
    levelDescription,
    level));
```

Now we can add our newly created thing to the server and start it
```java
        try {
            List<Thing> things = new ArrayList<>();
            things.add(light);

            // If adding more than one thing here, be sure to set the second
            // parameter to some string, which will be broadcast via mDNS.
            // In the single thing case, the thing's name will be broadcast.
            WebThingServer server = new WebThingServer(things, "LightAndTempDevice", 8888);

            Runtime.getRuntime().addShutdownHook(new Thread() {
                public void run() {
                    server.stop();
                }
            });

            server.start(false);
        } catch (IOException e) {
            System.out.println(e);
            System.exit(1);
        }
```
This will start the server, making the light available via the *wot* rest API and announcing it as a discoverable (zero-conf) resource on your local network.

## Sensor
Let's now also connect a humidity Sensor to server we setup for our light.

A multiLevelSensor (a sensor that can also return a level instead of
just true/false) has two required properties (besides the name type and 
optional description)
the *on*-Property and the *level*-Property. Other than with the light,
we want to monitor those properties and want to get notified if the value changes.

First we create a new Thing
```java
 this.thing = new Thing(
            "My HumiditySensor",
            "multiLevelSensor",
            "A web connected humidity sensor");
```

then we create and add the appropriate properties

**On**-Property
The **On** property tells us whether the sensor... (unsure what this actually does)
```java

Map<String, Object> onDescription = new HashMap<>();
onDescription.put("type", "boolean");
onDescription.put("description", "Whether the sensor is running");
Value<Boolean> on = new Value<>(true);
thing.addProperty(new Property(thing, "on", onDescription, on));
```

**Level**-Property
the **Level** property tells us what the sensor is actually reading.
Contrary to the light, the value cannot be set via an API call
(wouldn't make much sense, to SET what a sensor is reading)
Therefor we are utilizing a *readOnly* Value by omitting the Consumer parameter

```java
Map<String, Object> levelDescription = new HashMap<>();
levelDescription.put("type", "number");
levelDescription.put("description", "The current humidity in %");
levelDescription.put("unit", "%");

this.level = new Value<>(0.0);
```
Now we would have a sensor that constantly report 0.0%.
To make it usable we need a thread or some kind of input when the sensor has a new 
reading available. For this purpose we start a thread that queries the physical sensor 
every few seconds. (for our purposes it just calls a fake method)
```java
//start a thread that pulls the sensor reading every n-seconds
new Thread(()->{
    while(true){
        try {
            Thread.sleep(3000);
            //updates the underlying value, which in turn notifies all listeners
            this.level.notifyOfExternalUpdate(readFromGPIO());
        } catch (InterruptedException e) {
            throw new IllegalStateException(e);
        }
    }
}).start();
```

This will update our value object with the sensor readings via the
`this.level.notifyOfExternalUpdate(readFromGPIO());` call. The value object now notifies
the property and the thing that the value has changed, which in turn then notify all websocket listeners
