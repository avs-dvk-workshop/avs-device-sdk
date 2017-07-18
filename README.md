## Alexa Client SDK v0.6

This release introduces an implementation of the `Alerts` capability agent with support for timers and alarms, as well as classes used to handle directives and events in the [Systems interface](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/reference/system), and a sample app that demonstrates SDK functionality.

**NOTE**: This release requires a filesystem to store audio assets for timers and alarms.  

Native components for the following capability agents are **not** included in this release and will be added in future releases: `AudioPlayer`, `PlaybackController`, `Speaker`, and  `Settings`. However, it's important to note, this release does support directives from the referenced interfaces.

## Overview

The Alexa Client SDK provides a modern C++ (11 or later) interface for the Alexa Voice Service (AVS) that allows developers to add intelligent voice control to connected products. It is modular and abstracted, providing components to handle discrete functionality such as speech capture, audio processing, and communications, with each component exposing APIs that you can use and customize for your integration.  

The SDK also includes a sample app that demonstrates interactions with AVS.  

* [Common Terms](#common-terms)  
* [SDK Components](#sdk-components)   
* [Minimum Requirements and Dependencies](#minimum-requirements-and-dependencies)  
* [Prerequisites](#prerequisites)  
* [Create an Out-of-Source Build](#create-an-out-of-source-build)  
* [Run AuthServer](#run-authserver)  
* [Run Unit Tests](#run-unit-tests)  
* [Run Integration Tests](#run-integration-tests)  
* [Run the Sample App](#run-the-sample-app)
* [Alexa Client SDK API Documentation(Doxygen)](#alexa-client-sdk-api-documentation)
* [Resources and Guides](#resources-and-guides)  
* [Release Notes](#release-notes)

## Common Terms

* **Interface** - A collection of logically grouped messages called **directives** and **events**, which correspond to client functionality, such speech recognition, audio playback, and volume control.
* **Directives** - Messages sent from AVS that instruct your product to take action.
* **Events** - Messages sent from your product to AVS notifying AVS something has occurred.
* **Downchannel** - A stream you create in your HTTP/2 connection, which is used to deliver directives from AVS to your product. The downchannel remains open in a half-closed state from the device and open from AVS for the life of the connection. The downchannel is primarily used to send cloud-initiated directives to your product.
* **Cloud-initiated Directives** - Directives sent from AVS to your product. For example, when a user adjusts device volume from the Amazon Alexa App, a directive is sent to your product without a corresponding voice request.

## SDK Components

This architecture diagram illustrates the data flows between components that comprise the Alexa Client SDK.

![SDK Architecture Diagram](https://images-na.ssl-images-amazon.com/images/G/01/mobile-apps/dex/alexa/alexa-voice-service/docs/avs-cpp-sdk-architecture-20170601.png)

**Audio Signal Processor (ASP)** - Applies signal processing algorithms to both input and output audio channels. The applied algorithms are designed to produce clean audio data and include, but are not limited to: acoustic echo cancellation (AEC), beam forming (fixed or adaptive), voice activity detection (VAD), and dynamic range compression (DRC). If a multi-microphone array is present, the ASP constructs and outputs a single audio stream for the array.

**Shared Data Stream (SDS)** - A single producer, multi-consumer buffer that allows for the transport of any type of data between a single writer and one or more readers. SDS performs two key tasks: 1) it passes audio data between the audio front end (or Audio Signal Processor), the wake word engine, and the Alexa Communications Library (ACL) before sending to AVS; 2) it passes data attachments sent by AVS to specific capability agents via the ACL.

SDS is implemented atop a ring buffer on a product-specific memory segment (or user-specified), which allows it to be used for in-process or interprocess communication. Keep in mind, the writer and reader(s) may be in different threads or processes.

**Wake Word Engine (WWE)** - Spots wake words in an input stream. It is comprised of two binary interfaces. The first handles wake word spotting (or detection), and the second handles specific wake word models (in this case "Alexa"). Depending on your implementation, the WWE may run on the system on a chip (SOC) or dedicated chip, like a digital signal processor (DSP).

**Audio Input Processor (AIP)** - Handles audio input that is sent to AVS via the ACL. These include on-device microphones, remote microphones, an other audio input sources.

The AIP also includes the logic to switch between different audio input sources. Only one audio input source can be sent to AVS at a given time.

**Alexa Communications Library (ACL)** - Serves as the main communications channel between a client and AVS. The Performs two key functions:

* Establishes and maintains long-lived persistent connections with AVS. ACL adheres to the messaging specification detailed in [Managing an HTTP/2 Conncetion with AVS](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/docs/managing-an-http-2-connection).
* Provides message sending and receiving capabilities, which includes support JSON-formatted text, and binary audio content. For additional information, see [Structuring an HTTP/2 Request to AVS](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/docs/avs-http2-requests).

**Alexa Directive Sequencer Library (ADSL)**: Manages the order and sequence of directives from AVS, as detailed in the [AVS Interaction Model](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/reference/interaction-model#channels). This component manages the lifecycle of each directive, and informs the Directive Handler (which may or may not be a Capability Agent) to handle the message.

See [**Appendix B**](#appendix-b-directive-lifecycle-diagram) for a diagram of the directive lifecycle.

**Activity Focus Manager Library (AFML)**: Provides centralized management of audiovisual focus for the device. Focus is based on channels, as detailed in the [AVS Interaction Model](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/reference/interaction-model#channels), which are used to govern the prioritization of audiovisual inputs and outputs.

Channels can either be in the foreground or background. At any given time, only one channel can be in the foreground and have focus. If multiple channels are active, you need to respect the following priority order: Dialog > Alerts > Content. When a channel that is in the foreground becomes inactive, the next active channel in the priority order moves into the foreground.

Focus management is not specific to Capability Agents or Directive Handlers, and can be used by non-Alexa related agents as well. This allows all agents using the AFML to have a consistent focus across a device.

**Capability Agents**: Handle Alexa-driven interactions; specifically directives and events. Each capability agent corresponds to a specific interface exposed by the AVS API. These interfaces include:

* [SpeechRecognizer](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/reference/speechrecognizer) - The interface for speech capture.
* [SpeechSynthesizer](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/reference/speechsynthesizer) - The interface for Alexa speech output.
* [Alerts](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/reference/alerts) - The interface for setting, stopping, and deleting timers and alarms.
* [AudioPlayer](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/reference/audioplayer) - The interface for managing and controlling audio playback.
* [PlaybackController](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/reference/playbackcontroller) - The interface for navigating a playback queue via GUI or buttons.
* [Speaker](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/reference/speaker) - The interface for volume control, including mute and unmute.
* [System](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/reference/system) - The interface for communicating product status/state to AVS.

## Minimum Requirements and Dependencies

* C++ 11 or later
* [GNU Compiler Collection (GCC) 4.8.5](https://gcc.gnu.org/) or later **OR** [Clang 3.3](http://clang.llvm.org/get_started.html) or later
* [CMake 3.1](https://cmake.org/download/) or later
* [libcurl 7.50.2](https://curl.haxx.se/download.html) or later
* [nghttp2 1.0](https://github.com/nghttp2/nghttp2) or later
* [OpenSSL 1.0.2](https://www.openssl.org/source/) or later
* [Doxygen 1.8.13](http://www.stack.nl/~dimitri/doxygen/download.html) or later (required to build API documentation)  
* **NEW** - [SQLite 3.19.3](https://www.sqlite.org/download.html) or later (required for Alerts)  
* **NEW** - For Alerts to work as expected, the device system clock must be set to UTC time. We recommend using `NTP` to do this.

**MediaPlayer Reference Implementation Dependencies**  
Building the reference implementation of the `MediaPlayerInterface` (the class `MediaPlayer`) is optional, but requires:  
* [GStreamer 1.8](https://gstreamer.freedesktop.org/documentation/installing/index.html) or later and the following GStreamer plug-ins:  
**IMPORTANT NOTE FOR MACOS**: GStreamer 1.10.4 has been validated for macOS. GStreamer 1.12 **does not** work.    
* [GStreamer Base Plugins 1.8](https://gstreamer.freedesktop.org/releases/gst-plugins-base/1.8.0.html) or later.
* [GStreamer Good Plugins 1.8](https://gstreamer.freedesktop.org/releases/gst-plugins-good/1.8.0.html) or later.
* [GStreamer Libav Plugin 1.8](https://gstreamer.freedesktop.org/releases/gst-libav/1.8.0.html) or later **OR**
[GStreamer Ugly Plugins 1.8](https://gstreamer.freedesktop.org/releases/gst-plugins-ugly/1.8.0.html) or later, for decoding MP3 data.

**NOTE**: The plugins may depend on libraries which need to be installed as well for the GStreamer based `MediaPlayer` to work correctly.  

**Sample App Dependencies**  
Building the sample app is optional, but requires:  
* [PortAudio v190600_20161030](http://www.portaudio.com/download.html)
* GStreamer

**NOTE**: The sample app will still work with or without the SDK being built with a wake word engine. If built without a wake word engine, hands-free mode will be disabled in the sample app.

## Prerequisites

Before you create your build, you'll need to install some software that is required to run `AuthServer`. `AuthServer` is a minimal authorization server built in Python using Flask. It provides an easy way to obtain your first refresh token, which will be used for integration tests and obtaining access token that are required for all interactions with AVS.

**IMPORTANT NOTE**: `AuthServer` is for testing purposed only. A commercial product is expected to obtain Login with Amazon (LWA) credentials using the instructions provided on the Amazon Developer Portal for **Remote Authorization** and **Local Authorization**. For additional information, see [AVS Authorization](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/content/avs-api-overview#authorization).

### Step 1: Install `pip`

If `pip` isn't installed on your system, follow the detailed install instructions [here](https://packaging.python.org/installing/#install-pip-setuptools-and-wheel).

### Step 2: Install `flask` and `requests`

For Windows run this command:

```
pip install flask requests
```

For Unix/Mac run this command:

```
pip install --user flask requests
```

### Step 3: Obtain Your Device Type ID, Cliend ID, and Client Secret

If you haven't already, follow these instructions to [register a product and create a security profile](https://github.com/alexa/alexa-avs-sample-app/wiki/Create-Security-Profile).

Make sure you note the following, you'll need these later when you configure `AuthServer`:

* Device Type ID
* Client ID
* Client Secret

**IMPORTANT NOTE**: Make sure that you've set your **Allowed Origins** and **Allowed Return URLs** in the **Web Settings Tab**:
* Allowed Origins: http://localhost:3000
* Allowed Return URLs: http://localhost:3000/authresponse

## Create an Out-of-Source Build

The following instructions assume that all requirements and dependencies are met and that you have cloned the repository (or saved the tarball locally).

### CMake Build Types and Options

The following build types are supported:

* `DEBUG` - Shows debug logs with `-g` compiler flag.
* `RELEASE` - Adds `-O2` flag and removes `-g` flag.
* `MINSIZEREL` - Compiles with `RELEASE` flags and optimizations (`-O`s) for a smaller build size.

To specify a build type, use this command in place of step 4 below (see [Build for Generic Linux](#generic-linux) or [Build for macOS](#build-for-macosß)):
`cmake <path-to-source> -DCMAKE_BUILD_TYPE=<build-type>`

### Build with a Wake Word Detector

The Alexa Client SDK supports wake word detectors from [Sensory](https://github.com/Sensory/alexa-rpi) and [KITT.ai](https://github.com/Kitt-AI/snowboy/). The following options are required to build with a wake word detector, please replace `<wake-word-name>` with `SENSORY` for Sensory, and `KITTAI` for KITT.ai:

* `-D<wake-word-name>_KEY_WORD_DETECTOR=<ON or OFF>` - Specifies if the wake word detector is enabled or disabled during build.
* `-D<wake-word-name>_KEY_WORD_DETECTOR_LIB_PATH=<path-to-lib>` - The path to the wake word detector library.
* `-D<wake-word-name>_KEY_WORD_DETECTOR_INCLUDE_DIR=<path-to-include-dir>` - The path to the wake word detector include directory.

**Note**: To list all available CMake options, use the following command: `-LH`.

#### Sensory

If using the Sensory wake word detector, version [5.0.0-beta.10.2](https://github.com/Sensory/alexa-rpi) or later is required.

This is an example `cmake` command to build with Sensory:

```
cmake <path-to-source> -DSENSORY_KEY_WORD_DETECTOR=ON -DSENSORY_KEY_WORD_DETECTOR_LIB_PATH=.../alexa-rpi/lib/libsnsr.a -DSENSORY_KEY_WORD_DETECTOR_INCLUDE_DIR=.../alexa-rpi/include
```

Note that you may need to license the Sensory library for use prior to running cmake and building it into the SDK. A script to license the Sensory library can be found on the Sensory [Github](https://github.com/Sensory/alexa-rpi) page under `bin/license.sh`.

#### KITT.ai

A matrix calculation library, known as BLAS, is required to use KITT.ai. The following are sample commands to install this library:
* Generic Linux - `apt-get install libatlas-base-dev`
* macOS -  `brew install homebrew/science/openblas`

This is an example `cmake` command to build with KITT.ai:

```
cmake <path-to-source> -DKITTAI_KEY_WORD_DETECTOR=ON -DKITTAI_KEY_WORD_DETECTOR_LIB_PATH=.../snowboy-1.2.0/lib/libsnowboy-detect.a -DKITTAI_KEY_WORD_DETECTOR_INCLUDE_DIR=.../snowboy-1.2.0/include
```

### Build with an implementation of `MediaPlayer`

`MediaPlayer` (the reference implementation of the `MediaPlayerInterface`) is based upon [GStreamer](https://gstreamer.freedesktop.org/), and is not built by default. To build 'MediaPlayer' the `-DGSTREAMER_MEDIA_PLAYER=ON` option must be specified to CMake.

If GStreamer was [installed from source](https://gstreamer.freedesktop.org/documentation/frequently-asked-questions/getting.html), the prefix path provided when building must be specified to CMake with the `DCMAKE_PREFIX_PATH` option. This is an example CMake command:

```
cmake <path-to-source> -DGSTREAMER_MEDIA_PLAYER=ON -DCMAKE_PREFIX_PATH=<path-to-GStreamer-build>
```

### Build with PortAudio (Required to Run the Sample App)  

PortAudio is required to build and run the Alexa Client SDK Sample App. Build instructions are available for [Linux](http://portaudio.com/docs/v19-doxydocs/compile_linux.html) and [macOS](http://portaudio.com/docs/v19-doxydocs/compile_mac_coreaudio.html).  

This is sample CMake command to build the Alexa Client SDK with PortAudio:

```
cmake <path-to-source> -DPORTAUDIO=ON
-DPORTAUDIO_LIB_PATH=<path-to-portaudio-lib>
-DPORTAUDIO_INCLUDE_DIR=<path-to-portaudio-include-dir>
```

For example,
```
cmake <path-to-source> -DPORTAUDIO=ON
-DPORTAUDIO_LIB_PATH=.../portaudio/lib/.libs/libportaudio.a
-DPORTAUDIO_INCLUDE_DIR=.../portaudio/include
```
### Application Settings

The SDK will require a configuration JSON file, an example of which is located in `Integration/AlexaClientSDKConfig.json`. The contents of the JSON should be populated with your product information (which you got from the developer portal when registering a product and creating a security profile), and the location of your database and sound files. This JSON file is required for the integration tests to work properly, as well as for the Sample App.

The layout of the file is as follows:
```json
{
"authDelegate":{
"deviceTypeId":"<Device Type ID for your device on the Developer portal>",
"clientId":"<ClientID for the security profile of the device>",
"clientSecret":"<ClientSecret for the security profile of the device>",
"deviceSerialNumber":"<a unique number for your device>"
},
"alertsCapabilityAgent": {
"databaseFilePath":"/<path-to-db-directory>/<db-file-name>",
"alarmSoundFilePath":"/<path-to-alarm-sound>/alarm_normal.mp3",
"alarmShortSoundFilePath":"/<path-to-short-alarm-sound>/alarm_short.wav",
"timerSoundFilePath":"/<path-to-timer-sound>/timer_normal.mp3",
"timerShortSoundFilePath":"/<path-to-short-timer-sound>/timer_short.wav"
}
}
```
**NOTE**: The `deviceSerialNumber` is a unique identifier that you create. It is **not** provided by Amazon.
**NOTE**: The audio files for the alerts can be downloaded from [here](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/content/alexa-voice-service-ux-design-guidelines#attention). Note that the Alexa Voice Service UX guidelines mandate that these audio files must be used for Alexa alerts.

### Build for Generic Linux

To create an out-of-source build for Linux:

1. Clone the repository (or download and extract the tarball).
2. Create a build directory out-of-source. **Important**: The directory cannot be a subdirectory of the source folder.
3. `cd` into your build directory.
4. From your build directory, run `cmake` on the source directory to generate make files for the SDK: `cmake <path-to-source-code>`.
5. After you've successfully run `cmake`, you should see the following message: `-- Please fill <path-to-build-directory>/Integration/AlexaClientSDKConfig.json before you execute integration tests.`. Open `Integration/AlexaClientSDKConfig.json` with your favorite text editor and fill in your product information.
6. From the build directory, run `make` to build the SDK.

### Build for macOS

Building for macOS requires some additional setup. Specifically, you need to ensure that you are running the latest version of cURL and that cURL is linked to nghttp2 (the default installation does not).

To recompile cURL, follow these instructions:

1. Install [Homebrew](http://brew.sh/), if you haven't done so already.
2. Install cURL with HTTP2 support:
`brew install curl --with-nghttp2`
3. Force cURL to explicitly link to the updated binary:
`brew link curl --force`
4. Close and reopen terminal.
5. Confirm version and source with this command:
`brew info curl`

To create an out-of-source build for macOS:

1. Clone the repository (or download and extract the tarball).
2. Create a build directory out-of-source. **Important**: The directory cannot be a subdirectory of the source folder.
3. `cd` into your build directory.
4. From your build directory, run `cmake` on the source directory to generate make files for the SDK: `cmake <path-to-source-code>`.
5. After you've successfully run `cmake`, you should see the following message: `-- Please fill <path-to-build-directory>/Integration/AlexaClientSDKConfig.json before you execute integration tests.`. Open `Integration/AlexaClientSDKConfig.json` with your favorite text editor and fill in your product information.
6. From the build directory, run `make` to build the SDK.

## Run AuthServer

After you've created your out-of-source build, the next step is to run `AuthServer` to retrieve a valid refresh token from LWA.

* Run this command to start `AuthServer`:
```
python AuthServer/AuthServer.py
```
You should see a message that indicates the server is running.
* Open your favorite browser and navigate to: `http://localhost:3000`
* Follow the on-screen instructions.
* After you've entered your credentials, the server should terminate itself, and `Integration/AlexaClientSDKConfig.json` will be populated with your refresh token.
* Before you proceed, it's important that you make sure the refresh token is in `Integration/AlexaClientSDKConfig.json`.

## Run Unit Tests

Unit tests for the Alexa Client SDK use the [Google Test](https://github.com/google/googletest) framework. Ensure that the [Google Test](https://github.com/google/googletest) is installed, then run the following command:
`make all test`

Ensure that all tests are passed before you begin integration testing.

### Run Unit Tests with Sensory Enabled

In order to run unit tests for the Sensory wake word detector, the following files must be downloaded from [GitHub](https://github.com/Sensory/alexa-rpi) and placed in `<source dir>/KWD/inputs/SensoryModels` for the integration tests to run properly:

* [`spot-alexa-rpi-31000.snsr`](https://github.com/Sensory/alexa-rpi/blob/master/models/spot-alexa-rpi-31000.snsr)

### Run Unit Tests with KITT.ai Enabled

In order to run unit tests for the KITT.ai wake word detector, the following files must be downloaded from [GitHub](https://github.com/Kitt-AI/snowboy/tree/master/resources) and placed in `<source dir>/KWD/inputs/KittAiModels`:
* [`common.res`](https://github.com/Kitt-AI/snowboy/tree/master/resources)
* [`alexa.umdl`](https://github.com/Kitt-AI/snowboy/tree/master/resources/alexa/alexa-avs-sample-app) - It's important that you download the `alexa.umdl` in `resources/alexa/alexa-avs-sample-app` for the KITT.ai unit tests to run properly.

## Run Integration Tests

Integration tests ensure that your build can make a request and receive a response from AVS.
* All requests to AVS require auth credentials
* The integration tests for Alerts require your system to be in UTC

**Important**: Integration tests reference an `AlexaClientSDKConfig.json` file, which you must create.
See the `Create the AlexaClientSDKConfig.json file` section (above), if you have not already done this.

To exercise the integration tests run this command:
`TZ=UTC make all integration`

### Run Integration Tests with Sensory Enabled

If the project was built with the Sensory wake word detector, the following files must be downloaded from [GitHub](https://github.com/Sensory/alexa-rpi) and placed in `<source dir>/Integration/inputs/SensoryModels` for the integration tests to run properly:

* [`spot-alexa-rpi-31000.snsr`](https://github.com/Sensory/alexa-rpi/blob/master/models/spot-alexa-rpi-31000.snsr)

### Run Integration Tests with KITT.ai

If the project was built with the KITT.ai wake word detector, the following files must be downloaded from [GitHub](https://github.com/Kitt-AI/snowboy/tree/master/resources) and placed in `<source dir>/Integration/inputs/KittAiModels` for the integration tests to run properly:
* [`common.res`](https://github.com/Kitt-AI/snowboy/tree/master/resources)
* [`alexa.umdl`](https://github.com/Kitt-AI/snowboy/tree/master/resources/alexa/alexa-avs-sample-app) - It's important that you download the `alexa.umdl` in `resources/alexa/alexa-avs-sample-app` for the KITT.ai integration tests to run properly.

## Run the Sample App   

**Note**: Building with PortAudio and GStreamer is required.

Before you run the sample app, it's important to note that the application takes two arguments. The first is required, and is the path to `AlexaClientSDKConfig.json`. The second is only required if you are building the sample app with wake word support, and is the path of the of the folder containing the wake word engine models.

Navigate to the `SampleApp/src` folder from your build directory. Then run this command:

```
TZ=UTC ./SampleApp <REQUIRED-path-to-config-json> <OPTIONAL-path-to-wake-word-engine-folder-enclosing-model-files>
```

**Note**: Logging is currently disabled in the Sample App. We plan on updating the Sample App to allow the user to set logging levels when running the app, but at the moment, to turn on logging for the Sample App, the following lines (101-102) in SampleApp/src/main.cpp will need to be commented:
```
alexaClientSDK::avsCommon::utils::logger::ConsoleLogger::instance().setLevel(
alexaClientSDK::avsCommon::utils::logger::Level::NONE);
```
Alternatively, you can change the `NONE` to the desired level of logging.
**Note**: Enabling logging *might* cause Sample App messages and logging messages to become interwoven.

## Alexa Client SDK API Documentation

To build the Alexa Client SDK API documentation, run this command from your build directory: `make doc`.

## Resources and Guides

* [Step-by-step instructions to optimize libcurl for size in `*nix` systems](https://github.com/alexa/alexa-client-sdk/wiki/optimize-libcurl).
* [Step-by-step instructions to build libcurl with mbed TLS and nghttp2 for `*nix` systems](https://github.com/alexa/alexa-client-sdk/wiki/build-libcurl-with-mbed-TLS-and-nghttp2).  

## Appendix A: Memory Profile

This appendix provides the memory profiles for various modules of the Alexa Client SDK. The numbers were observed running integration tests on a machine running Ubuntu 16.04.2 LTS.

| Module | Source Code Size (Bytes) | Library Size RELEASE Build (libxxx.so) (Bytes) | Library Size MINSIZEREL Build (libxxx.so) (Bytes) |
|--------|--------------------------|------------------------------------------------|---------------------------------------------------|
| ACL | 356 KB | 250 KB | 239 KB |
| ADSL | 224 KB | 175 KB | 159 KB |
| AFML | 80 KB | 133 KB | 126 KB |
| ContextManager | 84 KB | 122 KB | 116 KB |
| AIP | 184 KB | 340 KB | 348 KB |
| SpeechSynthesizer | 120 KB | 311 KB | 321 KB |
| AVSCommon | 772 KB | 252 KB | 228 KB |
| AVSUtils | 332 KB | 167 KB | 133 KB |
| Total | 2152 KB | 1750 KB | 1670 KB |

**Runtime Memory**

Unique size set (USS) and proportional size set (PSS) were measured by SMEM while integration tests were run.

| Runtime Memory | Average USS | Max USS (Bytes) | Average PSS | Max PSS (Bytes) |
|----------------|-------------|-----------------|-------------|-----------------|
| ACL | 8 MB | 15 MB | 8 MB | 16 MB |
| ADSL + ACL | 8 MB | 20 MB | 9 MB | 21 MB |
| AIP | 9 MB | 12 MB | 9 MB | 13 MB |
| ** SpeechSynthesizer | 11 MB | 18 MB | 12 MB | 20 MB |

** This test was run using the GStreamer-based MediaPlayer for audio playback.

**Definitions**

* **USS**: The amount of memory that is private to the process and not shared with any other processes.
* **PSS**:  The amount of memory shared with other processes; divided by the number of processes sharing each page.

## Appendix B: Directive Lifecycle Diagram

![Directive Lifecycle](https://images-na.ssl-images-amazon.com/images/G/01/mobile-apps/dex/alexa/alexa-voice-service/docs/avs-directive-lifecycle.png)

## Appendix C: Runtime Configuration of path to CA Certificates

By default libcurl is built with paths to a CA bundle and a directory containing CA certificates.  You can direct the Alexa Client SDK to configure libcurl to use an additional path to directories containing CA certificates via the [CURLOPT_CAPATH](https://curl.haxx.se/libcurl/c/CURLOPT_CAPATH.html) setting.  This is done by adding a `"libcurlUtils/CURLOPT_CAPATH"` entry to the `AlexaClientSDKConfig.json` file.  Here is an example:

```
{
"authDelegate" : {
"clientId" : "INSERT_YOUR_CLIENT_ID_HERE",
"refreshToken" : "INSERT_YOUR_REFRESH_TOKEN_HERE",
"clientSecret" : "INSERT_YOUR_CLIENT_SECRET_HERE"
},
"libcurlUtils" : {
"CURLOPT_CAPATH" : "INSERT_YOUR_CA_CERTIFICATE_PATH_HERE"
}
}
```
**Note** If you want to assure that libcurl is *only* using CA certificates from this path you may need to reconfigure libcurl with the `--without-ca-bundle` and `--without-ca-path` options and rebuild it to suppress the default paths.  See [The libcurl documention](https://curl.haxx.se/docs/sslcerts.html) for more information.

## Release Notes

v0.6 released 7/14/2017:  

* Added a sample app that leverages the SDK.   
* Added an implementation of the `Alerts` capability agent.  
* Added the `DefaultClient` class.  
* Added the following classes to support directives and events in the [`Systems` interface](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/reference/system): `StateSynchronizer`, `EndpointHandler`, and `ExceptionEncounteredSender`.   
* Added unit tests for `ACL`.  
* Updated `MediaPlayer` to play local files given an `std::istream`.    
* Changed build configuration from `Debug` to `Release`.  
* Removed `DeprecatedLogger` class. 
* Known Issues:
    * MediaPlayer: Our `GStreamer` based implementation of `MediaPlayer` is not fully robust, and may result in fatal runtime errors, under the following conditions:
        * Attempting to play multiple simultaneous audio streams
        * Calling `MediaPlayer::play()` and `MediaPlayer::stop()` when the MediaPlayer is already playing or stopped, respectively.
        * Other miscellaneous issues, which will be addressed in the near future
    * `AlertsCapabilityAgent`:
        * This component has been temporarily simplified to work around the known `MediaPlayer` issues mentioned above
        * Fully satisfies the AVS specification except for sending retrospective Events, for example, sending `AlertStarted` for an Alert which rendered when there was no Internet connection
        * This component is not fully thread-safe, however, this will be addressed shortly
        * Alerts currently run indefinitely until stopped manually by the user. This will be addressed shortly by having a timeout value for an alert to stop playing.
        * Alerts do not play in the background when Alexa is speaking, but will continue playing after Alexa stops speaking.
    * `Sample App`:
      * Without the refresh token being filled out in the JSON file, the sample app crashes on start up.
      * Any connection loss during the `Listening` state keeps the app stuck in that state, unless the ongoing interaction is manually stopped by the user.
      * At the end of a shopping list with more than 5 items, the interaction in which Alexa asks the user if he/she would like to hear more does not finish properly.
  * `Tests`:
    * `SpeechSynthesizer` unit tests hang on some older versions of GCC due to a tear down issue in the test suite
    * Intermittent Alerts integration test failures caused by rigidness in expected behavior in the tests

v0.5 released 6/23/2017:

* Updated most SDK components to use new logging abstraction.
* Added a `getConfiguration()` method to `DirectiveHandlerInterface` to register Capability Agents with Directive Sequencer.
* Added ACL stream processing with Pause and redrive.
* Removed the dependency of ACL Library on `Authdelegate`.
* Added an interface to allow ACL to Add/Remove `ConnectionStatusObserverInterface`.
* Fixed compile errors in KittAi, DirectiveHandler and compiler warnings in AIP test.
* Corrected formatting of code in many files.
* Fixes for the following Github issues:
* [MessageRequest callbacks never triggered if disconnected](https://github.com/alexa/alexa-client-sdk/issues/21)
* [AttachmentReader::read() returns ReadStatus::CLOSED if an AttachmentWriter has not been created yet](https://github.com/alexa/alexa-client-sdk/issues/25)

v0.4.1 released 6/9/2017:

* Implemented Sensory wake word detector functionality
* Removed the need for a `std::recursive_mutex` in `MessageRouter`
* Added AIP unit test
* Added `handleDirectiveImmediately` functionality to `SpeechSynthesizer`
* Added memory profiles for:
* AIP
* SpeechSynthesizer
* ContextManager
* AVSUtils
* AVSCommon
* Bug fix for `MessageRouterTest` aborting intermittently
* Bug fix for `MultipartParser.h` compiler warning
* Suppression of sensitive log data even in debug builds. Use cmake parameter -DACSDK_EMIT_SENSITIVE_LOGS=ON to allow logging of sensitive information in DEBUG builds
* Fix crash in ACL when attempting to use more than 10 streams
* Updated MediaPlayer to use `autoaudiosink` instead of requiring `pulseaudio`
* Updated MediaPlayer build to suppport local builds of GStreamer
* Fixes for the following Github issues:
* [MessageRouter::send() does not take the m_connectionMutex](https://github.com/alexa/alexa-client-sdk/issues/5)
* [MessageRouter::disconnectAllTransportsLocked flow leads to erase while iterating transports vector](https://github.com/alexa/alexa-client-sdk/issues/8)
* [Build errors when building with KittAi enabled](https://github.com/alexa/alexa-client-sdk/issues/9)
* [HTTP2Transport race may lead to deadlock](https://github.com/alexa/alexa-client-sdk/issues/10)
* [Crash in HTTP2Transport::cleanupFinishedStreams()](https://github.com/alexa/alexa-client-sdk/issues/17)
* [The attachment writer interface should take a `const void*` instead of `void*`](https://github.com/alexa/alexa-client-sdk/issues/24)

v0.4 updated 5/31/2017:

* Added `AuthServer`, an authorization server implementation used to retrieve refresh tokens from LWA.

v0.4 release 5/24/2017:

* Added the `SpeechSynthesizer`, an implementation of the `SpeechRecognizer` capability agent.
* Implemented a reference `MediaPlayer` based on [GStreamer](https://gstreamer.freedesktop.org/) for audio playback.
* Added the `MediaPlayerInterface` that allows you to implement your own media player.
* Updated `ACL` to support asynchronous receipt of audio attachments from AVS.
* Bug Fixes:
* Some intermittent unit test failures were fixed.
* Known Issues:
* `ACL`'s asynchronous receipt of audio attachments may manage resources poorly in scenarios where attachments are received but not consumed.
* When an `AttachmentReader` does not deliver data for prolonged periods `MediaPlayer` may not resume playing the delayed audio.

v0.3 released 5/17/2017:

* Added the `CapabilityAgent` base class that is used to build capability agent implementations.
* Added the `ContextManager` class that allows multiple Capability Agents to store and access state. These events include `context`, which is used to communicate the state of each capability agent to AVS:
* [`Recognize`](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/reference/speechrecognizer#recognize)
* [`PlayCommandIssued`](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/reference/playbackcontroller#playcommandissued)
* [`PauseCommandIssued`](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/reference/playbackcontroller#pausecommandissued)
* [`NextCommandIssued`](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/reference/playbackcontroller#nextcommandissued)
* [`PreviousCommandIssued`](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/reference/playbackcontroller#previouscommandissued)
* [`SynchronizeState`](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/reference/system#synchronizestate)
* [`ExceptionEncountered`](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/reference/system#exceptionencountered)
* Implemented the `SharedDataStream` (SDS) to asynchronously communicate data between a local reader and writer.
* Added `AudioInputProcessor` (AIP), an implementation of a `SpeechRecognizer` capability agent.
* Added the WakeWord Detector (WWD), which recognizes keywords in audio streams. v0.3 implements a wrapper for KITT.ai.
* Added a new implementation of `AttachmentManager` and associated classes for use with SDS.
* Updated the `ACL` to support asynchronously sending audio to AVS.

v0.2.1 released 5/3/2017:
* Replaced the configuration file `AuthDelegate.config` with `AlexaClientSDKConfig.json`.
* Added the ability to specify a `CURLOPT_CAPATH` value to be used when libcurl is used by ACL and AuthDelegate.  See [**Appendix C**](#appendix-c-runtime-configuration-of-path-to-ca-certificates) for details.
* Changes to ADSL interfaces:
The v0.2 interface for registering directive handlers (`DirectiveSequencer::setDirectiveHandlers()`) was problematic because it canceled the ongoing processing of directives and dropped further directives until it completed. The revised API makes the operation immediate without canceling or dropping any handling.  However, it does create the possibility that `DirectiveHandlerInterface` methods `preHandleDirective()` and `handleDirective()` may be called on different handlers for the same directive.
* `DirectiveSequencerInterface::setDirectiveHandlers()` was replaced by `addDirectiveHandlers()` and `removeDirectiveHandlers()`.
* `DirectiveHandlerInterface::shutdown()` was replaced with `onDeregistered()`.
* `DirectiveHandlerInterface::preHandleDirective()` now takes a `std::unique_ptr` instead of a `std::shared_ptr` to `DirectiveHandlerResultInterface`.
* `DirectiveHandlerInterface::handleDirective()` now returns a bool indicating if the handler recognizes the `messageId`.
* Bug fixes:
* ACL and AuthDelegate now require TLSv1.2.
* `onDirective()` now sends `ExceptionEncountered` for unhandled directives.
* `DirectiveSequencer::shutdown()` no longer sends `ExceptionEncountered()` for queued directives.

v0.2 updated 3/27/2017:
* Added memory profiling for ACL and ADSL.  See [**Appendix A**](#appendix-a-mempry-profile).
* Added command to build API documentation.

v0.2 released 3/9/2017:
* Alexa Client SDK v0.2 released.
* Architecture diagram has been updated to include the ADSL and AMFL.
* CMake build types and options have been updated.
* New documentation for libcurl optimization included.

v0.1 released 2/10/2017:
* Alexa Client SDK v0.1 released.
