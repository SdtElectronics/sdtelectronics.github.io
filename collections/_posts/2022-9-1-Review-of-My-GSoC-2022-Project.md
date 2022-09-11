---
layout: article
title: Review of My GSoC-2022 Project
categories: Programming
tags: zephyr embedded
eyeCatcher: https://sdtelectronics.github.io/assets/gallery/2022-9-1-Review-of-My-GSoC-2022-Project.jpg
abstract: A brief summary to the GSoC-2022 Project - thrift for zephyr, which I have contributed to.
---

This post is written for the [Google Summer of Code 2022](https://summerofcode.withgoogle.com) project -  thrift for zephyr. Check the [Git repository](https://github.com/zephyrproject-rtos/gsoc-2022-thrift) and the [project proposal](https://summerofcode.withgoogle.com/programs/2022/projects/k7Be74Yb) for more context.

## What is Thrift?
Consider when you want to build a weather station with several sensors and a display to show the data. It is also connected so we can retrieve the data remotely. Without any high-level library, you have to devise a serialization scheme for sensor data, implement a data exchange protocol, and finally work on the logic relevant to the weather station. Quite some work nevertheless, this is not too bad. Now imagine you want to add a new sensor some day, and have to do all the repetitive work again.

![A weather station powered by ESP32](https://content.instructables.com/ORIG/FWW/5NIE/JA36PCOT/FWW5NIEJA36PCOT.jpg?auto=webp&fit=bounds&frame=1&height=1024&width=1024auto=webp&frame=1&height=150)

This is when a RPC framework like [Apache Thrift](https://thrift.apache.org/) shines. With Thrift, you can specify what data is going to be exchanged with a few lines of code, and all the serialization, protocol and server code is automatically generated. Let's get a closer look at how this is done.

The data and operation in a service are described with the Thrift IDL language. In the weather station example above, the IDL file will include the following sections:
* First, the data sampled by sensors. As each sensor generates a specific type of data, this is described by a union. Each field starts with an index, followed by the type and name:

``` c++
union SampleValue {
  1: i8 degrees_C;
  2: double lumens;
  3: i8 percent;
  4: i16 kPa;
  5: i16 dBm;
  6: i64 count;
}
```
* Then, we have to use a tag to specify the type of the sensor. Thrift provides enum type to accomplish this:

``` c++
enum Sensor {
  ALL = 0,
  TEMP = 1,
  LIGHT = 2,
  HUMIDITY = 3,
  PRESSURE = 4,
  VIBRATION = 5,
  COUNTER = 6,
}
```
* We are also interested in when a sample was recorded, so we'd like to pack a timestamp along with the sample value. These form a complete sample data:

``` c++
struct TimeStamp {
  1: i64 tv_sec;
  2: i64 tv_nsec;
}

struct Sample {
  1: TimeStamp time;
  2: SampleValue value;
}
```
* And finally the operations, which are described by a service in Thrift. Let's assume that each sensor has a ring buffer for storing the last N samples.

``` c++
service WeatherStation {
  /** @brief Display @p message on the LCD */
  void displayMessage(1: string message) throws (1: WSException error),
 
  /**
   * @brief Set buffer size to @p size
   * Size can be `{ALL: 42}` or `{TEMP: 42, LIGHT: 73}`.
   * @param size a buffer size
   * @throw WSException when an invalid size is provided
   */
  void setBufferSize(1: map<Sensor,i64> size) throws (1: WSException error),
 
  /**
   * @brief Set sample rate to @p Hz
   * Hz can be `{ALL: 42}` or `{TEMP: 0.001, LIGHT: 4242}`.
   * @param Hz sample rate in Hertz
   * @throw WSException when an invalid sample rate is provided
   */
  void setSampleRate(1: map<Sensor,double> Hz) throws (1: WSException error),
 
  /**
   * @brief Get samples for one or more sensors
   * Each returned list may be of different lengths due to varying sample rates
   * @param sensor either ALL or a specific subset of sensors
   * @return a list of samples for each requested @p sensor
   */
  map<Sensor,list<Sample>> getSamples(1: set<Sensor> sensor) throws (1: WSException error),
 
  /** @brief Update firmware with @p file */
  void updateFirmware(1: binary file) throws (1: WSException error),
}
```
The syntax is very similar to a C++ function declaration, except the index before the parameter. You may wonder what is the `WSException` after the parameter list. This is how to specify the exception can be thrown by a method. We can define `WSException` like this:
``` c++
exception WSException {
  1: i16 code;
  2: string message;
}
```
That's all of the IDL part and all of the boilerplate code can be generated with this. Every time the service has to be extended or modified, simply edit a few lines of IDL file and you get the service structure updated. There's no doubt that this is much more cleaner and maintainable than taking care of all the stuff manually.

## Why Use Thrift?
With the weather station example, you already had a general idea about how Thrift can be helpful, but you will also get advantages much beyond that:

* Layered architecture and useful components enable flexible combination of features
* Use a common serialization / deserialization format everywhere, regardless of transport
* Let developers focus on application logic instead of boilerplate code
* A rich set of primitive and complex types like i32,  string, binary, double, list, set, and map.
* User-defined structures, enumerations, and optional fields
* Code generation: reduce maintained LOC by a factor of 100
* Eliminate common mistakes encountered in data communication
* Avoid re-inventing the wheel

![Apache Thrift layered architecture](https://github.com/apache/thrift/raw/master/doc/images/thrift-layers.png)

## Try it yourself
Sounds interesting, but have no previous experience on Thrift? We have a simple example to get you covered. Check [the sample application](https://github.com/zephyrproject-rtos/gsoc-2022-thrift/blob/main/samples/lib/thrift/) to see how to write a simple Thrift IDL and wire everything up together. It can be run on QEMU so no extra hardware is required. To use this module in your project, simply add it to the manifest, or use a submanifest as our [README](https://github.com/zephyrproject-rtos/gsoc-2022-thrift/blob/main/README.md) page suggested. To learn more about Thrift, please refer to its [official documentation](https://thrift.apache.org/docs/).

## My Contributions
Before I started contributing, mentor [Christopher Friedt](https://github.com/cfriedt) managed to make the essential parts of thrift run on Zephyr, and the main effort of mine is integrating 3 of the components in Thrift, which was chosen based on the importance of each feature to the IoTs applications. The first one is compact protocol, whcih has no external dependency and was kinda work-out-of-box. Major challenges came from the later two, as we will see below.

### Zlib transport ([#123](https://github.com/zephyrproject-rtos/gsoc-2022-thrift/pull/123))
As its name suggests, Zlib transport applies [Zlib](https://www.rfc-editor.org/rfc/rfc1950) compression to the payload to speedup the transportation. This is important to IoTs devices connected wirelessly and powered by battery, as radio-frequency communication is very power-demanding. Obviously, this feature depends on Zlib for compression, while Zlib is too heavy for many embedded platforms. Therefore, the idea was to replace Zlib with a lightweight alternative, and unsurprisingly there are such libraries available - but not really. A subtle feature of Zlib is the capability of processing chunks of incomplete data on-the-fly, and this is required by the Zlib transport. This is formally called as streaming mode, which requires additional care to intermediate states and data alignment. It turned out that there's no available library supports both compression and decompression of Zlib data in streaming mode, and most of the existing libraries exposed interfaces very different from Zlib. After assessing the workload, It was decided to make a new library based on two existing works. The first one is [uzlib](https://github.com/pfalcon/uzlib), which provided the compressor, and an encapsulation has to be written to add the streaming capability to it. The second one is [Inflater](https://github.com/martin-rizzo/Inflater), which is the base of the decompressor. Combing this two parts together, a lightweight compression library was created. It was named as [muzic](https://github.com/SdtElectronics/muzic), where "mu" means tiny, and "zic" means Zlib-interface-compatible. Benchmarks showed significant saving of ROM and RAM with muzic in comparison with zlib, portraying it as a strong alternative to zlib on embedded platforms.

### TLS transport ([#126](https://github.com/zephyrproject-rtos/gsoc-2022-thrift/pull/126))
For IoTs services deployed on a open network, you definitely want to have the data encrypted and have the peer verified. This is when TLS transport becomes handy. With the TLS protocol originally implemented by OpenSSL, the TLS protocol also faced the same issue  of resource requirement. The solution was a bit simpler than what was done for the Zlib transport as the alternative implementation of TLS has already been integrated in Zephyr: the [mbedTLS](https://tls.mbed.org/) library. Thanks to this, the integration of TLS transport can be done by substituting OpenSSL APIs, without "inventing your own cryptography" (well, "implementing your own cryptography" could be more precise here), which is known to be risky for laymen.

### Additional Work
Besides the integration work described above, more has been done to make this project a proper module for Zephyr:
* Code style checking in CI ([#87](https://github.com/zephyrproject-rtos/gsoc-2022-thrift/issues/87))
* Rewrite tests using [the new ztest APIs](https://docs.zephyrproject.org/latest/develop/test/ztest.html) ([#131](https://github.com/zephyrproject-rtos/gsoc-2022-thrift/issues/131))
* Leveraged Zephyr's `testcase.yaml` and `sample.yaml` for testing multiple configs with [twister](https://docs.zephyrproject.org/latest/develop/test/twister.html) ([#92](https://github.com/zephyrproject-rtos/gsoc-2022-thrift/issues/92))

## Future Work
* C language support

    C is not yet supported as the Thrift C library depends on [glib](https://docs.gtk.org/glib/), which is licensed under LGPL-2.1. As a Zephyr module, we want to stick to permissive licenses compatible with Apache-2.0. If this dependency issue can be solved in the future, it's nice to have C supported as well.

* Other language support

    Other than C and C++, thrift also supports various mainstream languages including lua and rust. It is even possible to use python with [MicroPython](https://micropython.org/) on MCUs.

* Additional Transports and Protocols

    Components integrated to the module in this summer project are useful to IoTs applications, but they only constitute a small portion of the complete set. Other components like HTTP transport and JSON protocol are also nice to have.

## Conclusion
With all of its device drivers, network stacks and rich libraries, Zephyr stands out as the foundation of the next-generation IoTs products. Now that the Thrift is introduced to the Zephyr ecosystem, not only the hardware level of details are hidden, but also the low-level network communication is abstracted away. IoTs developers can focus on the logic of their services and iterate rapidly even on the most resource-constrained platforms. Finally, as a part of the Google Summer of Code (GSoC) 2022 program, this project wouldn't success without the support from the GSoC and Zephyr community. Especially, I want to thank two mentors [Christopher Friedt](https://github.com/cfriedt) and [Stephanos Ioannidis](https://github.com/stephanosio) who gave this brilliant project idea and helped me throughout the coding period. Hope you'll enjoy our work!
