# Inno-West
 ROSE-AP automated planning

# Docker Image
This project is part of [DIH2](http://www.dih-squared.eu/). For more information check the RAMP Catalogue entry for the [components](https://github.com/xxx).
 ## Contents

-   [Background](#background)
-   [Requirements](#requirements)
-   [Install](#install)
-   [Configuration](#configuration)
-   [Usage](#usage)
-   [License](#license)

## Background

CrateDB is utilized for data collection.

## Requirements

Requires [docker](https://github.com/docker) preinstalled.

## Installation

[Orion Context Broker](https://github.com/telefonicaid/fiware-orion/tree/master/docker)

To install run this command

```text
docker run -d --name orion1 -p 1026:1026 fiware/orion
```

[CrateDB](https://github.com/crate/crate)

To install run this command

```text
docker run --publish 4200:4200 --publish 5432:5432 crate -Cdiscovery.type=single-node
```

## Configuration

To configure the CrateDB two tables need to be created to store the data from production and also parts processed on the robot cell.

Querry to create Production table: 
```text
create table inno.production (ref_id TEXT primary key, assy_line text, prt_cntr_good int, prt_cntr_bad int, mode_tme_auto int, mode_tme_man int, mode_tme_init int, mode_tme_chg int, mode_tme_stop int, dtime timestamp);
```

Querry to create Parts Table:
```text
create table inno.parts (part_id TEXT primary key, ref_id text, prt_status text, prt_cyc_tme INT, weld_len int, weld_tme int, plsh_len int, plsh_tme int, dtime timestamp);
```

## Usage

To access the interface of CrateDB and see if all is running try the following:
```text
http://localhost:4200/
```

## License
The Inno West Rose-AP components are licensed under [Apache 2.0](/LICENSE) © 2022 Inno Robotics S.R.L.
