# Dataset
 ROSE-AP automated planning

The main service contributed by the ROSE-AP is automated planning, to minimize the up-front automated robotic cell productivity 

## Contents

-   [Settings things up](#Setting)
-   [Persisting Time Series data](#timeseries)
-   [Config Cleanup](#cleanup)
-   [License](#license)

## Setting

Entity modeling: we set up a simplified model, having one site, one area, one workstation, and two units. Physically we have one robotic cell performing part welding and polishing. We represented this in the figure below.

<img width="1000" alt="Architecture" src="docs/ds_architecture.png">

> Note: all commands below use the curl software tool, and assume the Orion Context Broker is up and running, and available in the same (local) network.

To create the site:
```text
curl -iX POST 'http://{context-broker-IP}:1026/v2/entities/' \
  -H 'Content-Type: application/json' \
  -d '{
  
  "id": "urn:ngsiv2:i40Asset:Site:001", "type": "i40Asset", "i40AssetType": {"type": "string", "value": "Site"}
  
  }
  '
```

To create the area:
```text
curl -iX POST 'http://{context-broker-IP}:1026/v2/entities/' \
  -H 'Content-Type: application/json' \
  -d '{
  
  "id": "urn:ngsiv2:i40Asset:Area:001", "type": "i40Asset", "i40AssetType": {"type": "string", "value": "Area"}, "hasParentI40Asset": {"type": "string", "value": "urn:ngsiv2:i40Asset:Site:001"}
  
  }
  '
```

To create the workstation:
```text
curl -iX POST 'http://{context-broker-IP}:1026/v2/entities/' \
  -H 'Content-Type: application/json' \
  -d '{
  
  "id": "urn:ngsiv2:i40Asset:Workstation:001", "type": "i40Asset", "i40AssetType": {"type": "string", "value": "Workstation"}, "hasParentI40Asset": {"type": "string", "value": "urn:ngsiv2:i40Asset:Area:001"}
  
  }
  '
```

To create workstation attributes (here its state):
```text
curl -iX POST 'http://{context-broker-IP}:1026/v2/entities/urn:ngsiv2:i40Asset:Workstation:001/attrs' \
  -H 'Content-Type: application/json' \
  -d '{
  
  "i40AssetState": {"type": "string", "value": "STOPPED"}
  }
  '
```

To create the two units:
```text
curl -iX POST 'http://{context-broker-IP}:1026/v2/entities/' \
  -H 'Content-Type: application/json' \
  -d '{
  
  "id": "urn:ngsiv2:i40Asset:Unit:001a", "type": "i40Asset", "i40AssetType": {"type": "string", "value": "Unit"}, "hasParentI40Asset": {"type": "string", "value": "urn:ngsiv2:i40Asset:Workstation:001"}
  
  }
  '
```
and
```text
curl -iX POST 'http://{context-broker-IP}:1026/v2/entities/' \
  -H 'Content-Type: application/json' \
  -d '{
  
  "id": "urn:ngsiv2:i40Asset:Unit:002a", "type": "i40Asset", "i40AssetType": {"type": "string", "value": "Unit"}, "hasParentI40Asset": {"type": "string", "value": "urn:ngsiv2:i40Asset:Workstation:001"}
  
  }
  '
```

The outcome, as seen on the context broker machine:

<img width="1000" alt="Result" src="docs/outcome_1.png">

To manage workstation states, we used this command template:
```text
curl -iX PUT 'http://{context-broker-IP}:1026/v2/entities/urn:ngsiv2:i40Asset:Workstation:001/attrs/i40AssetState/value' \
  -H 'Content-Type: application/json' \
  -d '{
  
  "value": "STARTING"
  
  }
  '  
```
> Note: where {state} could be IDLE, STARTING, RUNNING, STANDBY, STOPPING, or ERROR.

## Timeseries

In our current setup, we used Quantumleap directly to persist TimeSeries data. To do this, our broker creates a subscription for the workstation to Quantumleap and then posts data directly. To create the subscription, we used this commands:

(here the CRATE view in browser: http://localhost:4200)
(here the subscriptions: http://localhost:1026/v2/subscriptions – access this to get the subscription IDs which you need below!!)
```text
curl -iX POST \
  --url 'http://{context-broker-IP}:1026/v2/subscriptions' \
  --header 'content-type: application/json' \
  --data '{
        "description": "WorkstationSubscription",
        "subject": {
            "entities": [
            {
                "idPattern": ".*",
                "type": "Workstation"
            }
            ],
            "condition": {
                "attrs": [
                "prt_cntr_good", "prt_cntr_bad", "ws_state", "mode_tme_auto", "mode_tme_man", "mode_tme_init", "mode_tme_chg", "mode_tme_stop", "ws_msg"
                ]
            }
        },
        "notification": {
            "http": {
                "url": "http://{context-broker-IP}:8668/v2/notify"
            },
            "attrs": [
            "prt_cntr_good", "prt_cntr_bad", "ws_state", "mode_tme_auto", "mode_tme_man", "mode_tme_init", "mode_tme_chg", "mode_tme_stop", "ws_msg"
            ],
            "metadata": ["dateCreated", "dateModified"]
        }
    }
'
```
The parameter meanings are as follows:
-	"prt_cntr_good": counter for good parts; if a part processing operations has a correct outcome, it will held value 1
-	"prt_cntr_bad": similar meaning, but for an outcome with a „non-compliant“ part
-	"ws_state": the new workstation state (we may discontinue to use this param)
-	"mode_tme_auto", "mode_tme_man", "mode_tme_init", "mode_tme_chg", "mode_tme_stop": time of the just completed operation, as follows: _auto, for producing a part in „auto“ mode, _man for performing in manual mode, _init for initialization time (from OFF to STANDBY), _chg for changing the program parameters, and _stop for stopping (going from STANDBY or IDLE to OFF)
-	"ws_msg": a supplementary text message to clarify the outcome of the completed operation

To exemplify, the broker uses then this command to insert into Quantumleap the information that the workstation is starting (from OFF it goes to STANDBY):
```text
curl http://{context-broker-IP}:8668/v2/notify -s -S -H 'Content-Type: application/json' -d @- <<EOF
{
    "subscriptionId": "{workstation-subscription-id}",
    "data": [
        {
            "id": "Workstation1",
            "prt_cntr_good":
               {
                 "value": "0",
                 "type": "Number"
               },
             "prt_cntr_bad":
               {
                 "value": "0",
                 "type": "Number"
               },
             "ws_state":
               {
                 "value": "STANDBY",
                 "type": "Text"
               },
             "mode_tme_auto":
               {
                 "value": "0",
                 "type": "Number"
               },
             "mode_tme_man":
               {
                 "value": "0",
                 "type": "Number"
               },
             "mode_tme_init":
               {
                 "value": "24",
                 "type": "Number"
               },
             "mode_tme_chg":
               {
                 "value": "0",
                 "type": "Number"
               },
             "mode_tme_stop":
               {
                 "value": "0",
                 "type": "Number"
               },

             "ws_msg":
               {
                 "value": "Workstation initialized ok, status: STANDBY",
                 "type": "Text"
               },
            "type": "Workstation"
        }
    ]
}
EOF
```

Here an example of what is persisted when an operation on a part is started:
```text
curl http://{context-broker-IP}:8668/v2/notify -s -S -H 'Content-Type: application/json' -d @- <<EOF
{
    "subscriptionId": "{workstation-subscription-id}",
    "data": [
        {
            "id": "Workstation1",
            "prt_cntr_good":
               {
                 "value": "0",
                 "type": "Number"
               },
             "prt_cntr_bad":
               {
                 "value": "0",
                 "type": "Number"
               },
             "ws_state":
               {
                 "value": "RUNNING",
                 "type": "Text"
               },
             "mode_tme_auto":
               {
                 "value": "0",
                 "type": "Number"
               },
             "mode_tme_man":
               {
                 "value": "0",
                 "type": "Number"
               },
             "mode_tme_init":
               {
                 "value": "0",
                 "type": "Number"
               },
             "mode_tme_chg":
               {
                 "value": "0",
                 "type": "Number"
               },
             "mode_tme_stop":
               {
                 "value": "0",
                 "type": "Number"
               },

             "ws_msg":
               {
                 "value": "Processing command – working on part; new state: RUNNING",
                 "type": "Text"
               },
            "type": "Workstation"
        }
    ]
}
EOF
```

Here an example of what is persisted when an operation on a part is finished (part is produced):
```text
curl http://{context-broker-IP}:8668/v2/notify -s -S -H 'Content-Type: application/json' -d @- <<EOF
{
    "subscriptionId": "{workstation-subscription-id}",
    "data": [
        {
            "id": "Workstation1",
            "prt_cntr_good":
               {
                 "value": "1",
                 "type": "Number"
               },
             "prt_cntr_bad":
               {
                 "value": "0",
                 "type": "Number"
               },
             "ws_state":
               {
                 "value": "STANDBY",
                 "type": "Text"
               },
             "mode_tme_auto":
               {
                 "value": "232",
                 "type": "Number"
               },
             "mode_tme_man":
               {
                 "value": "0",
                 "type": "Number"
               },
             "mode_tme_init":
               {
                 "value": "0",
                 "type": "Number"
               },
             "mode_tme_chg":
               {
                 "value": "0",
                 "type": "Number"
               },
             "mode_tme_stop":
               {
                 "value": "0",
                 "type": "Number"
               },
             "ws_msg":
               {
                 "value": "Part produced ok",
                 "type": "Text"
               },
            "type": "Workstation"
        }
    ]
}
EOF
```
Our broker creates also a „unit“ subscription to Quantumleap and then posts time series data related to parts (actually the operation performed on the part). To create the unit subscription, we used this command:

```text
curl -iX POST \
  --url 'http://{context-broker-IP}:1026/v2/subscriptions' \
  --header 'content-type: application/json' \
  --data '{
        "description": "UnitSubscription",
        "subject": {
            "entities": [
            {
                "idPattern": ".*",
                "type": "Unit"
            }
            ],
            "condition": {
                "attrs": [
                "part_id", "ref_id", "prt_status", "prt_cyc_tme", "weld_len", "weld_tme", "plsh_len", "plsh_tme", "prt_msg"
                ]
            }
        },
        "notification": {
            "http": {
                "url": "http://{context-broker-IP}:8668/v2/notify"
            },
            "attrs": [
            "part_id", "ref_id", "prt_status", "prt_cyc_tme", "weld_len", "weld_tme", "plsh_len", "plsh_tme", "prt_msg"
            ],
            "metadata": ["dateCreated", "dateModified"]
        }
    }
'
```

The attribute meaning is as follows:
-	"part_id": the part identifier
-	"ref_id": an internal reference id
-	"prt_status": if the part is compliant or not
-	"prt_cyc_tme": operation time
-	"weld_len": length (mm) of welding
-	"weld_tme": time (s) needed for the welding
-	"plsh_len": length (mm) of polishing
-	„plsh_tme" : time (s) needed for the polishing
-	"prt_msg": a customizable string message

Here is an example of persisting information on the outcome of processing part with id P2290001:
```text
curl http://{context-broker-IP}:8668/v2/notify -s -S -H 'Content-Type: application/json' -d @- <<EOF
{
    "subscriptionId": "{unit-subscription-id}",
    "data": [
        {
            "id": "Unit1",
            "part_id":
               {
                 "value": "P2290001",
                 "type": "Text"
               },
             "ref_id":
               {
                 "value": "R20001",
                 "type": "Text"
               },
             "prt_status":
               {
                 "value": "GOOD",
                 "type": "Text"
               },
             "prt_cyc_tme":
               {
                 "value": "232",
                 "type": "Number"
               },
             "weld_len":
               {
                 "value": "820",
                 "type": "Number"
               },
             "weld_tme":
               {
                 "value": "132",
                 "type": "Number"
               },
             "plsh_len":
               {
                 "value": "820",
                 "type": "Number"
               },
             "plsh_tme":
               {
                 "value": "100",
                 "type": "Number"
               },
             "prt_msg":
               {
                 "value": "Part produced ok",
                 "type": "Text"
               },
            "type": "Unit"
        }
    ]
}
EOF
```

Here is another example of processing part with id P2290002:
```text
curl http://{context-broker-IP}:8668/v2/notify -s -S -H 'Content-Type: application/json' -d @- <<EOF
{
    "subscriptionId": "{unit-subscription-id}",
    "data": [
        {
            "id": "Unit1",
            "part_id":
               {
                 "value": "P2290002",
                 "type": "Text"
               },
             "ref_id":
               {
                 "value": "R20001",
                 "type": "Text"
               },
             "prt_status":
               {
                 "value": "BAD",
                 "type": "Text"
               },
             "prt_cyc_tme":
               {
                 "value": "232",
                 "type": "Number"
               },
             "weld_len":
               {
                 "value": "820",
                 "type": "Number"
               },
             "weld_tme":
               {
                 "value": "132",
                 "type": "Number"
               },
             "plsh_len":
               {
                 "value": "820",
                 "type": "Number"
               },
             "plsh_tme":
               {
                 "value": "100",
                 "type": "Number"
               },
             "prt_msg":
               {
                 "value": "FAILURE!! producing part",
                 "type": "Text"
               },
            "type": "Unit"
        }
    ]
}
EOF
```

## Cleanup

To clean up, while testing, we used this command template:
```text
curl -iX DELETE 'http://{context-vm-ip}:1026/v2/entities/{asset-id}/'
```
```text
curl -iX DELETE \
  --url 'http://{context-vm-ip}:1026/v2/subscriptions/{subscription-id}'
```

## License
The Inno West Rose-AP components are licensed under [Apache 2.0](/LICENSE) © 2023 Inno Robotics S.R.L.
