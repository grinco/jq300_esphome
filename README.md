# jq300_esphome
JQ300 ESPHome Configuration and flashing instructions

0. Order an SPI flasher with a clip tool like [the one that I ordered](https://www.aliexpress.com/item/4001045543107.html), and wait...
1. Dump the original firmware using flashrom tool `flashrom -p ch341a_spi -r ./jq300-orig.bin`
2. Create the device in esphome and download firmware using this template:
```
esphome:
  name: jq300
  friendly_name: jq300
  on_boot:
    - uart.write: "device_request_all_data\n"

esp8266:
  board: esp01_1m

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

# Enable logging
logger:
  baud_rate: 115200
  hardware_uart: uart1

# Enable Home Assistant API
api:
  encryption:
    key: !secret encryption_password

ota:
  password: !secret api_password

external_components:
  - source: github://grinco/esphome-stream-server/

uart:
  id: uart_bus
  tx_pin: GPIO1
  rx_pin: GPIO3
  baud_rate: 115200
  debug:
    direction: BOTH
    dummy_receiver: false
    after:
      timeout: 1s
      delimiter: "\n"
    sequence:
      - lambda: |- 
          // Log the received data
          uint8_t separator = ':';         
          UARTDebug::log_hex(direction, bytes, separator);

          // Convert uart text to string and publish raw data
          //std::string rawdata(bytes.begin(), bytes.end());
          //id(rawString).publish_state(rawdata);

          // Command received
          if (bytes.size()==7) {
            if(bytes[2]==0xF3) { ESP_LOGD("uart_bus","short press"); }
            if(bytes[2]==0xF4) { ESP_LOGD("uart_bus","long press"); }
          }

          // Measurement received
          if (bytes.size()==15) {
          // a is bytes[4-5]
          // b is bytes[6-7]
          
          char buf[5];

          // temp is bytes[8]
          sprintf(buf, "%02X", bytes[8]);
          id(temp).publish_state(std::atoi(buf));

          // humidity is bytes[10]
          sprintf(buf, "%02X", bytes[10]);
          id(humidity).publish_state(std::atoi(buf));
          
          // dust is bytes[12-13] uint16_t((bytes[12] << 8) | bytes[13] ))
          std::string dust;
          sprintf(buf, "%02X", bytes[12]);
          dust += buf;
          sprintf(buf, "%02X", bytes[13]);
          dust += buf;
          id(pm25).publish_state(std::stoi(dust));
          }

stream_server:
    uart_id: uart_bus
    port: 10000

switch:
  - platform: gpio
    pin: GPIO0
    name: "Blue Led"
  - platform: gpio
    pin: GPIO5
    name: "Green Led" 

sensor:
  - platform: template
    name: "Temperature"
    device_class: "temperature"
    state_class: "measurement"
    accuracy_decimals: 0
    id: "temp"
  - platform: template
    name: "Humidity"
    state_class: "measurement"
    accuracy_decimals: 0
    device_class: "humidity"
    id: "humidity"
  - platform: template
    name: "PM2.5"
    state_class: "measurement"
    accuracy_decimals: 0
    device_class: "pm25"
    id: "pm25"
```
3. Download and path the firmware using xxc tool: `cat <(xxd jq300-orig.bin) <(xxd jq300.bin) | xxd -r > jq300-fullflash.bin`
4. Flash using `flashrom -p ch341a_spi -w jq300-fullflash.bin`
