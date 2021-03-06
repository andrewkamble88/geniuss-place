---
title: "Monitoring the internal status of the software receiver"
permalink: /docs/tutorials/monitoring-software-receiver-internal-status/
excerpt: "This tutorial describes how to monitor the internal status of GNSS-SDR."
author_profile: false
header:
  teaser: /assets/images/gnss-sdr_monitoring_teaser.png
tags:
  - tutorial
sidebar:
  nav: "docs"
toc: true
toc_sticky: true
---


## Introduction

This guide assumes that GNSS-SDR and its software dependencies are already installed on your system, otherwise please check out the [building guide]({{ "/build-and-install/" | relative_url }}) and the [README.md](https://github.com/gnss-sdr/gnss-sdr/blob/master/README.md) file for more details on how to install GNSS-SDR.
{: .notice--info}

Since the introduction of the [Monitor]({{ "/docs/sp-blocks/monitor/" | relative_url }}) block, GNSS-SDR offers a mechanism for monitoring the status of the software receiver in real-time by providing access to 25 internal parameters that tell us about the performance of each channel. The complete list of parameters is documented [here]({{ "/docs/sp-blocks/monitor/#exposed-internal-parameters" | relative_url }}).

In this article we are going to learn how to create a minimal monitoring client application written in C/C++ that will print and update the PRN, CN0 and Doppler frequency shift for each channel on a terminal window while the receiver is running with the Monitor block activated.

The Monitor block implements this mechanism using the binary serialization format provided by the [Boost.Serialization](https://www.boost.org/doc/libs/1_68_0/libs/serialization/doc/index.html) library and the networking functions from the [Boost.Asio](https://www.boost.org/doc/libs/1_68_0/doc/html/boost_asio.html) library. The following diagram can help us to better understand how it works.

{% capture fig_img1 %}
![Client application monitoring GNSS-SDR]({{ "/assets/images/gnss-sdr_monitoring_block_diagram.png" | relative_url }})
{% endcapture %}

<figure>
  {{ fig_img1 | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption>The GNSS-SDR monitoring mechanism uses a binary serialization format.</figcaption>
</figure>

The colored boxes represent Gnss_Synchro objects moving across the receiver chain. These objects are special containers that hold a set of variables which capture the internal state of the receiver. Each color represents a different channel. When these objects reach the [PVT]({{ "/docs/sp-blocks/pvt/" | relative_url }}) block, they are consumed. Therefore they are not visible from the outside, as they do not exit the receiver. This is where the Monitor block comes into play. Its purpose is to stream these objects to the outside world using a binary serialization format. This stream is sent over UDP from a source port to a destination port that can either be on the same machine or on a different one.

Finally, at the other end, the monitoring client deserializes the Gnss_Synchro objects from the binary stream. Then we can access their member variables and use them for implementing our monitoring logic. In this exercise, we will simply print some of these parameters on the terminal.

Keep in mind that, in order to successfully deserialize the objects, we will need to include the Gnss_Synchro class and its dependecies (Gnss_Signal and Gnss_Satellite) to build our monitoring client.

Gnss_Synchro class
 * [https://github.com/gnss-sdr/gnss-sdr/blob/next/src/core/system_parameters/gnss_synchro.h](https://github.com/gnss-sdr/gnss-sdr/blob/next/src/core/system_parameters/gnss_synchro.h)

Gnss_Signal class
 * [https://github.com/gnss-sdr/gnss-sdr/blob/next/src/core/system_parameters/gnss_signal.h](https://github.com/gnss-sdr/gnss-sdr/blob/next/src/core/system_parameters/gnss_signal.h)
 * [https://github.com/gnss-sdr/gnss-sdr/blob/next/src/core/system_parameters/gnss_signal.cc](https://github.com/gnss-sdr/gnss-sdr/blob/next/src/core/system_parameters/gnss_signal.cc)

Gnss_Satellite class
 * [https://github.com/gnss-sdr/gnss-sdr/blob/next/src/core/system_parameters/gnss_satellite.h](https://github.com/gnss-sdr/gnss-sdr/blob/next/src/core/system_parameters/gnss_satellite.h)
 * [https://github.com/gnss-sdr/gnss-sdr/blob/next/src/core/system_parameters/gnss_satellite.cc](https://github.com/gnss-sdr/gnss-sdr/blob/next/src/core/system_parameters/gnss_satellite.cc)


## Building a minimal monitoring client application

### Install dependencies
Copy and paste the following line in a terminal:

```bash
$ sudo apt install build-essential cmake libboost-dev libboost-system-dev \
   libboost-serialization-dev libncurses5-dev libncursesw5-dev
```
This will install the GCC/g++ compiler, the CMake build system and the Boost and NCurses libraries.

### Download the required classes

Create a new directory in the home folder for storing all the source files of our project:

```bash
$ mkdir monitoring-client
$ cd monitoring-client
```

As mentioned earlier, since the Gnss_Synchro class depends on the Gnss_Signal class, and the latter depends on the Gnss_Satellite class, we must include all three of them in our project.

Use the following command to download the class files from the GNSS-SDR GitHub repository into the project folder:

```bash
$ wget https://github.com/gnss-sdr/gnss-sdr/raw/next/src/core/system_parameters/gnss_satellite.h \
   https://github.com/gnss-sdr/gnss-sdr/raw/next/src/core/system_parameters/gnss_satellite.cc \
   https://github.com/gnss-sdr/gnss-sdr/raw/next/src/core/system_parameters/gnss_signal.h \
   https://github.com/gnss-sdr/gnss-sdr/raw/next/src/core/system_parameters/gnss_signal.cc \
   https://github.com/gnss-sdr/gnss-sdr/raw/next/src/core/system_parameters/gnss_synchro.h
```

### Create the deserializer class

Open your IDE or text editor of choice, create a new class and call it Gnss_Synchro_Udp_Source. This class will be in charge of deserializing the Gnss_Synchro objects from the UDP stream.

Define the class header file first: gnss_synchro_udp_source.h

```cpp
#ifndef GNSS_SYNCHRO_UDP_SOURCE_H_
#define GNSS_SYNCHRO_UDP_SOURCE_H_

#include <boost/asio.hpp>
#include "gnss_synchro.h"

class Gnss_Synchro_Udp_Source
{
public:
    Gnss_Synchro_Udp_Source(const unsigned short& port);
    bool read_gnss_synchro(std::vector<Gnss_Synchro>& stocks);
    void populate_channels(std::vector<Gnss_Synchro> stocks);
    bool print_table();

private:
    boost::asio::io_service io_service;
    boost::asio::ip::udp::socket socket;
    boost::system::error_code error;
    boost::asio::ip::udp::endpoint endpoint;
    std::vector<Gnss_Synchro> stocks;
    std::map<int, Gnss_Synchro> channels;
};

#endif /* GNSS_SYNCHRO_UDP_SOURCE_H_ */
```

We are going to use 6 member variables:

|----------
|  **Variable Name**  | **Description** |
|:-:|:--|    
|--------------
| `io_service` | Abstraction of the operating system interfaces. (See [io_service](https://www.boost.org/doc/libs/1_68_0/doc/html/boost_asio/reference/io_service.html)). |
| `socket` | The UDP socket. (See [ip::udp::socket](https://www.boost.org/doc/libs/1_68_0/doc/html/boost_asio/reference/ip__udp/socket.html)). |
| `error` | Operating system-specific errors. (See [boost::system::error_code](https://theboostcpplibraries.com/boost.system)). |
| `endpoint` | Endpoint that will be associated with the UDP socket. (See [ip::udp::endpoint](https://www.boost.org/doc/libs/1_68_0/doc/html/boost_asio/reference/ip__udp/endpoint.html)). |
| `stocks` | Vector container of Gnss_Synchro objects received from the socket. |
| `channels` | Map container of Gnss_Synchro objects indexed by their `Channel_ID`. |
|----------

and 4 member functions:

|----------
|  **Function Name**  | **Description** |
|:-:|:--|    
|--------------
| `Gnss_Synchro_Udp_Source` | Constructor. Opens and binds the `socket` to the `endpoint`. |
| `read_gnss_synchro` | Fills the `stocks` vector with the latest deserialized Gnss_Synchro objects. |
| `populate_channels` | This function inserts the latest Gnss_Synchro objects from the `stocks` vector container into the `channels` map conatiner. |
| `print_table` | Prints the contents of the `channels` map in a table on the terminal screen. |
|----------

Now let's go ahead and write the functions in the implementation file: gnss_synchro_udp_source.cc

First, add the include block:

```cpp
#include "gnss_synchro_udp_source.h"
#include <boost/archive/binary_iarchive.hpp>
#include <boost/serialization/vector.hpp>
#include <sstream>
#include <ncurses.h>
```

Next, implement the constructor. Open the socket and bind it to the endpoint:

```cpp
Gnss_Synchro_Udp_Source::Gnss_Synchro_Udp_Source(const unsigned short& port) : socket{io_service},
                                                                               endpoint{boost::asio::ip::udp::v4(), port}
{
    socket.open(endpoint.protocol(), error);  // Open socket.
    socket.bind(endpoint, error);             // Bind the socket to the given local endpoint.
}
```

Now let's implement the `read_gnss_synchro` function. We need to create a buffer of memory and pass it to the [`receive`](https://www.boost.org/doc/libs/1_68_0/doc/html/boost_asio/reference/basic_datagram_socket/receive.html) function. Since this function is synchronous, the program execution will stop here waiting for incoming data. Once some data is received, the socket stores it in the buffer. The `bytes` variable keeps track of the received number of bytes. We use this information to extract the binary data from the memory buffer into a string of ones and zeros. After some intermediate steps, we end up deserializing the archived data (a vector container of Gnss_Synchro objects) and save it in the `stocks` variable.

```cpp
bool Gnss_Synchro_Udp_Source::read_gnss_synchro(std::vector<Gnss_Synchro>& stocks)
{
    char buff[1500];  // Buffer for storing the received data.

    // This call will block until one or more bytes of data has been received.
    int bytes = socket.receive(boost::asio::buffer(buff));

    try
        {
            std::string archive_data(&buff[0], bytes);
            std::istringstream archive_stream(archive_data);
            boost::archive::binary_iarchive archive(archive_stream);

            // Deserialize a stock of Gnss_Synchro objects from the binary archive.
            archive >> stocks;
        }
    catch (std::exception& e)
        {
            return false;
        }

    return true;
}
```

The implementation of `populate_channels` is straight forward. The function receives a vector of Gnss_Synchro objects as a parameter and inserts each object into the `channels` map container based on the `Channel_ID`. We only allow objects with a sampling frequency different from zero into the map container.

```cpp
void Gnss_Synchro_Udp_Source::populate_channels(std::vector<Gnss_Synchro> stocks)
{
    for (std::size_t i = 0; i < stocks.size(); i++)
        {
            Gnss_Synchro ch = stocks[i];
            if (ch.fs != 0)  // Channel is valid.
                {
                    channels[ch.Channel_ID] = ch;
                }
        }
}
```

Lastly, the `print_table` function calls the `read_gnss_synchro` and `populate_channels` functions, and prints a text table with the `printw` function from NCurses.

```cpp
bool Gnss_Synchro_Udp_Source::print_table()
{
    if (read_gnss_synchro(stocks))
        {
            populate_channels(stocks);

            clear();  // Clear the screen.

            // Print table header.
            attron(A_REVERSE);
            printw("%3s%6s%14s%17s\n", "CH", "PRN", "CN0 [dB-Hz]", "Doppler [Hz]");
            attroff(A_REVERSE);

            // Print table contents.
            for (auto const& ch : channels)
                {
                    int channel_id = ch.first;      // Key
                    Gnss_Synchro data = ch.second;  // Value

                    printw("%3d%6d%14f%17f\n", channel_id, data.PRN, data.CN0_dB_hz, data.Carrier_Doppler_hz);
                }
            refresh();  // Update the screen.
        }
    else
        {
            return false;
        }

    return true;
}
```

The complete implementation file should look like this:

```cpp
#include "gnss_synchro_udp_source.h"
#include <boost/archive/binary_iarchive.hpp>
#include <boost/serialization/vector.hpp>
#include <sstream>
#include <ncurses.h>

Gnss_Synchro_Udp_Source::Gnss_Synchro_Udp_Source(const unsigned short& port) : socket{io_service},
                                                                               endpoint{boost::asio::ip::udp::v4(), port}
{
    socket.open(endpoint.protocol(), error);  // Open socket.
    socket.bind(endpoint, error);             // Bind the socket to the given local endpoint.
}

bool Gnss_Synchro_Udp_Source::read_gnss_synchro(std::vector<Gnss_Synchro>& stocks)
{
    char buff[1500];  // Buffer for storing the received data.

    // This call will block until one or more bytes of data has been received.
    int bytes = socket.receive(boost::asio::buffer(buff));

    try
        {
            std::string archive_data(&buff[0], bytes);
            std::istringstream archive_stream(archive_data);
            boost::archive::binary_iarchive archive(archive_stream);

            // Deserialize a stock of Gnss_Synchro objects from the binary archive.
            archive >> stocks;
        }
    catch (std::exception& e)
        {
            return false;
        }

    return true;
}

void Gnss_Synchro_Udp_Source::populate_channels(std::vector<Gnss_Synchro> stocks)
{
    for (std::size_t i = 0; i < stocks.size(); i++)
        {
            Gnss_Synchro ch = stocks[i];
            if (ch.fs != 0)  // Channel is valid.
                {
                    channels[ch.Channel_ID] = ch;
                }
        }
}

bool Gnss_Synchro_Udp_Source::print_table()
{
    if (read_gnss_synchro(stocks))
        {
            populate_channels(stocks);

            clear();  // Clear the screen.

            // Print table header.
            attron(A_REVERSE);
            printw("%3s%6s%14s%17s\n", "CH", "PRN", "CN0 [dB-Hz]", "Doppler [Hz]");
            attroff(A_REVERSE);

            // Print table contents.
            for (auto const& ch : channels)
                {
                    int channel_id = ch.first;      // Key
                    Gnss_Synchro data = ch.second;  // Value

                    printw("%3d%6d%14f%17f\n", channel_id, data.PRN, data.CN0_dB_hz, data.Carrier_Doppler_hz);
                }
            refresh();  // Update the screen.
        }
    else
        {
            return false;
        }

    return true;
}
```

### Create the main function

Create a new file and implement the main function: main.cc

```cpp
#include "gnss_synchro_udp_source.h"
#include <boost/lexical_cast.hpp>
#include <iostream>
#include <chrono>
#include <thread>
#include <ncurses.h>

int main(int argc, char* argv[])
{
    try
        {
            // Check command line arguments.
            if (argc != 2)
                {
                    // Print help.
                    std::cerr << "Usage: monitoring-client <port>" << std::endl;
                    return false;
                }

            unsigned short port = boost::lexical_cast<unsigned short>(argv[1]);
            Gnss_Synchro_Udp_Source udp_source(port);

            initscr();  // Initialize ncurses.
            printw("Listening on port %d UDP...\n", port);
            refresh();  // Update the screen.

            while (true)
                {
                    udp_source.print_table();
                }
        }
    catch (std::exception& e)
        {
            std::cerr << e.what() << std::endl;
        }

    return true;
}
```



### Build the source code

Create the CMakeLists.txt file. This file contains a set of directives and instructions describing the project's source files and targets.

```bash
cmake_minimum_required (VERSION 2.8)
project (monitoring-client CXX C)

set (CMAKE_CXX_STANDARD 11)

set(Boost_USE_STATIC_LIBS OFF)
find_package(Boost COMPONENTS system serialization REQUIRED)
if(NOT Boost_FOUND)
     message(FATAL_ERROR "Fatal error: Boost required.")
endif(NOT Boost_FOUND)

set (CURSES_NEED_NCURSES TRUE)
find_package(Curses REQUIRED)
if(NOT CURSES_FOUND)
     message(FATAL_ERROR "Fatal error: NCurses required.")
endif(NOT CURSES_FOUND)

include_directories(
        ${CMAKE_SOURCE_DIR}
        ${Boost_INCLUDE_DIRS}
        ${CURSES_INCLUDE_DIRS}
     )

add_library(monitoring_lib ${CMAKE_SOURCE_DIR}/gnss_synchro_udp_source.cc)

ADD_EXECUTABLE(monitoring-client ${CMAKE_SOURCE_DIR}/main.cc)
target_link_libraries(monitoring-client monitoring_lib pthread ${Boost_LIBRARIES})
target_link_libraries(monitoring-client monitoring_lib pthread ${CURSES_LIBRARIES})
```

Save it in the project folder alongside the project source files.

Next, create a build folder inside the project folder:

```bash
$ mkdir build
```

We can make use of the `tree` command to list the contents of our project folder in a tree-like format. This is how it should look at this stage:

```bash
$ tree
.
├── build
├── CMakeLists.txt
├── gnss_satellite.cc
├── gnss_satellite.h
├── gnss_signal.cc
├── gnss_signal.h
├── gnss_synchro.h
├── gnss_synchro_udp_source.cc
├── gnss_synchro_udp_source.h
└── main.cc

1 directory, 9 files
```

Finally go ahead and build the source code:

```bash
$ cd build
$ cmake ../
$ make
```

The `monitoring-client` executable will be created in the build folder. Try running it with no arguments. It should print the usage help:

```bash
$ ./monitoring-client
Usage: monitoring-client <port>
```

Our monitoring client is ready. We have completed the first half of this tutorial. Now let's go ahead and configure the receiver.

Leave the terminal window open, as we will come back to it later, and switch to a new terminal window.


## Activation of the Monitor block in the software receiver

In order to run the receiver, we are going to use the same signal source file that is used in the [first position fix]({{ "/my-first-fix/" | relative_url }}) tutorial.

It is convenient to store the signal file and the receiver configuration in a separate directory from the project folder. Create a work directory in your home folder:

```bash
$ cd ~
$ mkdir work
$ cd work
```

Download the [file]({{ "/my-first-fix/#step-2-download-a-file-of-raw-signal-samples" | relative_url }}) containing the GNSS raw signal samples. This can be done directly from the terminal:

```bash
$ wget https://sourceforge.net/projects/gnss-sdr/files/data/2013_04_04_GNSS_SIGNAL_at_CTTC_SPAIN.tar.gz
$ tar -zxvf 2013_04_04_GNSS_SIGNAL_at_CTTC_SPAIN.tar.gz
```

Copy the [configuration file]({{ "/my-first-fix/#step-3-configure-gnss-sdr" | relative_url }}), paste it in a text editor and add the following fragment at the end of the configuration file to activate the Monitor block:

```ini
;######### MONITOR CONFIG ############
Monitor.enable_monitor=true
Monitor.output_rate_ms=1000
Monitor.client_addresses=127.0.0.1
Monitor.udp_port=1234
```

We will stream the receiver internal parameters to the localhost address on port 1234 UDP with a decimation factor of $$ N = 1000 $$, so that we get status updates at roughly once a second.

The complete configuration file should look like this:

```ini
[GNSS-SDR]

;######### GLOBAL OPTIONS ##################
GNSS-SDR.internal_fs_sps=2000000

;######### SIGNAL_SOURCE CONFIG ############
SignalSource.implementation=File_Signal_Source
SignalSource.filename=/home/your-username/work/data/2013_04_04_GNSS_SIGNAL_at_CTTC_SPAIN.dat
SignalSource.item_type=ishort
SignalSource.sampling_frequency=4000000
SignalSource.freq=1575420000
SignalSource.samples=0

;######### SIGNAL_CONDITIONER CONFIG ############
SignalConditioner.implementation=Signal_Conditioner

;######### DATA_TYPE_ADAPTER CONFIG ############
DataTypeAdapter.implementation=Ishort_To_Complex

;######### INPUT_FILTER CONFIG ############
InputFilter.implementation=Pass_Through
InputFilter.item_type=gr_complex

;######### RESAMPLER CONFIG ############
Resampler.implementation=Direct_Resampler
Resampler.sample_freq_in=4000000
Resampler.sample_freq_out=2000000
Resampler.item_type=gr_complex

;######### CHANNELS GLOBAL CONFIG ############
Channels_1C.count=8
Channels.in_acquisition=1
Channel.signal=1C

;######### ACQUISITION GLOBAL CONFIG ############
Acquisition_1C.implementation=GPS_L1_CA_PCPS_Acquisition
Acquisition_1C.item_type=gr_complex
Acquisition_1C.threshold=0.008
Acquisition_1C.doppler_max=10000
Acquisition_1C.doppler_step=250

;######### TRACKING GLOBAL CONFIG ############
Tracking_1C.implementation=GPS_L1_CA_DLL_PLL_Tracking
Tracking_1C.item_type=gr_complex
Tracking_1C.pll_bw_hz=40.0;
Tracking_1C.dll_bw_hz=4.0;

;######### TELEMETRY DECODER GPS CONFIG ############
TelemetryDecoder_1C.implementation=GPS_L1_CA_Telemetry_Decoder

;######### OBSERVABLES CONFIG ############
Observables.implementation=Hybrid_Observables

;######### PVT CONFIG ############
PVT.implementation=RTKLIB_PVT
PVT.averaging_depth=100
PVT.flag_averaging=true
PVT.output_rate_ms=10
PVT.display_rate_ms=500

;######### MONITOR CONFIG ############
Monitor.enable_monitor=true
Monitor.output_rate_ms=1000
Monitor.client_addresses=127.0.0.1
Monitor.udp_port=1234
```


## Testing the monitoring client

We are now ready to test our application. Switch back to the other terminal window and start the monitoring client on port 1234:

```bash
$ ./monitoring-client 1234
```

You will see this message printed on the screen:

```bash
Listening on port 1234 UDP...
```

Now start the receiver on the other terminal window:

```bash
$ gnss-sdr -c ./my-first-GNSS-SDR-receiver.conf
```

If all worked fine you should see a table like this:

```bash
CH   PRN   CN0 [dB-Hz]     Doppler [Hz]
 0     1     44.205502      7175.743399
 2    17     43.886524     10032.649712
 3    11     45.290539      5585.268260
 4    20     42.442753      8469.028326
 6    32     43.016476      6550.037773
```

If you see something similar to this... Yay! You are successfully monitoring the internals of your open source software-defined GPS receiver!
{: .notice--success}

{% capture fig_img2 %}
![Client application monitoring GNSS-SDR]({{ "/assets/images/gnss-sdr_and_monitoring-client.png" | relative_url }})
{% endcapture %}

<figure>
  {{ fig_img2 | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption>Client application displaying the values of PRN, CN0 and Doppler for each channel while GNSS-SDR is running simultaneously.</figcaption>
</figure>
