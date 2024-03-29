# CO_2 senor on Raspberry Pi with MQTT and Home Assistant integration

I read from hacker news ([ref](https://news.ycombinator.com/item?id=34648021)) and related posts that a slightly high CO_2 (carbon dioxide) concentration may decrease some people's cognitive function, like, 1200ppm (parts per million) results in 15% of decrease or so. Since I am staying in my bedroom in the most of time and I don't have a good habbit of regular ventilation, I believe a CO_2 sensor and an alert system may help me get out of any gloomy states. I have a Raspberry Pi 1b at hand, so I can use it to read measurement from a CO_2 sensor. It turns out it takes much much more efforts to get the 11-years-old Raspberry Pi to connect to the WiFi. I put this failure part at the end of this post.

![pending picture final results]()

## Requirement and Design

I want to measure the CO_2 concentration of my environment and read the measurement and get notificaiton from my PC and mobile phones. Therefore my plan is to buy a CO_2 sensor, connect it to my Raspberry Pi, write a program to read from CO_2 sensor and send the data to my server, and display the measurement on my server via web. 

Since my Raspberry Pi is quite old I am going to write in C and use MQTT (Message Queuing Telemetry Transport) to communicate with my server. On the server side, I have a Home Assistant instance running, so it would be great if I can integrate them. Once Home Assistant can receive these data, I will be able to view them on my PC and mobile phone and also let prometheus to scrape it for alerting. So these will be the stack I choose:

- C with libmosquitto on Raspberry Pi
- Mosquitto MQTT Server
- Home Assistant
- Prometheus, Grafana and alert manager

## Connecting to PC via ethernet and sharing Internet from PC
The first thing to do is to get my Raspberry Pi connected to the Internet, after days of trying I finally gave up on WiFi adapter drivers. So now I connect a CAT6 ethernet cable to a PC with internet access via WiFi ([ref](http://linux-wiki.cn/wiki/zh-hans/%E5%85%B1%E4%BA%AB%E4%B8%8A%E7%BD%91)).

Install dhcpd:

```bash
pacman -S dhcp
```

Add static IP for ethernet adapter:

```bash
ip addr add 192.168.155.1/24 dev enp8s0
```

`enp8s0` is my ethernet interface connecting raspberry pi.

Backup dhcpd config:

```bash
cp /etc/dhcpd.conf /etc/dhcpd.conf.example
```

Add dhcp service configuration:

```conf
/etc/dhcpd.conf
---

...

option domain-name-servers 1.1.1.1, 1.0.0.1;
option subnet-mask 255.255.255.0;
option routers 192.168.155.1;
subnet 192.168.155.0 netmask 255.255.255.0 {
  range 192.168.155.100 192.168.155.150;
}

# avoid serving dhcp under wifi router
subnet 192.168.2.0 netmask 255.255.255.0 {
}
```

Start dhcpd service:

```bash
systemctl enable dhcpd4
systemctl start dhcpd4
```

The two devices shoud be able to ping each other now.

Share the Internet:

```bash
echo "1" > /proc/sys/net/ipv4/ip_forward
```

Add `net.ipv4.ip_forward=1` to `/etc/sysctl.conf` to make this change permanent

Enable NAT with `iptables`:

```bash
iptables -F
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -t nat -A POSTROUTING -o wlp4s0 -j MASQUERADE
```

Save the rules to make the change persistant (This may only work on Archlinux ([ref](https://wiki.archlinux.org/title/Iptables)))

```bash
iptables-save -f /etc/iptables/iptables.rules
```

Otherwise simply `iptables-restore /path/to/iptables.rules`.

`wlp4s0` is my wireless adapter with internet connection

The Pi shoud be able to access internet now.

## CO_2 sensors
There are many types of CO_2 sensors on the market, Some of them are inaccurate, some of them are expensive, and some of them are even fake. I noticed that the price of a decent consumer sensor chip is at least 25 EUR, therefore I guess the multifunctional air quality sensor below this price can't fullfill my requirement. CO_2 equivalent (CO_2eq) or Estimated CO_2 (eCO_2) sensors are also not considered because they are not reliable and inaccurate. 

I asked friends about different types of CO_2 sensors and finally decide to buy an SCD41 sensor from AliExpress. This sensor uses photoacoustic spectroscopy to measure CO_2 concentration and also measures temperature and relative humidity. 

![pending picture scd41]()

## Reading data from sensor

SCD4x uses I2C (Inter-Integrated Circuit) bus and it's address is `0x62`. 

First enable I2C on Raspberry Pi in `sudo raspi-config`, connect the sensor to corresponding pins (I'm using Pin 3, 4, 5, 6) ([ref](https://pinout.xyz/pinout/i2c)), restart Pi. 

![pending picture scd41 with pi]()

Then we can see the SCD41 sensor at address `0x62`:

```bash
ls /dev/i2c*
apt install i2c-tools
i2cdetect 0
i2cdump 0 0x62
```

However SCD4x doesn't work that way ([ref](https://cdn.sparkfun.com/assets/d/4/9/a/d/Sensirion_CO2_Sensors_SCD4x_Datasheet.pdf))

We can read write directly ([ref](https://www.kernel.org/doc/Documentation/i2c/dev-interface))

Include `linux/i2c-dev.h` to get definition of `I2C_SLAVE`, include`fcntl.h` to open file, include `unistd.h` to `read`, `write` and `sleep`, and include `stdio.h` to print message to stdout:

```c
#include <linux/i2c-dev.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
```

Open I2C device (mine is `/dev/i2c-0`, it can be elsewhere):

```c
int device = open("/dev/i2c-0", O_RDWR);
if (device < 0) {
  printf("file open error: %x\n", device);
  exit(1);
}
```

Specify SCD4x sensor address:

```c
__u8 addr = 0x62;

if (ioctl(device, I2C_SLAVE, addr) < 0) {
  printf("fail to specify i2c addr %02x", addr);
  exit(1);
}
```

To send command `start_periodic_measurement`:

```c
__u8 command[2] = { 0x21, 0xb1, };
if (write(device, command, 2) != 2) {
  printf("fail to write\n");
}
```

The result of `read_measurement` has a CRC (Cyclic Redundancy Check) checksum, let's copy and adapt the CRC function in the SCD4x datasheet.

```c
__u8 crc(__u8* data, __u16 count) {
  __u16 current_byte;
  __u8 crc = CRC8_INIT;
  __u8 crc_bit;
  /* calculates 8-Bit checksum with given polynomial */
  for (current_byte = 0; current_byte < count; ++current_byte) {
    crc ^= (data[current_byte]);
    for (crc_bit = 8; crc_bit > 0; --crc_bit) {
      if (crc & 0x80)
        crc = (crc << 1) ^ CRC8_POLYNOMIAL;
      else
        crc = (crc << 1);
    }
  }
  return crc;
}
```

To read measurement:

```c
__u8 command[2] = { 0xec, 0x05, };
__u8 result[10];
if (write(device, command, 2) != 2) {
  printf("fail to write\n");
}
if (read(device, result, 9) != 9) {
  printf("read fails\n");
}
for (int i = 0; i < length; i += 3) {
  if (crc(data + i, 2) != data[i + 2]) {
    printf("crc fails on %d", i);
  }
}
int co2 = data[0];
co2 <<= 8;
co2 += data[1];
float t = data[3];
t *= 256;
t += data[4];
t = -45 + (175 * t) / 0x10000;
float rh = data[6];
rh *= 256;
rh += data[7];
rh = (rh * 100) / 0x10000;;
printf("CO_2: %dppm; T: %fC; RH: %f%%\n", co2, t, rh);
```

But this is ugly. let's add an abstract layer and define:

```c
int scd4x_send_command(int device, __u8 addr, __u16 command);
int scd4x_write(int device, __u8 addr, __u16 command, __u16 datum);
int scd4x_read(int device, __u8 addr, __u16 command, __u8* data, int length);
int scd4x_send_and_fetch(int device, __u8 addr, __u16 command, __u16 datum,
                          __u8* data, int length);
```

Then we can implement all the funtions of SCD41 happily:

```c
int start_periodic_measurement();
int read_measurement(__u16* co2_concentration, float* temperature,
                     float* relative_humidity);
int stop_periodic_measurement();
int set_temperature_offset(float temperature_offset);
int get_temperature_offset(float* temperature_offset);
int set_sensor_altitude(__u16 altitude);
int get_sensor_altitude(__u16* altitude);
int set_ambient_pressure(__u16 ambient_pressure);
int perform_forced_recalibration(__u16 target_co2_concentration,
                                 __u16* frc_correction);
int set_automatic_self_calibration_enabled(__u16 asc_enabled);
int get_automatic_self_calibration_enabled(__u16* asc_enabled);
int start_low_power_periodic_measurement();
int get_data_ready_status(__u16* signal);
int persist_settings();
int get_serial_number(__u16* serial_number_0, __u16* serial_number_1,
                      __u16* serial_number_2);
int perform_self_test(__u16* sensor_status);
int perform_factory_reset();
int reinit();
// SCD41 only
int measure_single_shot();
int measure_single_shot_rht_only();
```

Note: Combined transactions of mixing read and write messages are not supported. 

> Note that only a subset of the I2C and SMBus protocols can be achieved by the means of read() and write() calls. In particular, so-called combined transactions (mixing read and write messages in the same transaction) aren't supported. For this reason, this interface is almost never used by user-space programs. [ref](https://www.kernel.org/doc/Documentation/i2c/dev-interface)

But I currently don't have much need to do so. Will fix it in the future.

Now we can read our sensor data:

```c
char buff[512];
__u16 co2_concentration;
float temperature;
float relative_humidity;
read_measurement(&amp; co2_concentration, &amp; temperature, &amp;
                 relative_humidity);
sprintf(buff, "CO_2=%dppm,T=%fC,RH=%f%%\n", co2_concentration, temperature,
        relative_humidity);
printf("%s", buff);
```


## Publishing to mqtt server
Here we use Eclipse Mosquitto&trade; ([ref](https://mosquitto.org/)) to publish to my mqtt server. The server side is also Mosquitto so I use it again in client side.

```bash
apt install libmosquitto-dev
```

Include the header

```c
#include <mosquitto.h>
```

Get the server information from environment variables

```c
char* mqtt_host = getenv("MQTT_HOST");
char* mqtt_port_s = getenv("MQTT_PORT");
char* mqtt_username = getenv("MQTT_USERNAME");
char* mqtt_password = getenv("MQTT_PASSWORD");
if (mqtt_host == NULL || mqtt_port_s == NULL || mqtt_username == NULL ||
    mqtt_password == NULL) {
  printf(
      "one of the environment varabiles not set: MQTT_HOST, MQTT_PORT, "
      "MQTT_USERNAME, MQTT_PASSWORD\n");
  exit(1);
}
int mqtt_port = strtol(mqtt_port_s, (char**)NULL, 10);
```

Connect to mqtt server ([ref](https://mosquitto.org/api/files/mosquitto-h.html))

```c
char mqtt_client_id[32];
sprintf(mqtt_client_id, "scd4x_%s", serial_number);
char topic[64];

struct mosquitto* mosq = NULL;
mosquitto_lib_init();
mosq = mosquitto_new(mqtt_client_id, true, NULL);
mosquitto_username_pw_set(mosq, mqtt_username, mqtt_password);
mosquitto_connect(mosq, mqtt_host, mqtt_port, mqtt_keep_alive);
mosquitto_loop_start(mosq);
```

Let's try to publish something

```c
sprintf(topic, "test/%s", mqtt_client_id);
sprintf(buff, "it works!");
printf("mqtt:topic='%s',msg='%s'\n", topic, buff);
mosquitto_will_set(mosq, topic, strlen(buff), buff, 0, 0);
```

I can now receive the message on my mosquitto MQTT server.

## Integration with Home Assistant

We follow the pattern of mqtt discovery from Home assistant so that we can view the sensor data in home assistant ([ref](https://www.home-assistant.io/integrations/mqtt/#mqtt-discovery)) and use suppported device classes in home assistant ([ref](https://www.home-assistant.io/integrations/sensor/)):

```c
sprintf(topic, "homeassistant/sensor/%s_CO2/config", mqtt_client_id);
sprintf(buff,
        "{\"device_class\":\"carbon_dioxide\",\"name\":\"CO2 "
        "Concentration\",\"state_class\":\"measurement\",\"unique_id\":\"%s_"
        "CO2\",\"state_topic\":\"homeassistant/sensor/%s/"
        "state\",\"unit_of_measurement\":\"ppm\",\"value_template\":\"{{ "
        "value_json.co2_concentration }}\"}",
        mqtt_client_id, mqtt_client_id);
printf("mqtt:topic='%s',msg='%s'\n", topic, buff);
mosquitto_publish(mosq, NULL, topic, strlen(buff), buff, 0, 0);

sprintf(topic, "homeassistant/sensor/%s_T/config", mqtt_client_id);
sprintf(buff,
        "{\"device_class\":\"temperature\",\"name\":\"Temperature\",\"state_"
        "class\":\"measurement\",\"unique_id\":\"%s_T\",\"state_topic\":"
        "\"homeassistant/sensor/%s/"
        "state\",\"unit_of_measurement\":\"°C\",\"value_template\":\"{{ "
        "value_json.temperature }}\"}",
        mqtt_client_id, mqtt_client_id);
printf("mqtt:topic='%s',msg='%s'\n", topic, buff);
mosquitto_publish(mosq, NULL, topic, strlen(buff), buff, 0, 0);

sprintf(topic, "homeassistant/sensor/%s_RH/config", mqtt_client_id);
sprintf(buff,
        "{\"device_class\":\"humidity\",\"name\":\"Relative "
        "Humidity\",\"state_class\":\"measurement\",\"unique_id\":\"%s_RH\","
        "\"state_topic\":\"homeassistant/sensor/%s/"
        "state\",\"unit_of_measurement\":\"%%\",\"value_template\":\"{{ "
        "value_json.relative_humidity }}\"}",
        mqtt_client_id, mqtt_client_id);
printf("mqtt:topic='%s',msg='%s'\n", topic, buff);
mosquitto_publish(mosq, NULL, topic, strlen(buff), buff, 0, 0);
```

After successfull registration in Home Assistant, we can publish sensor data to specified `state_topic`:

```c
sprintf(topic, "homeassistant/sensor/%s/state", mqtt_client_id);
sprintf(buff,
        "\"co2_concentration\":%d,\"temperature\":%f,"
        "\"relative_humidity\":%f}",
        co2_concentration, temperature, relative_humidity);
printf("mqtt:topic='%s',msg='%s'\n", topic, buff);
mosquitto_publish(mosq, NULL, topic, strlen(buff), buff, 0, 0);
```

![pending screenshot home assistant]()

## Long(er) term storage, better dashboard and alerts
By default, Home Assistant has 10 day retention for history data. My Prometheus instance which has 12 weeks retention configured. Prometheus is not designed for long term storage either, but currently fullfil my requrement.

Adding prometheus support on Home Assistant is simple, just add a line of `prometheus:` to its configuration file ([ref](https://www.home-assistant.io/integrations/prometheus/)), and configure target and authentication on Prometheus side.

![pending screenshot grafana]()

Once the data are in Prometheus' database, we can query them from Grafana.

## Postscript

When I read the metric for the first time, it was around 2500ppm. I doubted whether I was reading something else but the CRC is correct and temperature and relative humidity readings looked accurate. Then I realized that I was in a 16m^2 room which hasn't been ventilized for days. I opened the door and the window, and swiched on a fan to blow the air outside of the door. Within 10 minutes the CO_2 reading decreased to about 420ppm, meanwhile the temperature decreased 5°C. As to me? I felt only cold. It turned out that I'm not sensitive to CO_2 concentration at all. I can remember it happened several times when some friend entered the room I stay and asked me to open the window because they felt stuffy. I thought that was a polite way of disguising the excuse to mention the bad odor (Maybe there was indeed bad odor, who knows). So the CO_2 concentration doesn't decrease my cognitive function. It doesn't function well no matter the CO_2 concentration.

Some obversations: 

- When the window and door are closed and nobody is in the room, if we put temperature and relative humidity readings together, with proper scale and range, the two curves are usually symetric. This is because when the amount of water in the air doesn't change, relative humidity is decided by water's vapor pressure, which in this case only related to temperature. So based on some calculations maybe we can figure out how much water is brought into or out of the room. This change usually relates to number of people in the room and when I took shower (I bring a wet towel back). 

![temp rh cor]()

- If I only open the window but not the door, the CO_2 concentration can go up to 800ppm. I'm not sure it is good enough but it's far better than 2000+ppm when both door and window are shut. 
- When nobody is in the room, the CO_2 concentration will also decrease to under 500ppm quite fast (but hours instead of minutes). I believe it's not because of the only two little plants. That must be the reason why I haven't suffocated yet.
- At last I have 3 temperature / relative humidity sensors at hand. No two of them have equal reading, and I have no method for calibration. Withstanding the inaccuracy and acquire information from noisy readings is a necessary ability. 

Anyway regular ventilation is a good habbit. I should do so no matther I feel stuffy or not.

## Appendix: How not to configure WiFi adapter for Raspberry Pi (this chapter is full of failures and can be skipped. )
this chapter is full of failures and can be skipped. I leave it here just in case some one wants to give a try and can use some of the information. 

To connect to internet from my room, I hoped to use a wireless adapter. I bought several different adapters but at last none of them worked. After several days trying, I finally gave up on this.
### drivers from source (failed)
https://github.com/fastoe drivers doesn't work on RPi1b. 

### precompiled drivers (failed)
- using http://downloads.fars-robotics.net/wifi-drivers/8822bu-drivers/

- downgrade kernel to 5.10.73+ with `rpi-update 9fe1e973b550019bd6a87966db8443a70b991129` ([ref](https://github.com/Hexxeh/rpi-firmware/blob/9fe1e973b550019bd6a87966db8443a70b991129/uname_string), [ref](https://raspberrypi.stackexchange.com/questions/19959/how-to-get-a-specific-kernel-version-for-raspberry-pi-linux))

- lock kernel version by `apt-mark hold libraspberrypi-bin libraspberrypi-dev libraspberrypi-doc libraspberrypi0 raspberrypi-bootloader raspberrypi-kernel raspberrypi-kernel-headers` ([ref](https://forums.raspberrypi.com//viewtopic.php?p=1701547#p1701547))

```bash
wget http://downloads.fars-robotics.net/wifi-drivers/install-wifi -O /usr/bin/install-wifi
chmod +x /usr/bin/install-wifi
install-wifi
```

Failed. system hang after wlan0 read

### older kernel with precompiled drivers (failed)
- image from https://downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2020-02-14/
- using `dd` to flash image
- using `parted` to resize partition to avoid wasting space
- using `e2fsck -f` and `resize2fs` to make size valid
- boot into rpi
- repeat last chapter
- result: wlan0 appears but cannot be brought up.
