# rtl88x2eu-20230815
Linux Driver for WiFi Adapters that are based on the RTL8812EU and RTL8822EU Chipsets - v5.15.0.1  
``` rtl88x2EU_rtl88x2CU-VE_WiFi_linux_v5.15.0.1-186-g768722062.20230815_COEX20230616-330a-beta.tar.gz ```  
(Checkout [commit 690d429](https://github.com/libc0607/rtl88x2eu-20230815/commit/690d429ec272892d5388d744097e3c3cb15dad1b) for original driver)

I've asked LB-LINK (a Wi-Fi module vendor) for any RTL8812EU driver. Then he sends me this tar.   
So, it should work with RTL8812EU and RTL8822EU.   

According to the file name, it may work with RTL8812CU or RTL8822CU (if exists). But you should use an in-kernel driver instead.

My personal to-do list: 
- Figure out how to make the packet injection work correctly (idk if I'm doing it correctly -- needs help) -- generally, it works! but it's not been fully tested and idk how many bugs it has (
- Test with any OpenIPC camera -- It was reported working on SSC338/Hi3516
- Figure out how to transmit in 5M/10M bandwidth (The feature is claimed to be supported in the module's product page. see [CONFIG_NARROWBAND_SUPPORTING](https://github.com/search?q=repo%3Alibc0607%2Frtl88x2eu-20230815+CONFIG_NARROWBAND_SUPPORTING&type=code) and [hal8822e_fw_10M.c](https://github.com/libc0607/rtl88x2eu-20230815/blob/v5.15.0.1/hal/rtl8822e/hal8822e_fw_10M.c))
- Build with dkms
- Test on more architecture, kernels, and distributions
- An open-source hardware design using the LB-LINK module, then share it somewhere else -- done, see [here](https://oshwhub.com/libc0607/bl-m8812eu2-demoboard-v1p0)  

PRs welcome.

## Increasing TX Power in Monitor Mode How-To  
The driver supports changing TX power dynamically with no additional patch needed.  
Just add ```rtw_tx_pwr_by_rate=0 rtw_tx_pwr_lmt_enable=0``` when ```insmod```, then use ```iw set txpower fixed```.

The relative TX gain under different settings was measured by my HackRF with the same gain setting and several cascaded attenuators.   
The results do tell the difference. However, I don't have a spectrum analyzer, so I don't know the absolute TX power value.   

Be careful when you try these cmds as the adaptor can be VERY HOT. Use a good heat sink and install the antennas properly.  

Example: 
```
# load the driver
sudo modprobe cfg80211
sudo insmod 8812eu.ko rtw_tx_pwr_by_rate=0 rtw_tx_pwr_lmt_enable=0

# set monitor mode and channel; It depends on your board

# Set tx power in mbm, the range is 0~3150
# On my BL-M8812EU2 module, the real TX power measured by HackRF increased accordingly when increasing the mbm value
# e.g. when mbm increases by 500, the signal strength seen by HackRF increases by +5dB
# but when mbm is higher than ~2000 (may different), the PA starts to saturate and the increase becomes smaller
sudo iw dev wlan0 set txpower fixed <mBm>
```
Tested on my Ubuntu 22.04 VM, kernel 6.5.   

## Use with OpenIPC  
1. Add driver package to your firmware: see [this commit](https://github.com/libc0607/openipc-firmware/commit/cc990c07cc367915b74f74e87f02f199dfba2ac8), then set ```BR2_PACKAGE_RTL88X2EU_OPENIPC=y``` in your target board config
2. Edit [package/wifibroadcast](https://github.com/libc0607/openipc-firmware/blob/master/general/package/wifibroadcast/files/wifibroadcast) to support driver loading on boot 
3. Make sure ```CONFIG_WIRELESS_EXT``` is enabled in your board's kernel config (Thanks to ```邮递员派克 (953271800)``` from QQ group)


