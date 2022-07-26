# Sonoff Mini Home Assistant

This guide was written using Sonoff Mini v1.0 (MPN:IM190416001 on the back of the box)

This will be the steps we make:\
`Original Firmware` &rarr; `Tasmota Firmware` &rarr; `ESPHome Firmware`


### **Getting the Sonoff ready**
- power on Sonoff Mini.
- Download eWeLink app, configure the sonoff as you normally would (hold the button for 5s to enter pairing mode)
- Upgrade its firmware to the latest version (currently 3.6.0)


### **Installing Tastmota Firmware**
- Put the Sonoff in **DIY Mode** following the steps below (found [here](https://tasmota.github.io/docs/Sonoff-DIY/#more-info)):
    - Long press the button for 5 seconds to enter pairing mode, then press another 5 seconds to ender Compatible Pairing Mode (AP). The LED indicator should blink continuously.
    - From a mobile device, connect to a WiFi named like `ITEAD-XXXXXXXX` with password `12345678`
    - On the same device, open a browser and go to [http://10.10.7.1/](http://10.10.7.1/), this will open the WiFi settings of the Sonoff
    - Here click **WIFI SETTING**, insert SSID and Password of your real WiFi connection where you want to connect the Sonoff and press **SAVE**.
    > Note: the first time you do this will probably fail. It will connect your sonoff to your wifi but the LED on the Sonoff will be ON without blinking and will not be in DIY mode. Just repeat the steps from the first (long press button etc) and everything will be good.

- Check if your Sonoff is in DIY mode by opening a terminal window on your PC and running these 2 commands (macOS or Linux only, sorry windows):
    ```sh
    export SONOFF_IP=<your_sonoff_ip> # replace this with your Sonoff IP

    curl -XPOST --header "Content-Type: application/json" --data-raw '{"deviceid": "", "data": {}}' http://$SONOFF_IP:8081/zeroconf/info
    ```
    It should respond with a json with some data.
- Unlock OTA with this command:
    ```sh
    curl -XPOST --header "Content-Type: application/json" --data-raw '{"deviceid": "", "data": {}}' http://$SONOFF_IP:8081/zeroconf/ota_unlock
    ```
- Flash Tasmota Firmware:
    > Before going on note one thing: Sonoff Mini supports a **MAXIMUM** of 508KB of firmware size, **do NOT** flash anything bigger.

    You will find the lite firmware version on [sonoff-ota.aelius.com](http://sonoff-ota.aelius.com/), go there and copy the `SHA256` string of that firmware and put it inside the `HASH` variable in the first command below.
    ```sh
    export HASH=<sha256_firmware_hash>

    curl -XPOST --data "{\"deviceid\":\"\",\"data\":{\"downloadUrl\": \"http://sonoff-ota.aelius.com/tasmota-latest-lite.bin\", \"sha256sum\": \"$HASH\"} }" http://$SONOFF_IP:8081/zeroconf/ota_flash
    ```
- Pick your mobile device and connect to a WiFi network named like `tasmota_XXXXXX-####` without any password ([taken from Tasmota initial configuration](https://tasmota.github.io/docs/Getting-Started/#initial-configuration))
- Open a browser and go to [http://192.168.4.1](http://192.168.4.1) (it could open automatically for you) and insert again your real WiFi Credentials, then hit **Save**.
- When done, go to your PC and open a browser on your `Sonoff IP`, it should open the Tasmota Web UI.


### **Installing ESPHome Firmware**
To keep things simple, we will use ESPHome CLI, so you will need [Python](https://www.python.org/downloads/) installed on your PC.

- Install ESPHome CLI on your PC with these 2 commands:
    ```sh
    pip3 install wheel
    pip3 install esphome
    ```
- Create a new folder where you will store (locally) your ESPHome firmwares and config files.
In my case I made it under `/home/<user>/Projects/esphome`
- Inside that folder create a `start.sh` file, with this content:
    ```sh
    esphome dashboard ./config/
    ```
- Inside that same folder, create a new `config/` folder, used by the command above
> Remember to **backup your config files**, those are the "source" of your firmwares containing their configuration, they cannot be recovered from an existing flashed device. It would be a good idea to make a private git repo to store them.
- Inside the `config/` folder create a `secrets.yaml` file that will contain sensitive informations like WiFi SSID, Passwords, API keys etc, and insert this content inside it (insert your actual WiFi data, and some new password for api and ota):
    ```yaml
    # Your Wi-Fi SSID and password
    wifi_ssid: "My WiFi Name"
    wifi_password: "my_wifi_password"
    api_password: "invent_a_password"
    ota_password: "invent_a_password"
    ```
- Again, in the `config/` folder create a `sonoffmini.yaml` that will be your sonoff configuration for your firmware, and insert this (remember to customize device_name, device_ip and device_gateway if needed):
    > Config for Sonoff Mini used as a lamp, see more [here](https://www.esphome-devices.com/devices/Sonoff-Mini-Relay).\
    Just a tip: decide your device IP that you want **BEFORE** doing any flashing, and then put it in here, because [changing IP afterwards](https://community.home-assistant.io/t/can-i-change-ip-address-over-the-air/168870) is a bit annoying. Remember to assign that IP to your Sonoff inside your home Router.
    ```yaml
    substitutions:
        device_name: sonoffmini
        device_ip: 192.168.1.x
        device_gateway: 192.168.1.1

    esphome:
        name: ${device_name}
        platform: ESP8266
        board: esp01_1m

    wifi:
        ssid: !secret wifi_ssid
        password: !secret wifi_password
        manual_ip:
            static_ip: ${device_ip}
            gateway: ${device_gateway}
            subnet: 255.255.255.0
        # Enable fallback hotspot (captive portal) in case wifi connection fails
        ap:
            ssid: "ESPHOME"
            password: "12345678"

    logger:
        
    api:
        reboot_timeout: 15min
        password: !secret api_password

    ota:
        password: !secret ota_password

        # the web_server & sensor components can be removed without affecting core functionaility.

    web_server:
        port: 80

    sensor:
        - platform: wifi_signal
            name: ${device_name} Wifi Signal Strength
            update_interval: 60s
        - platform: uptime
            name: ${device_name} Uptime
        #######################################
        # Device specific Config Begins Below #
        #######################################

    binary_sensor:
        # the 7 lines below define the reset button
        - platform: gpio
            pin: GPIO00
            id: reset
            internal: true  # hides reset switch from HomeAssistant
            filters:
            - invert:
            - delayed_off: 10ms
        # the 3 lines below toggle the main relay on press of reset button
            on_press:
            - light.toggle:
                id: light_id

        # the 13 lines below toggle the main relay on command
        - platform: gpio
            name: relay_toggle
            internal: true  # hides relay toggle from HomeAssistant
            pin: GPIO04
            id: gpio_light_id
            on_press:
            then:
                - light.toggle:
                    id: light_id
            on_release:
            then:
                - light.toggle:
                    id: light_id

        # the 2 lines below create a status entity in HomeAssistant.
        - platform: status
            name: ${device_name} Status

    status_led:
        pin:
            number: GPIO13
            inverted: true

    output:
        # the 3 lines below control the main relay
        - platform: gpio
            pin: GPIO12
            id: main_light_relay  
        # the 3 lines below control the Blue LED
        - platform: esp8266_pwm
            id: blue_led
            pin: GPIO13
            inverted: True

    light:
        # the 4 lines below define the main relay as a light
        - platform: binary
            name: ${device_name}_light # you can enter a custom name to appear in HomeAsistant here.
            output: main_light_relay  
            id: light_id
        # the 4 lines below define the Blue LED light on Sonoff Mini, to expose in HomeAssistant remove line "internal: true"
        - platform: monochromatic
            name: ${device_name}_blueled
            output: blue_led
            internal: true # hides the Blue LED from Homeassistant
    ```
- Make `start.sh` executable and run it:
    ```sh
    chmod +x start.sh
    ./start.sh
    ```
    This will start a Web UI to manage your ESPHome configs and install them on your devices on [http://localhost:6052](http://localhost:6052)
- Here you should see your config created before and your secrets under the secrets section
- We now need to create the firmware in `.bin` format to manually flash it inside our Sonoff
    - In the ESPHome Web UI, on your sonoffmini config "box" click the bottom right "3-dots" button, and click "Install"
    - On the window that opens up, click **Manual Download** > **Modern Format**
    - After it finishes it will download a `sonoffmini-factory.bin` file. That, finally, is our ESPHome firmware.
- In your browser open your Sonoff IP to access Tasmota, click **Firmware Upgrade**, under "Upgrade by file upload" upload your `sonoffmini-factory.bin` and click **Start upgrade**
- Once finished you should see the ESPHome Web interface opening a browser tab on your Sonoff IP

### **Add to Home Assistant**
- Open the [Home Assistant ESPHome Integration](https://www.home-assistant.io/integrations/esphome/) page
- Click the blue "**Add integration**" button to open your Home Assistant dashboard
- Here proceed with the configuration, using as **Host** your Sonoff IP, and leaving the **Port** as it is.

Done! Now you should have your Sonoff between the devices configured in your Home Assistant ready to be used.