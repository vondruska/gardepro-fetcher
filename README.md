# gardepro-fetcher
I have a [GardePro E9P trailcam](https://gardepro.com/products/gardepro-wifi-trail-camera-e9p-with-rechargeable-battery) on my property to keep an eye on wildlife. When no device is connected, the camera is always advertising via Bluetooth LE. Using the GardePro mobile app, it will scan for Bluetooth devices, find the camera, connect to the camera via Bluetooth, send a command that enables wifi, disconnect from Bluetooth, connect to the newly visible wifi network.

I want to be able to pull images off the camera in an automated fashion. This repo will house a script/application to accomplish this. At this moment, it's research documentation. 

## Connection workflow
### Enabling wifi

Using [Android Bluetooth snooping](https://source.android.com/docs/core/connect/bluetooth/verifying_debugging#debugging-with-logs), I was able to find a couple packets that looked interesting. Particularly, "Send write command, Handle: 0x001e" with the value of `AT+WAKEPULSE=10\r\n`.

```
sudo gatttool -b {BT MAC address} --char-write-req -a 0x001e -n 41542b57414b4550554c53453d31300d0a
``` 

`0x001e` is the Bluetooth characteristic to enable wifi. `41542b57414b4550554c53453d31300d0a` is the hex representation of `AT+WAKEPULSE=10\r\n`. The line ending is critical for the wifi to spin up.

When successful, the device wifi network appears right away. `gatttool` might fail with `connect to {BT MAC address}: Function not implemented (38)`. Retry the command. The second or third attempt has worked 100% of the time.

### Connecting to the wifi
The wifi network uses WPA2. The password is '1234567890'.

The DHCP server typically gives the client the IP address 192.168.8.30. The default gateway is the camera itself at 192.168.8.1.

nmap:
```
$ nmap 192.168.8.1

PORT     STATE SERVICE
554/tcp  open  rtsp
8080/tcp open  http-proxy
```

### Staying connected
After a certain amount of idle time, the camera will disconnect the client and shutdown the wifi network. Presumably to save the battery. A HTTP GET request to `/cmd/standby/reset` must be sent on an interval to keep the camera from shutting down the network

```
$ curl -v http://192.168.8.1:8080/cmd/standby/reset
*   Trying 192.168.8.1:8080...
* Connected to 192.168.8.1 (192.168.8.1) port 8080
> GET /cmd/standby/reset HTTP/1.1
> Host: 192.168.8.1:8080
> User-Agent: curl/8.5.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Wed Feb  5 22:51:26 2025
< Connection: keep-alive
< Content-Type: application/json
< Content-Length: 14
<
{
        "code": 0
}
```

### Using the API
Discovered endpoints:

| Endpoint | Purpose  |
|---|---|
|`GET /cmd/getSetting`   | Gets current state of camera settings  |
|`GET /cmd/info/1`   |Gets device information (brand, product and version number)   | 
|`GET /cmd/info/2`  |Gets sensor and power info (temperature, voltage, vol_value, ext_power)    | 
|`GET /cmd/info/3`   |Gets SD storage information (total space, used, photos/video count)   |
|`GET /cmd/info/4`  |Gets time (clock, timezone)|
|`GET /cmd/info/5` |Gets detailed version strings|
|`GET /cmd/standby/now`|Put camera to sleep |
|`GET /cmd/standby/reset` |Keep-alive to prevent camera from shuttung down wifi network (see above) |
|`GET /cmd/getSetting` |Get camera settings |
|`GET /cmd/getParaSetting` |Get valid options for settings |
|`POST /cmd/setSetting` | `{"data": {"key": val}}` |
|`POST /cmd/setGmtClock` | `{"data": "yyyy-MM-dd HH:mm:ss"}` |
|`POST /cmd/setGmtClock2` | `{"data": "yyyy-MM-dd HH:mm:ss"}` |
|`GET /cmd/reboot` |Reboots |
|`GET /cmd/resetFact` |Factory reset |
|`GET /cmd/format/start` |Format SD card |
|`GET /cmd/format/result` |Result of formatting |
|`GET /cmd/delete/{id}/{JPG\|MP4}` |Delete image or video |
|`GET /list/detail/{type}/{startId}/{count}` |List files  |
|`GET /file/{id}/{JPG\|MP4}` |Download full-resolution file |
|`GET /thumb/{id}/{JPG\|MP4}` |Download thumbnail |
|`POST /media/pic/take` |Remote picture capture |
|`POST /media/pic/result` |Result of picture capture |
|`POST /media/video/start` |Remote video start |
|`POST /media/video/stop` |Remote video stop |
|`GET /media/getIrStatus` |Get IR Status |
|`POST /media/setDayNightMode` | `{"data": {"DayNightMode": val}}` |Set day / night mode |

