OTA app proejct
===============
This document explains how `OTA app` works in microbit_v2 and 
provides a guide to write an application by using `OTA app`.
In addition, it describes the design overview.

<!-- npm i -g markdown-toc; markdown-toc -i ota_app_system_document.md -->
<!-- toc -->

- [Designe overview of ota app](#design-overview-of-ota-app)  
    *[Update Scenario](#update-scenario)  
    *[Module Dependency](#module-dependency)  
    *[Key points](#key-points)  
    *[State Machine](#state-machine)  
- [Guide for demo](#guide-for-demo)
- [To do list](#to-do-list)
- [Issues](#issues)
- [Security](#security)
- [Code and Pull Request](#code-and-pull-request)

<!-- tocstop -->

## Designe overview of OTA app

OTA (Over The Air) app project starts with the aim to make OTA
as a general standard independent to specific operating systems.
As IoT devices (e.g., smart watch, smart home appliance, smart farm,
autonomous driving, smart building) are getting increased, we need to consider
"How to manage tons of its devices in terms of cost, maintenance, and security".

IoT industry has become time-to-market. This property makes 3rd parties
difficult to build their reliable IoT device.
Thus, they choose to update the software after launching their IoT device.
Normally, there are two ways to update IoT devices.

First is to update the device manually. In this case, Flash Boot Loader will
be in charge of flashing software. But this manual update is time-consuming
and it is hard to track all of the update history of IoT devices.

Second is to update the device by OTA. Since the software update is executed
from a web server wirelessly, it is convenient to update their software,
to add new features, and to improving security issues. However, 3rd parties
adopt their own OTA policy as well as a specific operating system.
Such diversity causes that some IoT devices holds brand new features
and up-to-date security, whereas other devices stay old version software
and vulnerable security in the fully connected IoT device network.

If we can standardize OTA and entrench it in IoT industry, we can overcome
such problems, while building more smarter and secure world.
To do this, I choose to implement OTA at the application layer.

There are two reasons.
First, most of devices adopt operating systems instead of bare metal code,
and modern operating systems pursue POSIX system. If an OTA app standardizes
APIs used in updating software, that APIs can be built in the POSIX-based
operating system (e.g., Linux), then this standard can proliferate to other
operating systems. It means that OTA can be general and independent to
a specific operating system. 

Second, If OTA is implemented in the application layer, developers don't need to
be limited to a specific programming language, because modern operating systems
can run applications programmed with diverse programming language.
Thus, regardless of programming language, if programmers follow OTA policy
and use APIs provided by the operating system, they can easily implement OTA.

### Update Scenario

[2022-07-22] `OTA app` proivdes updating only a new app (not driver) at runtime. 
Since flashing applications should follow `MPU alignment rule`, it is only 
possible to update an application which has the size smaller or equal than 
the size of OTA app. If an application which don't follow MPU alignment rule is 
flahsed, the loaded application will be erased.

Furtuermore, if the number of application loaded on the target board reaches to 
the maximum number of application that the target board can run, 
OTA app doesn't execute update.

[2022-08-01] `OTA app` offers loading a new app at the start address satisfying
`MPU rules`. So, you can load any size of app without considering order of apps,
if there are enough flash region to write apps. After finding the start address,
we check whether or not a new flash region for a new app inavdes other regions
occupied by other apps as fail-safety.



### Module Dependency
[2022-07-22] The following image describes `OTA app` module dependency 
implemented on Tock. We assume that `ota_uart.send.py` acts as web server which
send data to IoT device. Binary data move through `①` from the external tool.

When receiving the specified size of data (517), console callback function `②`
is triggered. Then `OTA app` parses the receiving data and do actions according 
to a command which is positioned at index 0 of the data. 

After completing actions corresponding commands, `OTA app` sends 
`ota_uart_send.py` a coressponding response for next sequence.
An application (.tbf) binary is written to flash memory via `③`. 
When flashing the binary data is done, `ota_send_uart.py` delivers crc32 value 
which it sends to `OTA app`, and it also calculates crc32 about the data which 
it received. Then, the app request `process_load_utilities.rs` to calculate 
the written binary data into flash memory, and return the resulting value 
to `OTA app` via `④ and ⑤`.

`OTA app` checks whether or not the three crc32 values are same. If there is
incorrect crc32 consistency, `OTA app` erases the loaded data.
When the update procedure passes the crc32 consistency check, `OTA app` requests
loading the loaded application through ⑥. If the loaded app doesn't follow
`MPU alignment rule`, `OTA app` erases the loaded data and don't load
the entry pointof the loaded application into `PROCESS global array` at main.rs.
  
<p align="center">
<img src="./img/OTA_app_module_dependency.png"><br>
<strong>OTA app module dependency</strong>
<p>

### Key points
 
[2022-07-22] Dynamically changing start address of flash memory and sram.
When we update an application at `runtime`, we don't have to invade flash and 
sram memory region which is occupied by the kernel and other apps.
If we commit such memory access violation, the system is going to be crushed.

To prevent this issue, there are three key variables that save the dynamically 
changing start address of unused flash and sram memory 
at grant region of "process_load_utilities.rs". 

`find_dynamic_start_address_of_writable_flash_advanced` parses an start address
satisfying `MPU rules` and an index used to save the entry point of
the loaded app into PROCESS global array.
Then it saves the address and index to `dynamic_flash_start_addr` and 
`index` at grant region separately.

```rust
struct ProcLoaderData{
    index: usize,
    dynamic_flash_start_addr: usize,
    dynamic_unsued_sram_start_addr: usize,
}
```

```rust
fn find_dynamic_start_address_of_writable_flash_advanced(
        &self,
        proc_data: &mut ProcLoaderData,
    ) -> Result<(), ProcessLoadError>
```

The most tricky part is to find out `a start address of unused sram`.
The idea is that, since we load applications by using tockloader and then the system
executes reset, we can figure out what is `the start address of unused sram` 
by getting it from `kernel::process::load_processes` at main.rs.

This returned address is saved to `dynamic_unsued_sram_start_addr`
as the initial value, when OTA app calls a command at init stage (Only one-time).

After parsing `dynamic_flash_start_addr`, `index`, and 
`dynamic_unsued_sram_start_addr`, when we first attempt to load a new application
with `OTA app` at `load_processes_advanced_air`, such three variables are used.

After loading the new application. If there is no process load error caused
by MPU rules, the entry point of the loaded app is saved to PROCESS global array,
and `dynamic_unsued_sram_start_addr` is replaced by the next start address
of unused sram. 

```rust
 fn load_processes_advanced_air(
        &self,
        proc_data: &mut ProcLoaderData,
    ) -> Result<(usize, Option<&'static dyn Process>), ProcessLoadError>
```

[2022-08-01] Finding the start address of flash based on MPU rules.
Below pseudo code shows the concept of finding a start address satisfying 
MPU rules. Let start with an assumption that ota app size is 128k.

In `ota alignment1` picture, we want to load 4k blink app.
`start_addr` starts at `0x40000` and the `appsize` comes from `OTA app`.
Then we go to the while statement. We get the size of the app already loaded
(i.e., 128k). Since parsing is possible at 0x40000, `start_addr` jumps to 
the maximum value between `appsize` and `entry_length`.

That is, `start_addr` jumps to 0x60000. At 0x60000, we can't parse anymore,
So we save the blink app from 0x60000 after checking validation of 
the start address as shown in `ota alignment2`

```
    #Variables
    start_addr = 0x40000 (sapp)
    end_addr = 0x80000 (eapp)
    'appsize' = received from OTA app

    While(start_addr < end_addr)
        get 'entry_length' from start_addr #the size of a loaded app 
        if UnableToParse
            check_overlap_region(start_addr) 
            if Ok
                Set start_addr as start point!
                return Ok
            Else
                # Set start_addr and Set entry_length to the calibrated values 
                Recalibate start_addr()

        start_addr += Max('appsize', 'entry_length')
    EndWhile
```

<p align="center">
<img src="./img/Alignment1.PNG"><br>
<strong>ota alignment1</strong>
<p>
    
<p align="center">
<img src="./img/Alignment2.PNG"><br>
<strong>ota alignment2</strong>
<p>

In `ota alignment3` picture, we will load 64k ble app. It follows same sequence
as the above pseudo code. `start_addr` starts at 0x40000 -> go to 0x60000 -> 
0x70000. We save ble app from 0x70000

<p align="center">
<img src="./img/Alignment3.PNG"><br>
<strong>ota alignment3</strong>
<p>

In `ota alignment4` picture, we will load 4k blink app. It follows same sequence
as the above pseudo code. `start_addr` starts at 0x40000 -> go to 0x60000 -> 
0x60800. We save ble app from 0x60800

<p align="center">
<img src="./img/Alignment4.PNG"><br>
<strong>ota alignment4</strong>
<p>

`Note that` this pseudo code don't consider MPU subregion rules,
because the MPU subregion rules make the implementation more complex and 
the usage of flash memory inefficiency. 

Since `tockloader` currently provides loading app bundles by decreasing size
(large -> small consecutively). The above simple pseudo code works powerfully.

[2022-08-11] 
The loaded apps from OTA app are loaded successfully even after pushing
the reset button. Since OTA app loads apps based on MPU rules, the loaded apps
located sparsely in flash memory. Thus, the loaded apps from OTA app
are not loaded after pushing the reset button. So we need to insert padding apps
between the loaded apps. Padding apps are loaded after loading a new app 
from OTA app. Below picture shows the result of `tockloader list --verbose`

<p align="center">
<img src="./img/Alignment_With_Padding_Apps.PNG"><br>
<strong>ota alignment with padding apps</strong>
<p>
    
[2022-08-11] 
There are two arrays.
`PROCESSES_REGION_FLASH_START_ADDRESS` and `PROCESSES_REGION_FLASH_SIZE`.
at main.rs. We save the start address and the size of a loaded process into the two
arrays respectively. The two arrays are used as below.
1) Check whether or not a new flash region invades the other regions occupied by
the exisiting apps at `check_overlap_region` function.
2) Check whether or not a remnant app binaray data actually are loaded into PROCESS
global array at `find_dynamic_start_address_of_writable_flash_advanced` function.
3) Check whether or not a malicious ota app write data to the flash regions occupied
by the existing apps at `check_offset_is_in_processes` function.

[2022-08-14]
Before writing TBF binary data into flash, we check TBF header validity
1) The header length isn't greater than the entire app
2) The header length is at least as large as the v2 required header (16 bytes)
3) Check consistency between the requested app size and the app size in TBF header

[2022-08-15]
Added a security feature.
1) Attack Scenario: A malicious ota app is installed via OTA app, and it deletes 
(0xff) all of the app flash region by using `nonvolatile_storage_driver`. 
2) Result: Although the malicious ota app manipulate the regions unoccupied by the 
existing apps, it doesn't have to invade the other regions occupied by the existing apps.

[2022-08-15]
Added validation check of TBF base header
1) The header length isn't greater than the entire app
2) The header length is at least as large as the v2 required header (which is 16 bytes)
3) Check Base Header Checksum consistency
4) Check consistency between the requested app size and the actual app size in TBF header


### State Machine
[2022-07-22] `OTA app` follows the below state machine.
0) [Init stage]
    - Init stage is executed in main function only one time. In this stage,
      constant value are saved. (e.g., app start address, Rom end address,
      the number of supported process)
    
When receving commands, the below state machine is executed.
1) [COMMAND_FIND_STADDR]
    - The size of an app which will be loaded are saved.
    - Request to find dynamically changing flash start address, and 
      get the address.
    - Get an index to write the entry point of the app
    - Check whether or not the index is greater or equal than 4 
      (the number of supported process) and there is enough flash region 
      to write the app
    
2) [COMMAND_WRITE_BINARY_DATA]
    - Write the app binary into flash memory (512 bytes)
    - Repeat writing the binary
    
3) [COMMAND_WRITE_PADDING_BOUNDARY]
    - Write 01, 01, 01.. (512 bytes) padding data in order to make boundaries 
      between apps.

4) [COMMAND_WRITE_PADDING_APP]
    - After loading a new app, insert padding apps between sparsely loaded apps
    - So, the loaded apps will be loaded successfully even after a reset!
    - Additionally, we also check CRC32 consistency of the inserted padding apps

5) [COMMAND_SEND_CRC]
    - Check whether or not three CRC32 values are same. If not, 
      send the external tool fail response.
      Then, the loaded app will be erased. 
    
6) [COMMAND_APP_LOAD]
    - Request loading the entry point of the loaded app. 
      If the flashed app doesn't meet `MPU alignment rule`, `OTA app` sends 
      the external tool fail response. Then, the loaded app will be erased. 
    
7) [COMMAND_APP_ERASE]
    - When receiving the erase request, it erases the loaded app.

[2022-08-01] Delete [COMMAND_WRITE_PADDING_BOUNDARY] state. It causes that 
a new app manipulates other regions already occupied by other apps!

[2022-08-11] Add [COMMAND_WRITE_PADDING_APP]
    - After loading a new app, insert padding apps between sparsely loaded apps
    - So, the loaded apps will be loaded successfully even after a reset!
    - Additionally, we also check CRC32 consistency of the inserted padding apps

[2022-08-14] Revive [COMMAND_WRITE_PADDING_BOUNDARY] state.
    - Although it causes deleting a header information of an existing app 
    - in some cases, it is conditional. So, it's ok


## Guide for demo
[2022-07-22] In the directory(tock/tool/ota_app), there is `ota_uart.py` tool
and a couple of test tbf files. After copying, and merging OTA app project code
into your local work folder, it is necessary to disable the below code snippet
at main.rs, because undesired strings (e.g., $tock) interrupt 
the communication protocol between the tool and OTA app.
After compiling the kernel code and loading it, do run the python tool 
by entering `python ota_uart.py [file name]`.

Then you will see the update procedure by `OTA app`.

```rust
let process_printer =
        components::process_printer::ProcessPrinterTextComponent::new().finalize(());
    PROCESS_PRINTER = Some(process_printer);

    let _process_console = components::process_console::ProcessConsoleComponent::new(
        board_kernel,
        uart_mux,
        mux_alarm,
        process_printer,
    )
    .finalize(components::process_console_component_helper!(
        nrf52833::rtc::Rtc
    ));
    let _ = _process_console.start();
```

[2022-08-01] If a new app is loaded by OTA app, tockloader should not be used
together with OTA app. Since tockloader adds 512 bytes of 01 padding 
from the end of an app, It causes manipulating the header information of 
another app which is next to the loaded app immediately.

[2022-08-14] OTA app can be used together with `tocklaoder erase-apps`, and it
is totally compatible with `tocklaoder erase-apps`.

## To do list
1) Adding security features
- Prevent a malicious ota app from manipulating the region occupied by
  the existing apps [2022-08-15]  
- System call filter and permission header [Todo]
2) Need to come up with an idea to meet `MPU alignment rule`
- Basic MPU alignment rules [2022-08-01]
- Subregion MPU rules [Todo]
3) Document dynamic view of `OTA app`
4) Erase function and etc..

### Issues
1) Uninteded erase of already loaded app <br>
512 bytes (1 page) size of 0x01 padding, which are attached to the end of the new app that is supposed to be loaded on the device deletes the already loaded app on the device. For exampe, when loading 128k app and then 64k app, the first 1 page (512 bytes) of 128k app will be deleted.<br>
[2022-08-14] Sloved: Added a logic to check whether or not the 128k app is 
actually loaded app. If so, we do not write 512 bytes (1 page) size of the 0x01 padding.

2) Flash Memory Leakage <br>
If we iterate loading apps by OTA app and `tockloader erase-apps`, we face the
Flash Memory Leakage issue. Because `tockloader erase-apps` does not actaully
erase the entire region of the existing apps on the device. So, when finding a start
address, the logic skips this remnant app.
[2022-08-14] Sloved: Added a logic to check whether or not the existing apps are 
actually loaded app. If not, we load a new app from there.

## Security
When implementing OTA app, it is crucial to add security features to this project. As OTA is widely utilized to update the up-to-date software, it provides the room that attackers can appropriate improperly. In OTA app design, the kernel grants the OTA app the right to access to EEPROM. So, If the interface that the OTA app uses is exposed to attackers, our system will become vulnerable to cyber security.

Thus, our strategy is as follows.<br>
1) When we design the interface to access to EEPROM, we only expose the pointer of the shared buffer between the OTA app and Kernel.<br>
2) We check whether the format of the new app follows the Elf2Tab rules.<br>
3) We verify the version of kernel and Elf2Tab.<br>
4) We examine whether CRC included in the header of the new app is correct.<br>
5) We add the digital signature to the new app and verify it before starting the flash process.<br>
6) Consider a new plan to add HSM hardware component for security algorithm.<br>

## Code and Pull Request
1) [Kernel Interface:] https://github.com/tock/tock/pull/3068
2) [OTA App:] https://github.com/tock/libtock-c/pull/281
