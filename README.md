# How to read the status of the Lynx Distributor device from Victron Energy
How to read the status of the Lynx Distributor device from Victron Energy without using an official Victron device. This will allow us to do remote fuse supervision.

**Warning:** Here is presented simplified electrical assemblies for educational purposes, they are not recomanded. Do your own research to ensure that you do not endanger your appliances and people.

Victron Energy is a very serious company, if they provide certain cables that are relatively expensive, it is because they integrate all that it is necessary to electrically isolate each device in prevention of the failure of one of them.

This project is in no way connected with Victron Energy, and this project has no knowledge of the inner workings of their device, other than what can be read on their [documentation]([url](https://www.victronenergy.com/upload/documents/Lynx_Distributor/Lynx_Distributor-en.pdf)). Everything presented here are simple deductions made by connecting the Lynx Distributor circuit board to an I2C bus and attempting to talk to it via a master I2C device.

## Test power supply

To the DC bus are connected :

- a 12 V power supply to simulate the power supply of the supervised bus
- a 12-48 V to 5 V transformer (use a fuse between the Lynx DC bus and the transformer), which will supply the master device and the electronic fuse supervision board
- 4 DC fuses, one per slot, as indicated by the Victron documentation

For the RJ10 cable you can use the provided one, and cut it at the end. You can also chain up to 4 Lynx Distributors together on the same I2C bus, by connecting them with RJ10 cables.

Please note: there may be other configurations depending on the country and board version, there is no reverse polarity protection, so check your connections before you power up.

- Yellow: 5V
- Green: DATA
- Orange/Red: CLOCK
- Black: GND

For this demo the 12-48 V to 5 V transformer is connected to:

- A Raspberry Pi Pico W as an I2C master device
- The Lynx Distributor Board (until 4 can be chained through 3 additional RJ10 cables, but for this demo we will use only one Lynx Distributor)

### I2C hardware setup

- Connect the Lynx I2C DATA line to the Pico W GP20 pin (pin 27)
- Connect the Lynx I2C CLOCK line to the Pico W GP21 pin (pin 26)
- Set the Lynx Distributor I2C address through the 2 way DIP switch :

      Board   A     B     C     D
      I2C-ADD 0xO8  0x09  0x0A  0x0B
      SW-1    Off   On    Off   On
      SW-2    Off   Off   On    On

### Software setup

- To report the Lynx Distributor status, we will use [ESPHOME]([url](https://esphome.io/)) in conjonction with the [HOME ASSISTANT]([url](https://www.home-assistant.io/)) project.
- At the time of writing, the Raspberry Pi W can only be used via the developer version of ESPHOME.
- The theory :

The I2C request "0x00 with one byte response" made to the Lynx Distributor can obtain the status of the DC bus (Ok or no current), as well as the status of each fuse if the bus is indeed powered.

This gives:

```
"If the 2nd bit of the response is 1, then the bus is not powered and we cannot measure the state of the fuses, otherwise the bus is powered, the fuses have been tested.

"If the 5th bit is 0 then fuse 1 is OK, otherwise it is blown".
"If the 6th bit is 0 then fuse 2 is OK, otherwise it is blown".
"If the 7th bit is 0 then fuse 3 is OK, otherwise it is blown
"If the 8th bit is 0 then fuse 4 is OK, otherwise it is blown".
```
- Here is the ESPHOME YAML configuration of the Raspberry Pi Pico W:

```
esphome:
  name: victron-lynx-distributor
  includes:
    - ve-lynx-distributor.h

rp2040:
  board: rpipicow
  framework:
    # Required until https://github.com/platformio/platform-raspberrypi/pull/36 is merged
    platform_version: https://github.com/maxgerhardt/platform-raspberrypi.git

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "REDACTED"

ota:
  password: "REDACTED"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  manual_ip:
    static_ip: REDACTED
    gateway: REDACTED
    subnet: REDACTED

  # Enable fallback hotspot in case wifi connection fails
  ap:
    ssid: "Victron-Lynx-Distributor"
    password: "REDACTED"

i2c: 
  sda: 20
  scl: 21
  scan: true
  id: bus_a

binary_sensor:
- platform: custom
  lambda: |-
    auto ve_lynx_distributor = new VELynxDistributor();
    App.register_component(ve_lynx_distributor);
    return {ve_lynx_distributor->bus_bar, ve_lynx_distributor->fuse_1,ve_lynx_distributor->fuse_2, ve_lynx_distributor->fuse_3, ve_lynx_distributor->fuse_4};

  binary_sensors:
  - name: "Bus Bar"
  - name: "Fuse 1"
  - name: "Fuse 2"
  - name: "Fuse 3"
  - name: "Fuse 4"
```

- and the ve-lynx-distributor.h file :
```
#include "esphome.h"
#include "Wire.h"

class VELynxDistributor : public PollingComponent {
  public:
    BinarySensor *bus_bar = new BinarySensor();
    BinarySensor *fuse_1 = new BinarySensor();
    BinarySensor *fuse_2 = new BinarySensor();
    BinarySensor *fuse_3 = new BinarySensor();
    BinarySensor *fuse_4 = new BinarySensor();
    
    VELynxDistributor() : PollingComponent(10000) { }
 
    void setup() override {
      Wire.begin();
    }

    void update() override {
      unsigned char distributorState = 1;

      Wire.requestFrom(0x08,1);
      while(Wire.available()){
      distributorState = Wire.read();
            
      if (distributorState & (1 << 1)) {
        bus_bar->publish_state(false);
      }
      else {
        bus_bar->publish_state(true);

        fuse_1->publish_state(!(distributorState & (1 << 4)));
        fuse_2->publish_state(!(distributorState & (1 << 5)));
        fuse_3->publish_state(!(distributorState & (1 << 6)));
        fuse_4->publish_state(!(distributorState & (1 << 7)));
      }
    }      
  }
};
```

## Conclusion

This setup is minimalist and very succinct but it provides all the data needed to build a more advanced system. Thank you for studying this demo. Please feel free to ask questions if you have any.
