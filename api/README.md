# Inno Broker v2.3
This project is part of [DIH2](http://www.dih-squared.eu/). For more information check the RAMP Catalogue entry for the [components](https://github.com/xxx).
 ## Contents

-   [Background](#background)
-   [Requirements](#requirements)
-   [Install](#install)
-   [Usage](#usage)
-   [License](#license)

## Background

Proposed solution allows access to additional production line performance data in an automated manner. Inno system uses this data for performance analysis (reports). The new services delivered within DIH2 include:

1. Real time product Tracking - visualisation of machines’ current status, i.e. uptime, number of cycles completed, time to complete, etc.
2. Early warnings for production station status (monitoring) – sending notifications to shop-floor workers when the robot’s task is near to completion so there is no idling time
3. Data Collection – Automatic data collection which will be used for a concise evaluation of the entire process
4. Performance Analysis – using collected data a real time performance analysis helps improving the OEE KPIs

In order to achive this the broker was required to facilitate the exchange of data between the PLC, cratedb and fiware.

## Requirements

Broker requires Crate database, thus install the following [docker](/docker/) image and also [.net5](https://dotnet.microsoft.com/en-us/download/dotnet/5.0) framework or newer.<br>
Also it requires a preinstalled functional welding robot cell with a Siemens PLC.

## Installation

There is no instalation required, the software is a standalone that can be started by adding it to the windows startup.

## Usage

Details on how to use the software are described under docs in the [Programmers Manual](/docs/)

### API

## License
The Inno West Rose-AP components are licensed under [Apache 2.0](/LICENSE) © 2022 Inno Robotics S.R.L.