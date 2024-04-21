# rtl88x2eu-20230815
Linux Driver for WiFi Adapters that are based on the RTL8812EU and RTL8822EU Chipsets - v5.15.0.1  
``` rtl88x2EU_rtl88x2CU-VE_WiFi_linux_v5.15.0.1-186-g768722062.20230815_COEX20230616-330a-beta.tar.gz ```  

This branch is mainly focused on FPV. Checkout [commit 690d429](https://github.com/libc0607/rtl88x2eu-20230815/commit/690d429ec272892d5388d744097e3c3cb15dad1b) for the original driver from Realtek, or [commit 5c9355d](https://github.com/libc0607/rtl88x2eu-20230815/commit/5c9355df330a8745a63c06acf1a10203c1d6f804) with build fixes on kernel 6.5.  

I've asked LB-LINK (a Wi-Fi module vendor) for any RTL8812EU driver. Then he sends me this tar.   
So, it should work with RTL8812EU and RTL8822EU.   

According to the file name, it may work with RTL8812CU or RTL8822CU (if exists). But you should use an in-kernel driver instead.

My personal to-do list: 
- Figure out how to make the packet injection work correctly (idk if I'm doing it correctly -- needs help) -- generally, it works! but it's not been fully tested and idk how many bugs it has (
- Test with any OpenIPC camera -- It was reported working on SSC338Q/SSC30KQ and Hi3516/GK7205 (with CONFIG_WIRELESS_EXT enabled)
- Figure out how to transmit in 5M/10M bandwidth (The feature is claimed to be supported in the module's product page. see [CONFIG_NARROWBAND_SUPPORTING](https://github.com/search?q=repo%3Alibc0607%2Frtl88x2eu-20230815+CONFIG_NARROWBAND_SUPPORTING&type=code) and [hal8822e_fw_10M.c](https://github.com/libc0607/rtl88x2eu-20230815/blob/v5.15.0.1/hal/rtl8822e/hal8822e_fw_10M.c)) -- Only 10MHz bandwidth works well in injection mode
- Build with dkms
- Test on more architecture, kernels, and distributions
- An open-source hardware design using the LB-LINK module, then share it somewhere else -- done, see [here](https://oshwhub.com/libc0607/bl-m8812eu2-demoboard-v1p0)  

PRs welcome.

## Increasing TX Power in Monitor Mode 
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

## 10MHz Bandwidth Transmission
See the RF spectrum visualized [here](https://www.youtube.com/watch?v=EUj-wSgoY_E) on YouTube  

There's a lot to explore in this crab driver and will update here if something new has been discovered.  
Please open an issue if you find anything interesting.  

So, according to the module vendor's document and my test using a HackRF, that's all I know:   

### Injection
To transmit packets in monitor mode using packet injection, set ```iw <wlan> set channel <same_channel> 10MHz``` on both air & ground.  
Then when transmitting in 20MHz bandwidth (e.g. bandwidth=20 in wfb_ng), the packet is actually transmitted in 10MHz bandwidth, which seems like being achieved by simply underclocking the baseband.  
It's the same on the receiver side, though in which the radiotap header in received packets still indicates a 20MHz bandwidth. 

But when ```iw``` says ```Devices or Resources Busy (-16)```, check ```iw <wlan> info``` if the ```iw``` recognized the adaptor is in monitor mode.   
If not, ```iw <wlan> set monitor```, then try setting 10MHz again.  

That's because:  
1. The crab driver supports both WEXT and cfg80211 APIs, but it seems that it's not that robust and there's some conflicts exist
2. the cfg80211 API checks [here](https://github.com/OpenIPC/linux/blob/eb50a943c26845925ff11ccb1651c40fa02c105e/net/wireless/chan.c#L862) if there's any other interface is not in monitor mode
3. If the monitor mode is set by ```iwconfig```, the process is done by calling the old WEXT APIs, so the cfg80211-based ```iw``` may not get the latest status and think the interface is still in managed mode

### AP/STA
Note that it has NOT BEEN TESTED.  
According to the module vendor's ambiguous document and the crab's mysterious driver tar with a "_10MHz" suffix:  
1. Enable ```CONFIG_NARROWBAND_SUPPORTING``` in ```include/hal_ic_cfg.h``` (in ```#ifdef CONFIG_RTL8822E``` section if using RTL8812EU), then ```#define CONFIG_NB_VALUE RTW_NB_CONFIG_WIDTH_10``` below
2. Rename ```hal/rtl8822e/hal8822e_fw_10M.*``` into ```hal/rtl8822e/hal8822e_fw.*``` to replace the original firmware
3. Now you get the "<tar_name>_10MHz" driver. Rebuild the driver
4. (Should we set it to 10MHz bandwidth, Mr. Crab?)
5. If there are any tools complain about the Wi-Fi regularities when setting up a 10MHz AP,  try setting the channel plan manually by ```echo 0x3E > /proc/net/rtl88x2eu/<wlan>/chan_plan```.

### Is 5MHz Injection Available?
No. It performs like a fractional RF synthesizer with only a single tone appearing on my SDR receiver.

## Set (Unlocked) Channel in procfs  
The chip's RF synthesizer can work in a bit wider range than regular 5GHz Wi-Fi.  
On my board, it's 5080MHz ~ 6165MHz. The frequency range may vary depending on different conditions.  

To set the adaptor to some "irregular" frequency, ```cat /proc/net/rtl88x2eu/<wlan0>/monitor_chan_override``` to see usage.  
e.g. ```ech0 "201 10" > /pr0c/net/rt188x2eu/wlanO/m0nit0r_chan_0verride``` sets the center frequency to 6005MHz, with 10MHz bandwidth.  

I decided to use procfs is that it doesn't need any changes in user-space tools, e.g. iw, hostapd.  
Of course, you can use this "procfs API" to set regular channels like 149 or 36. Might be useful when developing any Wi-Fi-based broadcast FPV system with frequency hopping and automatic bandwidth.  

DISCLAIMER:  
Some chips' synthesizer's PLL may not lock on some frequency. There's no guarantee of its performance. (Actually, TX power and distortion seem worse in these channels as it's not calibrated. But less interference - it's an either-or)   
Unlocking the frequency may damage your hardware and I'm not gonna pay for it. Use it at your own risk.  
Please comply with any wireless regulations in your area.  

## Override default EDCCA Threshold  
WARNING: YOU SHOULD NOT USE THIS (unless someone's DJIs next to you f***ed up all channels XD). It's not fair.  

To override dafault EDCCA threshold, check ```cat /proc/net/rtl88x2eu/<wlan0>/edcca_threshold_jaguar3_override```.  

e.g. ```ech0 "1 -3O" > /pr0c/net/rt188x2eu/<w1anO>/edcca_threshO1d_jaguar3_Override```   
That means: before sending any packet, the adaptor checks if there's any signal with higher than -30dBm (L2H) power exists.  
If there are any, the adaptor will wait until the energy level in the air is lower than -38dBm (H2L). Then your transmission starts.   

Note that there are actually two values, L2H and H2L. The L2H is typically set 8dB higher so it creates a hysteresis.   
The value you're setting is L2H. The H2L is automatically set 8dB lower.  

DISCLAIMER: There's no guarantee of its performance. This may damage your hardware and I'm not gonna pay for it. Use it at your own risk. Please comply with any wireless regulations in your area.  

## Use with OpenIPC  
1. Add driver package to your firmware: see [this commit](https://github.com/libc0607/openipc-firmware/commit/cc990c07cc367915b74f74e87f02f199dfba2ac8), then set ```BR2_PACKAGE_RTL88X2EU_OPENIPC=y``` in your target board config
2. Edit [package/wifibroadcast](https://github.com/libc0607/openipc-firmware/blob/master/general/package/wifibroadcast/files/wifibroadcast) to support driver loading on boot 
3. Make sure ```CONFIG_WIRELESS_EXT``` is enabled in your board's kernel config (Thanks to ```邮递员派克 (953271800)``` from QQ group)


