---
title: "OpenWrt Speedtest"
excerpt: How fast and consistent is your internet speed?
image: 
  path: /images/speedtest_graph.jpg
  thumbnail: /images/openwrt_speedtest.jpg
  caption: ""
---

I like how the [OpenWrt Traffic Usage Monitor](https://openwrt.org/packages/pkgdata/nlbwmon) can show me how much data each device on my networked has dowloaded/uploaded in a given time.

I also wanted a way to see if how fast my internet really is. I searched around but couldn't really find anything ready to use, so i spent some time building it myself.

The plan is simple: 
1. run a speedtest every hour 
2. store the result and visualize it as a graph

Here's what I ended up doing to achieve this

![graph](images/speedtest_graph.png)

## Running the Speed test 

ookla provides a [cli version](https://www.speedtest.net/apps/cli) of their speedtest. Their tool allows:
* Set up automated scripts to collect connection performance data, including trends over time
* View test results via CSV, JSONL or JSON

I downloaded the binary for my router and created a cronjob that runs every hour and stores the data in a json file

`0 * * * * speedtest --accept-license --accept-gdpr --f json-pretty > speedtest_output.json`

## Collecting data

Luci offers some facilities for visualizing data. One of them is [luci-app-statistics](https://openwrt.org/docs/guide-user/luci/luci_app_statistics), a [collectd](https://collectd.org/) and [rrdtool](https://oss.oetiker.ch/rrdtool/) based statistics tool.

It was a lot of fun reading about RRD (Round Robin Database) and admire the engineering and thought that went into these systems to allow them to function in such limited environments. 

After understanding a bit more about how collectd and rrdtool worked I found collectd [exec plugin](https://collectd.org/wiki/index.php/Plugin:Exec) to be the most suitable and started following the Logging custom statistics [tutorial](https://openwrt.org/docs/guide-user/perf_and_log/statistic.custom)

I created a lua script (based on [this gist](https://gist.github.com/janaz/edc26e298e00d8bc172a78eff4bea770)) that is called by collectd every N seconds and reads the data from the json file created by the cronjob.

```lua
#!/usr/bin/lua

local cjson = require "cjson"

function get_contents(filename)
  local f = io.open(filename, "rb")
  local contents = ""
  if f then
    contents = f:read "*a"
    f:close()
  end
  return contents
end

local hostname = os.getenv("COLLECTD_HOSTNAME") or "localhost"
local interval = os.getenv("COLLECTD_INTERVAL") or 30

local json = cjson.decode(get_contents('speedtest_output.json'))
local download = json.download.bandwidth/125000
local upload = json.upload.bandwidth/125000

print(string.format("PUTVAL %s/exec-speedtest/gauge-download interval=%s N:%s", hostname, interval, download))
print(string.format("PUTVAL %s/exec-speedtest/gauge-upload interval=%s N:%s", hostname, interval, upload))
```

Regarding the magic number [125000](https://community.openhab.org/t/speedtest-cli-by-ookla-internet-up-downlink-measurement-integration/94447)
> While the human-readable output shows the bandwidth as Mbps the JSON output just shows the bandwidth in bits. This output can´t be changed and we need to divide the result by 125000 to get the bandwidth in Mbps.

Getting the script to work was a nightmare. It was impossible to debug and it took some time to get the syntax of the [PUTVAL](https://collectd.org/wiki/index.php/Plain_text_protocol#PUTVAL) right.
Once I finally got it working a folder showed up in `/tmp/rrd/router/exec-speedtest` with the two files inside, `gauge-download.rrd` and `gauge-upload.rrd`

## Rendering the graph

With the data in the Round Robin Database it was time to add the graph to luci. In the tutorial I followed there was a snippet to add a new tab and render a graph so I tweaked it to render the speed test graph

```js
'use strict';

return L.Class.extend({
    title: _('Exec'),

    rrdargs: function(graph, host, plugin, plugin_instance, dtype) {
        if (plugin_instance == 'speedtest') {
            return {
                title: "%H: Internet Speed",
                vlabel: "Mbit/sec",
                number_format: "%5.3lf",
                y_min: 1,
                data: {
                    types: ["gauge"],
                    sources: {
                        speedtest: ["gauge-download", "gauge-upload"]
                    },
                    options: {
                        gauge_download: {
                            color: "ff0000",
                            title: "Download",
                            overlay: true,
                            noarea: false,
                            total: false
                        },
                        gauge_upload: {
                            color: "0000ff",
                            title: "Upload",
                            overlay: true,
                            noarea: false,
                            total: false
                        }
                    }
                }
            };
        }
    }
});
```

There is a [mapping ](https://github.com/openwrt/luci/blob/master/applications/luci-app-statistics/htdocs/luci-static/resources/statistics/rrdtool.js) between the options in this file and the [RRDtool - rrdgraph](https://oss.oetiker.ch/rrdtool/doc/rrdgraph.en.html) configurations.

This was helpful to know what options were available like `y_min`.
