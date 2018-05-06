# Favoring one group of torrents over 2 other groups

This is one of the most frequently asked questions regarding to the usage of `rTorrent`. It turned out that its realization is pretty simple: let's adjust upload rate dynamically for each group and other attributes of them.

This feature is so powerful that it will change the way you used to work with `rTorrent`.

**Contents**

- [Scenario](#scenario)
- [Theory](#theory)
- [Realization](#realization)
  - [Sample config](#sample-config)
  - [Extra methods](#extra-methods)
  - [How it works](#how-it-works)
    - [Getting new uprate limit for slowup or tardyup throttle](#getting-new-uprate-limit-for-slowup-or-tardyup-throttle)
    - [Getting new global downrate limit](#getting-new-global-downrate-limit)
    - [Getting number of new upload slots for slowup or tardyup choke groups](#getting-number-of-new-upload-slots-for-slowup-or-tardyup-choke-groups)

## Scenario

There are sites where seeding data back is pretty hard while it's easy for others, not to mention public torrents that can easily eat all of the upload slots and bandwidth. So we want to distinguish between 3 groups, a special group, the rest of private downloads and public downloads:
- special group: we want to favor this private group in every possible way
  - this doesn't need any additional throttle group, it uses the global one
- `slowup` group: rest of the private downloads
  - it's not so important to provide the best circumstances for them, but we still want to seed them as long as we can
- `tardyup` group: public downloads if they're throttled (e.g. by moving into `fullseed` they can be special)
  - it's not important at all to provide the best circumstances for them, but we still want to seed them as long as we can

Applying this to downloading is easier (we have to set `d.priority` accordingly and it will take care about everything, apart from limiting the available download slots for a group) but not to seeding.


## Theory

Let's take a look what we need for this:
- define `throttle.up` group named `slowup` and set an initial value for it that will contain all the unimportant private  torrents (torrents that are assigned to this group)
- define `throttle.up` group named `tardyup` and set an initial value for it that will contain all the throttled public  torrents
- torrents that don't belong to either of the throttle groups belong to the special group (using the global throttle)
- set `d.priority` to low for the `slowup` and `tardyup` throttle groups that will take care of the downloading part
  - leaving `d.priority` set to normal for special group, so any download can be set to high manually if it's required
- lower the default `min_peers`, `max_peers` and `max_uploads` values for `slowup` and `tardyup` throttle groups (note: these settings aren't saved along the session hence we have to modify them on the fly, if necessary)
- add config entries to be able to specify limits for each group
  - adjust the value for each throttle groups in `.rtorrent-config.rc`
- setup 2 watch directories to automate this assignment between torrents and groups
  - we don't need a 3rd one: public torrents are handled automatically
- adjust the upload rate of `slowup` and `tardyup` throttle groups on the fly, depending on the current upload rate of the special group and the current global upload rate
- use 6 [choke groups](https://github.com/rakshasa/rtorrent/wiki/Choke-Groups) to limit the number of upload slots per group on the fly:
  - `default_leech` and `default_seed` for special group
  - `slowup_leech` and `slowup_seed` for other private downloads
  - `tardyup_leech` and `tardyup_seed` for public downloads
  - add config entries to be able to specify limits for each group
    - adjust the value for each choke groups in `.rtorrent-config.rc`
  - use the above naming convention to auto-assign the different choke groups to the proper download with the corresponding throttle group all the time!

Since all we want to do is seeding (unfortunately we have to download before it :) ), we can also deal with the common problem of asynchronous connections (e.g. ADSL) where upload speed can be slowed down if download speed is close to maximum bandwidth:
- adjusts the global download rate dynamically to always leave enough bandwidth to the upload rate of special group


## Realization

**Note**: this feature requires `rTorrent-PS-CH 1.5.0-0.9.7` or newer!

### Sample config

Let's see the related settings at once first. These settings are used with 74/20 Mbps connection.

```ini
# Global upload and download rate in KiB, `0` for unlimited (`download_rate`, `upload_rate`)
throttle.global_down.max_rate.set_kb = 10000
throttle.global_up.max_rate.set_kb   = 2500

# Maximum number of simultaneous downloads and uploads slots (global slots!) (`max_downloads_global`, `max_uploads_global`)
throttle.max_downloads.global.set = 300
throttle.max_uploads.global.set   = 300

# Maximum and minimum number of peers to connect to per torrent while downloading (`min_peers`, `max_peers`) Default: `100` and `200` respectively
throttle.min_peers.normal.set = 99
throttle.max_peers.normal.set = 100

# Same as above but for seeding completed torrents (seeds per torrent), `-1` for same as downloading (`min_peers_seed`, `max_peers_seed`) Default: `-1` for both
throttle.min_peers.seed.set = -1
throttle.max_peers.seed.set = -1

# Maximum number of simultaneous downloads and uploads slots per torrent (`max_uploads`) Default: `50` for both
throttle.max_downloads.set = 50
throttle.max_uploads.set = 50

# Throttles rates for (groups of) downloads or IP ranges. (throttle_up) These throttles borrow bandwidth from
#  the global throttle and thus are limited by it too.
# You can assign throttles to a stopped (!) download with Ctrl-T. The `NULL` throttle is a special unlimited
#  throttle that bypasses the global throttle.
# Limits the upload rate to 600 kb/s for the `slowup` throttle group. We also use this property to distinguish
#  `special` and `rest` group. This value will be auto-adjusted by a helper script in Favoring section.
throttle.up = slowup,600
throttle.up = tardyup,300

# initial `slowup` group values for the previous similar 3 settings that will be overridden by per torrent settings
#  Favouring section. `cfg.slowup.d.peers_min` ~ `throttle.min_peers.normal` ,
#  `cfg.slowup.d.peers_max` ~ `throttle.max_peers.normal` , `cfg.slowup.d.uploads_max` ~ `throttle.max_uploads`
method.insert = cfg.slowup.d.peers_min,     value,    29
method.insert = cfg.slowup.d.peers_max,     value,    50
method.insert = cfg.slowup.d.uploads_max,   value,    15


## Variables for getting upload rate limit for throttle groups and max number of upload/download slots for choke groups
# Max, Min value of uprate limit throttle in KB and Max number of upload/download slots during leeching and seeding for slowup throttle group
method.insert = cfg.slowup.uprate.max,      value,  1600
method.insert = cfg.slowup.uprate.min,      value,    75
method.insert = cfg.slowup.slots.leech.max, value,   200
method.insert = cfg.slowup.slots.seed.max,  value,   125
# Max, Min value of uprate limit throttle in KB and Max number of upload/download slots during leeching and seeding for tardyup throttle group
method.insert = cfg.tardyup.uprate.max,     value,  1200
method.insert = cfg.tardyup.uprate.min,     value,    25
method.insert = cfg.tardyup.slots.leech.max,value,   150
method.insert = cfg.tardyup.slots.seed.max, value,    75

## Variables for getting global downrate limit
# Max, Min value of global downrate in KB
method.insert = cfg.global.downrate.max,    value,  9999
method.insert = cfg.global.downrate.min,    value,  8000
# Threshold values for global and special group uprate in KB
method.insert = cfg.global.upall.threshold, value,  1600
method.insert = cfg.global.upmain.threshold,value,  1100

# Min value of uprate per upload slot (unchoked peers) in KB
method.insert = cfg.global.slot.uprate.min, value,     5


# Setting up choke groups that restricts the number of unchoked peers in a group
# Modify default choke groups for specail group
choke_group.up.heuristics.set = default_leech,upload_leech_experimental
choke_group.tracker.mode.set  = default_leech,aggressive
choke_group.tracker.mode.set  = default_seed,aggressive
# Set up choke groups for slowup group
choke_group.insert = slowup_leech
choke_group.insert = slowup_seed
choke_group.up.heuristics.set = slowup_leech,upload_leech
choke_group.up.heuristics.set = slowup_seed,upload_seed
choke_group.down.max.set = slowup_leech,(cat,(cfg.slowup.slots.leech.max))
choke_group.up.max.set   = slowup_leech,(cat,(cfg.slowup.slots.leech.max))
choke_group.up.max.set   = slowup_seed,(cat,(cfg.slowup.slots.seed.max))
# Set up choke groups for tardyup group
choke_group.insert = tardyup_leech
choke_group.insert = tardyup_seed
choke_group.up.heuristics.set = tardyup_leech,upload_leech
choke_group.up.heuristics.set = tardyup_seed,upload_seed
choke_group.down.max.set = tardyup_leech,(cat,(cfg.tardyup.slots.leech.max))
choke_group.up.max.set   = tardyup_leech,(cat,(cfg.tardyup.slots.leech.max))
choke_group.up.max.set   = tardyup_seed,(cat,(cfg.tardyup.slots.seed.max))


# Watch directories for new torrents (meta files). Also specifying the final directories (data_dir and meta_dir) for them, whether it belongs to special group, whether its data is deletable (in this order) by setting:
#   - normal priority for the special group ; - low priority and slowup throttle for the 2nd group (rest of the torrents) ; - unsafe_data custom field for those we want to delete their data upon removal
#   'd.attribs.set' command receives 3 arguments: dirname,specialgroup,unsafe_data
branch = ((cfg.inotify.use)), ((directory.watch.added, (cat,(cfg.dir.meta_downl),rotating/), load.start,  "d.attribs.set=rotating,1,1"))
branch = ((cfg.inotify.use)), ((directory.watch.added, (cat,(cfg.dir.meta_downl),unsafe/),   load.start,  "d.attribs.set=unsafe,,1"))


##### begin: Favoring special group of torrents over the rest #####

# helper method: Sets choke group to one of the default ones if there's no throttle or throttle is special one (NULL) otherwise sets it to one of the throttle name ones
method.insert = d.modify_choke_group, simple|private, "branch=((and,((d.throttle_name)),((not,((equal,((d.throttle_name)),((cat,NULL)))))))),((d.group.set,(cat,(d.throttle_name),_,(d.connection_current)))),((d.group.set,(cat,default_,(d.connection_current))))"
# helper method: Modifies the peers_min, peers_max, max_uploads values of a torrent for both downloading and uploading
method.insert = d.modify_slots, simple|private, "d.peers_min.set=(argument.0); d.peers_max.set=(argument.1); d.uploads_max.set=(argument.2)"
# Modifies the above properties for a torrent based on which group it belongs to
method.insert = d.modify_slots_both, simple, "branch=((not,(d.throttle_name))),((d.modify_slots,(throttle.min_peers.normal),(throttle.max_peers.normal),(throttle.max_uploads))),((d.modify_slots,(cfg.slowup.d.peers_min),(cfg.slowup.d.peers_max),(cfg.slowup.d.uploads_max)))"
# Modify both group values when a torrent is resumed (even after hashchecking or after rtorrent is restarted)
method.set_key = event.download.resumed, modify_slots_resumed_both, "d.modify_slots_both=; d.modify_choke_group="
method.set_key = event.download.finished, modify_finished_choke_group, "d.modify_choke_group="
method.set_key = event.download.partially_restarted, modify_partially_restarted_choke_group, "d.modify_choke_group="

# Gets current uprate in KB for special group
method.insert = get_uprate_main, simple, "math.max=0,(math.div,(math.sub,(throttle.global_up.rate),(throttle.up.rate,slowup),(throttle.up.rate,tardyup)),1024)"
# helper methods: get new uprate limit in KB (based on special group uprate) for slowup and tardyup throttle groups
method.insert = get_uprate_slowup,  simple|private, "math.min=(cfg.slowup.uprate.max),(math.max,(cfg.slowup.uprate.min),(math.sub,(cfg.slowup.uprate.max),(get_uprate_main),(math.div,(throttle.up.rate,tardyup),1024)))"
method.insert = get_uprate_tardyup, simple|private, "math.min=(cfg.tardyup.uprate.max),(math.max,(cfg.tardyup.uprate.min),(math.sub,(cfg.tardyup.uprate.max),(get_uprate_main),(math.div,(throttle.up.rate,slowup),1024)))"
# Set new uprate limit for slowup and tardyup throttle groups in every 20 seconds
schedule2     = adjust_throttle_slowup,  14, 20, "throttle.up=slowup,(cat,(get_uprate_slowup))"
schedule2     = adjust_throttle_tardyup, 15, 20, "throttle.up=tardyup,(cat,(get_uprate_tardyup))"

# helper method: get new global downrate limit in KB (based on special group uprate and given threshold values)
method.insert = get_downrate_global, simple|private, "branch=(and,((greater,((math.div,(throttle.global_up.rate),1024)),((cfg.global.upall.threshold)))),((greater,((get_uprate_main)),((cfg.global.upmain.threshold))))),(cfg.global.downrate.min),(cfg.global.downrate.max)"
# Set new global downrate limit in every 60 seconds
schedule2     = adjust_throttle_global_down_max_rate, 54, 60, "throttle.global_down.max_rate.set_kb=(get_downrate_global)"

# helper methods: get new upload slots limit for slowup_seed and tardyup_seed choke groups (based on their current uprate and the given cfg values)
method.insert = get_slots_slowup,  simple|private, "math.min=(cfg.slowup.slots.seed.max),(math.max,(math.div,(cfg.slowup.uprate.min),(cfg.global.slot.uprate.min)),(math.div,(throttle.up.rate,slowup),1024,(cfg.global.slot.uprate.min)))"
method.insert = get_slots_tardyup, simple|private, "math.min=(cfg.tardyup.slots.seed.max),(math.max,(math.div,(cfg.tardyup.uprate.min),(cfg.global.slot.uprate.min)),(math.div,(throttle.up.rate,tardyup),1024,(cfg.global.slot.uprate.min)))"
# Set new upload slots (unchoked peers) limit for slowup_seed and tardyup_seed choke groups in every 100 seconds
schedule2     = adjust_slots_slowup,  47, 100,"choke_group.up.max.set=slowup_seed,(cat,(get_slots_slowup))"
schedule2     = adjust_slots_tardyup, 48, 100,"choke_group.up.max.set=tardyup_seed,(cat,(get_slots_tardyup))"

##### end: Favoring special group of torrents over the rest #####

### begin: Other UI enhancements ###
# UI/STATUS: Display additional throttle up info in status bar (it needs rT-PS-CH)
branch=pyro.extended=,"branch=pyro.extended.ch=,\"ui.status.throttle.up.set = slowup,tardyup\""
### end: Other UI enhancements ###
```


### Extra methods

Since we can do basic arithmetic operations in `rTorrent-PS-CH 1.5.0-0.9.7` or newer, we don't need the previously used external script file.

They do 3 things:
- gets new uprate limit for `slowup` and `tardyup` throttle in every 20 seconds (based upon the configured variables)
  - specify the top of the cap (`cfg.slowup.uprate.max`, `cfg.tardyup.uprate.max`: highest allowable value in KiB, e.g. `1600` and `1200`)
  - specify the bottom of the cap (`cfg.slowup.uprate.min`, `cfg.tardyup.uprate.min`: lowest allowable value in KiB, e.g. `75` and `25`)
- gets new global downrate limit (based on special group uprate and the configured variables) in every 1 minute
  - specify the top of the cap (`cfg.global.downrate.max` in KiB, e.g. `9999`) for global downrate
  - specify the bottom of the cap (`cfg.global.downrate.min` in KiB, e.g. `8000`) for global downrate
  - specify the global uprate limit (`cfg.global.upall.threshold` in KiB, e.g. `1600`) that above this value it should lower global downrate
  - specify uprate limit for special group (`cfg.global.upmain.threshold` in KiB, e.g. `1100`) that above this value it should lower global downrate
- gets new upload slots limit for `slowup_seed` and `tardyup_seed` choke groups (based on their current uprate and the configured variables) in every 100 seconds
  - specify the top of the cap (`cfg.slowup.slots.seed.max`, `cfg.tardyup.slots.seed.max`: highest allowable value per group, e.g. `125` and `75`)

Also specify the max amount of upload/download slots for both groups during leeching (`cfg.slowup.slots.leech.max`, `cfg.tardyup.slots.leech.max`: highest allowable value per group, e.g. `200` and `150`), these values won't be changed.

You have to edit the variables in `.rtorrent-config.rc` according to your needs, it probably also needs some experimenting which values works best for your needs. These variables can be adjusted on-the-fly while `rTorrent` is running.


### How it works

- once you put a torrent into a watch directory all the necessary attributes are assigned to it
- `adjust_throttle_slowup` and `adjust_throttle_tardyup` runs in every 20 seconds
- `adjust_throttle_global_down_max_rate` runs in every 60 seconds
- `adjust_slots_slowup` and `adjust_slots_tardyup` runs in every 100 seconds

#### Getting new uprate limit for slowup or tardyup throttle

It doesn't use a complicated algorithm to always give back reliable result, e.g. for slowup throttle  `cfg.slowup.uprate.max - uprate_main -  throttle.up.rate,tardyup`, but it works pretty efficiently:
- checks the current uprate of the other throttle group and uprate of main (special) group (with the help of the global uprate) then it raises or lowers the throttle limit according to the uprate of the main (special) and the other throttle group
- you should leave a considerable amount of gap around `20%` (`500 KiB`, in this case) between the top of the cap (`cfg.slowup.uprate.max` , e.g. `1600`) and the max global upload rate (`throttle.global_up.max_rate.set_kb` : the global upload limit , e.g. `2500`) and the other top cap of the other throttle group (`cfg.tardyup.uprate.max`, e.g. `1200`) to be able to work efficiently: to leave bandwidth to the main (special) group between checks (within that 20 seconds interval)

#### Getting new global downrate limit

This one even less sophisticated then the above one, it only sets one of the 2 defined `cfg.global.downrate.max`, `cfg.global.downrate.min` values for global downrate:
- whether global upload rate is bigger than the specified global uprate threshold (`cfg.global.upall.threshold`) and main (special) upload rate is bigger than the specified main uprate threshold (`cfg.global.upmain.threshold`)

Remember, if you don't need this feature then comment out `adjust_throttle_global_down_max_rate` scheduling in `rTorrent` config.

#### Getting number of new upload slots for slowup or tardyup choke groups

This is really simple, it just checks for the current uprate of the given throttle group and divides it by the minimum rate a slot can have (`cfg.global.slot.uprate.min`, e.g. `5` KiB).

