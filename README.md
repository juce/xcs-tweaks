# Misc tweaks for Xcode Server


## xcs-devices-patch.diff

There is a problem that appeared in Xcode Server 8.3, and is still present in Xcode 8.3.1 and 8.3.2.
beam.smp and node processes get stuck using a lot of CPU, which makes Xcode Server unresponsive, and
has a cascading effect of failing test, failing integrations, missing devices and simulators.

The issue has been reported to Apple multiple times, but so far there is no official fix.

- https://forums.developer.apple.com/message/225059
- https://forums.developer.apple.com/message/222707
- https://forums.developer.apple.com/message/221457

Thus, this workaround.


## Apparent problem

There is an execution path in Xcode API (which is hit a lot of times, apparently), where
the information about installed simulators and devices is retrieved and filtered. The very first time,
when Xcode Server is started, the query is sent to CouchDB, and then the response is cached in Redis -
to speed up subsequent requests for it. Problem is, this read-from-cache operation is extremely slow.

I can't really say why... but it's pretty simple to check and time it.
On my system:
- a CouchDB query takes about 35 seconds
- reading that cached info from Redis takes ~80 seconds!

This logic is in:
/Library/Developer/XcodeServer/CurrentXcodeSymlink/Contents/Developer/usr/share/xcs/xcsd/classes/deviceClass.js
(the .list method)


## How the fix works

So how to fix the slow cache access?
If we eliminate cache, we get a good speed improvement, but it is still not fast enough.
What we can do is a different kind of cache: file system

See the diff file for exact changes, but basically the idea is very simple:
Instead of caching the devices JSON info in Redis, we store it in a file on disk. Reading that file
apparently takes a lot less time then retrieving it from Redis. On my system, the file is ~28 MB in size,
and the difference between two types of access is huge:
- read from Redis: over 1 minute
- read from file on disk: 0.5 sec

The file written is:

    /Library/Developer/XcodeServer/Logs/xcs_devices.json

(I suspect that Redis is not really to blame here, but rather the way it is used. However i haven't investigated
further to be able to really pinpoint the root cause. This is a work-around after all ;-) )


## How to apply the patch:

Execute the following commands to apply the patch:

    cd /Library/Developer/XcodeServer/CurrentXcodeSymlink/Contents/Developer/usr/share/xcs/xcsd/classes
    sudo patch deviceClass.js /path/to/xcs-tweaks/xcs-devices-patch.diff

Then, either restart Xcode Server via the Server app, or you can just kill the node processes and they will automatically re-spawn themselves:

    sudo pkill -f node


## Check if things are good now:

Run this command a few times:

    sudo xcrun xcscontrol --list-simulators

It will fail the first couple of times (will say 0 simulators found), because it makes a call to Xcode API and times out. But after about 30 seconds or so, the xcs\_devices.json file will be created in /Library/Developer/XcodeServer/Logs/ directory, and once it appears, the next attempt to list the simulators should succeed now and be pretty fast too.


## Updating your simluators

If later on, you install another simulator, you would want to refresh the xcs\_devices.json file.
To do that simply remove it, and then try to list the simulators again a few times, as described in the previous section.

