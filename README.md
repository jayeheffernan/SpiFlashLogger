# SPIFlashLogger 2.1.0

The SPIFlashLogger manages all or a portion of a SPI flash (either via imp003+'s built-in [hardware.spiflash](https://electricimp.com/docs/api/hardware/spiflash) or any functionally compatible driver such as the [SPIFlash library](https://github.com/electricimp/spiflash)).

The SPIFlashLogger creates a circular log system, allowing you to log any serializable object (table, array, string, blob, integer, float, boolean and `null`) to the SPIFlash. If the log systems runs out of space in the SPIFlash, it begins overwriting the oldest logs.

**To add this library to your project, add `#require "SPIFlashLogger.class.nut:2.1.0"` to the top of your device code.**

You can view the library’s source code on [GitHub](https://github.com/electricimp/spiflashlogger/tree/v2.1.0).

## Memory Efficiency

The SPIFlash logger operates on 4KB sectors and 256-byte chunks. Objects needn't be aligned with chunks or sectors.  Some necessary overhead is added to the beginning of each sector, as well as each serialized object (assuming you are using the standard [Serializer library](https://electricimp.com/docs/libraries/utilities/serializer.1.0.0/)). The overhead includes:

- 6 bytes of every sector are expended on sector-level metadata.
- A four-byte marker is added to the beginning of each serialized object to aid in locating objects in the datastream.
- The *Serializer* object also adds some overhead to each object (see the [Serializer's documentation](https://electricimp.com/docs/libraries/utilities/serializer.1.0.0/) for more information).
- After a reboot the sector metadata allows the class to locate the next write position at the next chunk. This wastes some of the previous chunk, though this behaviour can be overridden using the *getPosition()* and *setPosition()* methods.

## Class Usage

### Constructor: SPIFlashLogger(*[start, end, spiflash, serializer]*)

The SPIFlashLogger’s constructor takes four parameters, all of which are optional:

| Parameter | Default Value | Description |
| --- | --- | --- |
| *start* | 0 | The first byte in the SPIFlash to use (must be the first byte of a sector). |
| *end*  | *spiflash.size()*   | The last byte in the SPIFlash to use (must be the last byte of a sector). |
| *spiflash*  | **hardware.spiflash** | hardware.spiflash, or an object with an equivalent interface such as the [SPIFlash](https://electricimp.com/docs/libraries/hardware/spiflash.1.0.1/) library. |
| *serializer* | Serializer class | The static [Serializer library](https://electricimp.com/docs/libraries/utilities/serializer.1.0.0/), or an object with an equivalent interface. |

```squirrel
// Initializing a SPIFlashLogger on an imp003+
#require "Serializer.class.nut:1.0.0"
#require "SPIFlashLogger.class.nut:2.1.0"

// Initialize Logger to use the entire SPI Flash
logger <- SPIFlashLogger();
```

```squirrel
// Initializing a SPIFlashLogger on an imp002
#require "Serializer.class.nut:1.0.0"
#require "SPIFlash.class.nut:1.0.1"
#require "SPIFlashLogger.class.nut:2.1.0"

// Setup SPI Bus
spi <- hardware.spi257;
spi.configure(CLOCK_IDLE_LOW | MSB_FIRST, 30000);

// Setup Chip Select Pin
cs <- hardware.pin8;
spiFlash <- SPIFlash(spi, cs);

// Initialize the logger object using the entire SPIFlash
logger <- SPIFlashLogger(null, null, spiFlash);
```

## Class Methods

### dimensions()

The *dimensions()* method returns a table with the following keys, each of which gives access to an integer value:

| Key | Description |
| --- | --- |
| *size* | The size of the SPIFlash in bytes |
| *len* | The number of bytes allocated to the logger |
| *start* | The first byte used by the logger |
| *end* | The last byte used by the logger |
| *sectors* | The number of sectors allocated to the logger |
| *sector_size* | The size of sectors in bytes |

### write(*object*)

Writes any serializable object to the memory allocated to the SPIFlashLogger. If the memory is full, the logger will begin overwriting the oldest entries.

```squirrel
function readAndSleep() {
    // Read and log the data
    local data = getData();
    logger.write(data);

    // Go to sleep for an hour
    imp.onidle(function() { server.deepsleepfor(3600); });
}
```

### read(*onData[, onFinish, step, skip]*)

The '.read()' method reads objects from the logger asynchronously, calling `onData` on each (subject to `step`, `skip`, and early termination within `onData`).  This allows for the asynchronous processing of each log object, such as sending data to the agent and waiting for an acknowledgement.

The optional *onFinish* callback will be called after the last object is located (with no parameters).

`step` is an optional parameter controlling the rate at which the scan steps through objects, for example, setting `step == 2` will cause `onData` to be called only for every second object found.  Negative values are allowed for scanning through objects backwards, for example, `step == -1` will scan through all objects, starting from the most recently written and stepping backwards.  Skip can be used to skip a number of objects at the start of reading.  For example, a `step` of 2 and `skip` of 0 (the default) will call `onData` for every second object *starting from the first*, whereas with `skip == 1` it will be every second object *starting from the second*, thus the two options provide full coverage with no overlap.  As a potential use case, one might log two versions of each message: a short, concise version, and a longer, more detailed version.  `step == 2` could then be used to pick up only the concise versions.

#### *onData(object, address, next)*

The on data callback is called for each requested object with three parameters: the deserialized object, the SPIFlash address of the (start of) the object, and a `next` callback.  Reading an object does not erase it, but the object can be erased in the body of `onData` by passing `address` to `erase()`.  `onData` should call next when it is is ready to scan for the next item.  Passing `false` to `next` aborts the scanning, skipping to `onFinish`

```squirrel
logger.read(
    // For each object in the logs
    function(dataPoint, addr, next) {
        // Send the dataPoint to the agent
        server.log(format("Found object at spiflash address %d", addr))
        agent.send("data", dataPoint);
        // Erase it from the logger
        logger.erase(addr);
        // Wait a little while for it to arrive
        imp.wakeup(0.5, next);
    },

    // All finished
    function() {
        server.log("Finished sending and all entries are erased")
    }
);
```

### erase(*[addr]*)

This method erases an object at spiflash address `addr` by marking it erased.  If `addr` is not given, it will (properly) erase all allocated memory.

### eraseAll(*[force = false]*)

Erases the entire allocated SPIFlash area.

### getPosition()

The *getPosition()* method returns the current SPI flash pointer, ie. where the SPIFlashLogger will perform the next read/write task. This information can be used along with the *setPosition()* method to optimize SPIFlash memory usage between deep sleeps.

See *setPosition()* for sample usage.

### setPosition(*position*)

The *setPosition()* method sets the current SPI flash pointer, ie. where the SPIFlashLogger will perform the next read/write task. Setting the pointer can help optimize SPI flash memory usage between deep sleeps, as it allows the SPIFlashLogger to be precise to one byte rather 256 bytes (the size of a chunk).

```squirrel
// Create the logger object
logger <- SPIFlashLogger();

// Check if we have position information in the nv table:
if ("nv" in getroottable() && "position" in nv) {
    // If we do, update the position pointers in the logger object
    logger.setPosition(nv.position);
} else {
    // If we don't, grab the position points and set nv
    local position = logger.getPosition();
    nv <- { "position": position };
}

// Get some data and log it
data <- getData();
logger.write(data);

// Increment a counter
if (!("count" in nv)) {
    nv.count <- 1;
} else {
    nv.count++;
}

// If we have more than 100 samples
if (nv.count > 100) {
    // Send the samples to the agent
    logger.read(
        function(dataPoint, addr, next) {
            // Send the dataPoint to the agent
            agent.send("data", dataPoint);
            // Erase it from the logger
            logger.erase(addr);
            // Wait a little while for it to arrive
            imp.wakeup(0.5, next);
        },
        function() {
            server.log("Finished sending and all entries are erased");
            // Reset counter
            nv.count <- 1;
            // Go to sleep when done
            imp.onidle(function() {
                // Get and store position pointers for next run
                local position = logger.getPosition();
                nv.position <- position;

                // Sleep for 1 minute
                imp.deepsleepfor(60);
            });
        }
    );
} else {
    // Go to sleep
    imp.onidle(function() {
        // Get and store position pointers for next run
        local position = logger.getPosition();
        nv.position <- position;

        // Sleep for 1 minute
        imp.deepsleepfor(60);
    });
}
```

# License

The SPIFlashLogger class is licensed under [MIT License](https://github.com/electricimp/spiflashlogger/tree/master/LICENSE).
