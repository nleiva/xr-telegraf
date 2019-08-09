# IOS XR Telegraf testing

Getting started with Telegraf and IOS XR Streaming Telemetry.

## Installing Telegraf

- Install Go

```bash
wget https://dl.google.com/go/go1.12.7.linux-amd64.tar.gz
sudo rm -rf /usr/lib/go
sudo tar -C /usr/local -xzf go1.12.7.linux-amd64.tar.gz
export GOROOT=/usr/local/go
export PATH=$PATH:$GOROOT/bin
```

Validate with `go version`.

- Install Dep

```bash
go get -d -u github.com/golang/dep 
cd $(go env GOPATH)/src/github.com/golang/dep
DEP_LATEST=$(git describe --abbrev=0 --tags) 
git checkout $DEP_LATEST
go install -ldflags="-X main.version=$DEP_LATEST" ./cmd/dep 
git checkout master
```

- Build Telegraf

```bash
dep ensure -vendor-only
export GO111MODULE=on
export GOPROXY=https://proxy.golang.org
go build -mod=vendor ./cmd/telegraf
```

Config: [telegraf.conf](telegraf.conf).

## Run InfluxDB

- Run it on a Docker Container

```bash
docker run -p 8086:8086 \
    -v $PWD:/var/lib/influxdb \
    influxdb
```

Modify `$PWD` to the directory where you want to store data associated with the InfluxDB container.

- Create a DB

```bash
export MDT_DURATION=168
export MDT_SHARD=6
MDT_DB=`echo q=CREATE DATABASE mdt_db WITH DURATION ${MDT_DURATION}h SHARD DURATION ${MDT_SHARD}h`
curl -s -XPOST http://localhost:8086/query --data-urlencode "$MDT_DB"
```

Validate with: `curl -s -XPOST http://localhost:8086/query --data-urlencode "q=show databases"`.

## Run Grafana

```bash
docker run \
  -d \
  -p 3000:3000 \
  --name=grafana \
  -e "GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-simple-json-datasource,briangann-gauge-panel" \
  -e "GF_SECURITY_ADMIN_PASSWORD=secret" \
  grafana/grafana
```

Grafana is now at: `http://<host ip>:3000` (admin/secret). Need to add InfluxDB as a data source.

## Configure the router

```bash
grpc
 port 57344
 address-family ipv4
!
```

```bash
telemetry model-driven
 sensor-group Basics
  sensor-path Cisco-IOS-XR-shellutil-oper:system-time/uptime
  sensor-path Cisco-IOS-XR-mpls-te-oper:mpls-te/tunnels/summary
  sensor-path Cisco-IOS-XR-pfi-im-cmd-oper:interfaces/interface-summary
  sensor-path Cisco-IOS-XR-wdsysmon-fd-oper:system-monitoring/cpu-utilization
  sensor-path Cisco-IOS-XR-nto-misc-oper:memory-summary/nodes/node/summary
  sensor-path Cisco-IOS-XR-ip-rsvp-oper:rsvp/interface-briefs/interface-brief
  sensor-path Cisco-IOS-XR-clns-isis-oper:isis/instances/instance/statistics-global
  sensor-path Cisco-IOS-XR-ip-rsvp-oper:rsvp/counters/interface-messages/interface-message
  sensor-path Cisco-IOS-XR-controller-optics-oper:optics-oper/optics-ports/optics-port/optics-info
  sensor-path Cisco-IOS-XR-clns-isis-oper:isis/instances/instance/levels/level/adjacencies/adjacency
  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
  sensor-path Cisco-IOS-XR-ipv4-bgp-oper:bgp/instances/instance/instance-active/default-vrf/process-info
  sensor-path Cisco-IOS-XR-ip-rib-ipv4-oper:rib/vrfs/vrf/afs/af/safs/saf/ip-rib-route-table-names/ip-rib-route-table-name/protocol/isis/as/information
  sensor-path Cisco-IOS-XR-ip-rib-ipv6-oper:ipv6-rib/vrfs/vrf/afs/af/safs/saf/ip-rib-route-table-names/ip-rib-route-table-name/protocol/isis/as/information
 !
 subscription Dashboard
  sensor-group-id Basics sample-interval 30000
 !
```

## Run Telegraf

Execute this: `./telegraf --config telegraf.conf`.

## Create Dashboard

Use [dashboard.json](dashboard.json).

### Querying the DB for troubleshooting

```json
$ curl -G 'http://localhost:8086/query?pretty=true' --data-urlencode "db=mdt_db" --data-urlencode "epoch=ms"  --data-urlencode "q=SELECT max(\"total-cpu-one-minute\") FROM \"Cisco-IOS-XR-wdsysmon-fd-oper:system-monitoring/cpu-utilization\""
{
    "results": [
        {
            "statement_id": 0,
            "series": [
                {
                    "name": "Cisco-IOS-XR-wdsysmon-fd-oper:system-monitoring/cpu-utilization",
                    "columns": [
                        "time",
                        "max"
                    ],
                    "values": [
                        [
                            1565292318129,
                            2
                        ]
                    ]
                }
            ]
        }
    ]
}
```

```bash
$ curl -G 'http://localhost:8086/query?pretty=true' --data-urlencode "db=mdt_db" --data-urlencode "epoch=ms"  --data-urlencode "q=SELECT \"adjacency-state\" FROM \"Cisco-IOS-XR-clns-isis-oper:isis/instances/instance/levels/level/adjacencies/adjacency\""
{
    "results": [
        {
            "statement_id": 0,
            "series": [
                {
                    "name": "Cisco-IOS-XR-clns-isis-oper:isis/instances/instance/levels/level/adjacencies/adjacency",
                    "columns": [
                        "time",
                        "adjacency-state"
                    ],
                    "values": [
                        [
                            1565292322899,
                            "isis-adj-up-state"
                        ],
                        [
                            1565292322899,
                            "isis-adj-up-state"
                        ],
                        [
                            1565292357571,
                            "isis-adj-up-state"
                        ],
                        ...
```

## Links

- [IOS XR Plugin](https://github.com/ios-xr/telegraf-plugin)
- [Cisco GNMI Telemetry](https://docs.influxdata.com/telegraf/v1.11/plugins/plugin-list/#cisco_telemetry_gnmi)