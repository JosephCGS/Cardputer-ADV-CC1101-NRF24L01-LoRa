# Cardputer-ADV-CC1101-NRF24L01-LoRa
![9082439408](https://github.com/user-attachments/assets/f7679754-3829-4d2e-b69f-2b8fac79e70f)
Firmware Download Procedure
1、Press and hold the G0 button on the Cardputer ADV, then press the RST button to enter Download Mode.
2、Open flash_download_tool_3.9.2.exe, and select the type/mode for downloading according to the picture.
<img width="217" height="206" alt="image" src="https://github.com/user-attachments/assets/99abad9a-d58b-4c9b-b709-3f9e70ae47af" />
3、Select the COM port, configure the settings as shown in the picture, then click START to begin downloading. Wait until the download is complete, then press the RST button to run normally.
<img width="650" height="676" alt="image" src="https://github.com/user-attachments/assets/aca0dea2-f43c-4af3-b069-00ca01646d20" />

Main Firmware Modification Workflow
1、Modify the pin assignments as shown in the figure below.
<img width="465" height="445" alt="image" src="https://github.com/user-attachments/assets/14390437-e0d7-4bc0-b263-06c6f9a13015" />
2、Add an I2C pin-switching function: when reading the keyboard, switch the pins to I2C mode, and then switch them back to Output mode afterward (because the I2C pins are being shared, mode switching is required).
3、During nRF initialization, switching to I2C mode is not allowed, so the switching function must be disabled during initialization, keeping the pins in Output mode at all times.

