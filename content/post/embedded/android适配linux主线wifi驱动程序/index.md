---
title: "Android 适配 Linux 主线 WiFi 驱动程序"
date: 2025-07-31T16:18:09+08:00
tags:
    - "Linux"
    - "Android"
    - "Embedded"
categories: "blog"
---

NanoPi M1 有若干 USB 接口但是没有 WiFi 模块，不想插网线，所以想把一个闲置的 USB WiFi Dodge 利用起来联网。

<!--more-->

# 环境

* 硬件：NanoPI M1（与 Orange Pi One 基本相同）
* 系统：[Android 7.0](http://www.orangepi.cn/html/hardWare/computerAndMicrocontrollers/service-and-support/Orange-pi-One.html)（Linux 4.4）
* USB WiFi Dodge：MediaTek MT7601U

# 最重要的事

***首先确认 Linux 本身支持这款 USB WiFi Dodge***

从 https://en.wikipedia.org/wiki/Comparison_of_open-source_wireless_drivers 这里可以看到从 Linux 4.2 开始 MediaTek MT7601U 被 mainline 支持。

# 准备工作

先按照《Orange Pi One User Manual》把 Android ROM 编出来，能正常启动，再进行适配。

源代码需要做的修改：

1. kernel 编译报错：新版 gcc 不支持 main 函数没有返回值类型导致 ncurses 相关的检查出错，需要修改 check-lxdialog.sh。

```
diff -cbr lichee/linux-4.4/scripts/kconfig/lxdialog/check-lxdialog.sh ../allwinner-h3/android/lichee/linux-4.4/scripts/kconfig/lxdialog/check-lxdialog.sh
*** lichee/linux-4.4/scripts/kconfig/lxdialog/check-lxdialog.sh	Mon Nov 13 16:51:33 2017
--- ../allwinner-h3/android/lichee/linux-4.4/scripts/kconfig/lxdialog/check-lxdialog.sh	Fri Jun 20 15:49:43 2025
***************
*** 47,53 ****
  check() {
          $cc -x c - -o $tmp 2>/dev/null <<'EOF'
  #include CURSES_LOC
! main() {}
  EOF
  	if [ $? != 0 ]; then
  	    echo " *** Unable to find the ncurses libraries or the"       1>&2
--- 47,53 ----
  check() {
          $cc -x c - -o $tmp 2>/dev/null <<'EOF'
  #include CURSES_LOC
! int main() { return 0; }
  EOF
  	if [ $? != 0 ]; then
  	    echo " *** Unable to find the ncurses libraries or the"       1>&2
```


# 适配

## Linux 部分

### 修改 config 选项
```
make menuconfig ARCH=arm
```

打开 mt7601u 驱动的编译选项【MediaTek MT7601U (USB) support】（注意先打开被依赖的功能【MAC80211】）

NOTE：由于这个驱动加载的时候需要加载 firmware，而安卓 firmware 的路径并不是内核标准路径，我只有编译成模块在启动系统以后加载才能成功（如果你能处理好 firmware 的事情可以不编译成模块）。


### 修改驱动代码


MT7601U 驱动适配新固件版本：

```
diff -cbr lichee/linux-4.4/drivers/net/wireless/mediatek/mt7601u/eeprom.h ../allwinner-h3/android/lichee/linux-4.4/drivers/net/wireless/mediatek/mt7601u/eeprom.h
*** lichee/linux-4.4/drivers/net/wireless/mediatek/mt7601u/eeprom.h     Mon Nov 13 16:51:31 2017
--- ../allwinner-h3/android/lichee/linux-4.4/drivers/net/wireless/mediatek/mt7601u/eeprom.h     Mon Jun 23 11:01:11 2025
***************
*** 17,23 ****
  
  struct mt7601u_dev;
  
! #define MT7601U_EE_MAX_VER                    0x0c
  #define MT7601U_EEPROM_SIZE                   256
  
  #define MT7601U_DEFAULT_TX_POWER              6
--- 17,23 ----
  
  struct mt7601u_dev;
  
! #define MT7601U_EE_MAX_VER                    0x0d
  #define MT7601U_EEPROM_SIZE                   256
  
  #define MT7601U_DEFAULT_TX_POWER              6
```

### 编译

```
./build.sh
```

编译完成后一定要到 android 的目录执行

```
extract-bsp
```

才能将新编译出的 kernel 和 module 同步

## Android 部分

### 代码修改

1.首先在 device/softwinner/dolphin-fvd-p1/BoardConfig.mk 中定义 WiFi 驱动

```
diff -cbr android/device/softwinner/dolphin-fvd-p1/BoardConfig.mk ../allwinner-h3/android/android/device/softwinner/dolphin-fvd-p1/BoardConfig.mk
*** android/device/softwinner/dolphin-fvd-p1/BoardConfig.mk	Mon Nov 13 16:33:28 2017
--- ../allwinner-h3/android/android/device/softwinner/dolphin-fvd-p1/BoardConfig.mk	Tue Jun 24 17:44:16 2025
***************
*** 51,56 ****
--- 51,63 ----
  BOARD_HOSTAPD_DRIVER        := NL80211
  BOARD_HOSTAPD_PRIVATE_LIB   := lib_driver_cmd_common
  
+ WIFI_VENDOR_NAME := mediatek
+ WIFI_MODULE_NAME := mt7601u
+ WIFI_DRIVER_NAME := mt7601u
+ WIFI_DRIVER_FW_PATH_STA := /etc/firmware/mt7601u.bin
+ WIFI_DRIVER_FW_PATH_AP  := /etc/firmware/mt7601u.bin
+ WIFI_DRIVER_FW_PATH_P2P := /etc/firmware/mt7601u.bin
+ 
  include hardware/aw/wlan/firmware/firmware.mk
  
  # 2. Bluetooth Configuration
```


2.将 MT7601U 的 firmware （可以从 [mediatek](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/tree/mediatek) 这里下载）放到 hardware/aw/wlan/firmware/mediatek/mt7601u/mt7601u.bin，创建 device-mt.mk，内容如下：

```
# mt7601u firmware
PRODUCT_COPY_FILES += \
   hardware/aw/wlan/firmware/mediatek/mt7601u/mt7601u.bin:/system/etc/firmware/mt7601u.bin
```

3.在 hardware/aw/wlan/firmware/firmware.mk 中添加刚才创建的 device-mt.mk 文件

```
diff -cbr android/hardware/aw/wlan/firmware/firmware.mk ../allwinner-h3/android/android/hardware/aw/wlan/firmware/firmware.mk
*** android/hardware/aw/wlan/firmware/firmware.mk	Mon Nov 13 16:38:03 2017
--- ../allwinner-h3/android/android/hardware/aw/wlan/firmware/firmware.mk	Mon Jun 23 10:41:42 2025
***************
*** 25,27 ****
--- 25,28 ----
  include hardware/aw/wlan/firmware/xradio/xr819/device-xr.mk
  include hardware/aw/wlan/firmware/qualcomm/device-qualcomm.mk
  include hardware/aw/wlan/firmware/realtek/rtl8189fs/device-rtw.mk
+ include hardware/aw/wlan/firmware/mediatek/mt7601u/device-mt.mk
```

4.修改 android/hardware/aw/wlan/wpa_supplicant_8_lib，用于适配驱动加载逻辑、wpa_supplicant 启动参数等。

```
diff -cbr android/hardware/aw/wlan/wpa_supplicant_8_lib/driver_cmd_nl80211.c ../allwinner-h3/android/android/hardware/aw/wlan/wpa_supplicant_8_lib/driver_cmd_nl80211.c
*** android/hardware/aw/wlan/wpa_supplicant_8_lib/driver_cmd_nl80211.c	Mon Nov 13 16:38:03 2017
--- ../allwinner-h3/android/android/hardware/aw/wlan/wpa_supplicant_8_lib/driver_cmd_nl80211.c	Sun Jul  6 15:49:49 2025
***************
*** 116,122 ****
  		ifr.ifr_data = &priv_cmd;
  
  		if ((ret = ioctl(drv->global->ioctl_sock, SIOCDEVPRIVATE + 1, &ifr)) < 0
! 				&& strcmp(get_wifi_vendor_name(), "xradio") != 0) {
  			wpa_printf(MSG_ERROR, "%s: failed to issue private command: %s", __func__, cmd);
  			wpa_driver_send_hang_msg(drv);
  		} else {
--- 116,123 ----
  		ifr.ifr_data = &priv_cmd;
  
  		if ((ret = ioctl(drv->global->ioctl_sock, SIOCDEVPRIVATE + 1, &ifr)) < 0
! 				&& strcmp(get_wifi_vendor_name(), "xradio") != 0
! 				&& strcmp(get_wifi_vendor_name(), "mediatek") != 0) {
  			wpa_printf(MSG_ERROR, "%s: failed to issue private command: %s", __func__, cmd);
  			wpa_driver_send_hang_msg(drv);
  		} else {
diff -cbr android/hardware/libhardware_legacy/wifi/wifi.c ../allwinner-h3/android/android/hardware/libhardware_legacy/wifi/wifi.c
*** android/hardware/libhardware_legacy/wifi/wifi.c	Mon Nov 13 16:38:03 2017
--- ../allwinner-h3/android/android/hardware/libhardware_legacy/wifi/wifi.c	Sat Jun 28 18:13:21 2025
***************
*** 851,857 ****
      char wifi_driver_fw_patch_param[256] = {0};
  
      ALOGD("Eneter: %s, fwpath = %s.\n", __FUNCTION__, fwpath);
!     if(strncmp(get_wifi_vendor_name(), "realtek", strlen("realtek")) == 0) {
          return 0;
      } else if(strncmp(get_wifi_vendor_name(), "xradio", strlen("xradio")) == 0) {
          return 0;
--- 851,859 ----
      char wifi_driver_fw_patch_param[256] = {0};
  
      ALOGD("Eneter: %s, fwpath = %s.\n", __FUNCTION__, fwpath);
!     if (strncmp(get_wifi_vendor_name(), "mediatek", strlen("mediatek")) == 0) {
!         return 0;
!     } else if(strncmp(get_wifi_vendor_name(), "realtek", strlen("realtek")) == 0) {
          return 0;
      } else if(strncmp(get_wifi_vendor_name(), "xradio", strlen("xradio")) == 0) {
          return 0;
diff -cbr android/hardware/libhardware_legacy/wifi_hardware_info/wifi_hardware_info.c ../allwinner-h3/android/android/hardware/libhardware_legacy/wifi_hardware_info/wifi_hardware_info.c
*** android/hardware/libhardware_legacy/wifi_hardware_info/wifi_hardware_info.c	Mon Nov 13 16:38:03 2017
--- ../allwinner-h3/android/android/hardware/libhardware_legacy/wifi_hardware_info/wifi_hardware_info.c	Sun Jun 29 02:41:23 2025
***************
*** 212,217 ****
--- 212,223 ----
      "-m/data/misc/wifi/p2p_supplicant.conf "
      "-e/data/misc/wifi/entropy.bin "
      "-g@android:wpa_wlan0";
+ static const char mediatek_supplicant_para[] =
+     "-ddd -iwlan0 -Dnl80211 -c/data/misc/wifi/wpa_supplicant.conf "
+     "-I/system/etc/wifi/wpa_supplicant_overlay.conf "
+     "-O/data/misc/wifi/sockets "
+     "-e/data/misc/wifi/entropy.bin "
+     "-g@android:wpa_wlan0";
  
  static enum{running, exiting, exited} thread_state = exited;
  
***************
*** 723,728 ****
--- 729,736 ----
              return xradio_p2p_supplicant_para;
          } else if (strcmp(vendor_name, "atheros") == 0) {
              return atheros_p2p_supplicant_para;
+         } else if (strcmp(vendor_name, "mediatek") == 0) {
+             return mediatek_supplicant_para;
          }
      } else {
          if (strcmp(vendor_name, "realtek") == 0) {
***************
*** 733,738 ****
--- 741,748 ----
              return xradio_wpa_supplicant_para;
          } else if (strcmp(vendor_name, "atheros") == 0) {
              return atheros_p2p_supplicant_para;
+         } else if (strcmp(vendor_name, "mediatek") == 0) {
+             return mediatek_supplicant_para;
          }
      }
      return NULL;
```

### 编译 Android 并烧录、测试

使用 adb logcat 看日志出现对应的模块加载信息

```
...
01-01 08:29:15.320  2315  9150 D WifiHW  : Start to insmod mt7601u.ko
01-01 08:29:15.320  2315  9150 D WifiHW  : module_arg=
01-01 08:29:15.320  2315  9150 D WifiHW  : module_path=/system/vendor/modules/mt7601u.ko
01-01 08:29:15.934  2315  3537 W PppoeNetworkFactory: interfaceAdded
01-01 08:29:15.945  2315  3537 W PppoeNetworkFactory: interfaceLinkStateChanged
01-01 08:29:15.945  2315  3537 D PppoeNetworkFactory: updateInterface: mIface: mPhyIface:eth0 iface:wlan0 link:down
01-01 08:29:15.945  2315  3537 D PppoeNetworkFactory: not tracker interface
01-01 08:29:15.955  2315  9150 D WifiHW  : faied to read proc/net/wireless
01-01 08:29:15.955  2315  9150 D WifiHW  : loading wifi driver...
01-01 08:29:15.957  1418  2145 D WifiHW  : Eneter: wifi_change_fw_path, fwpath = /etc/firmware/mt7601u.bin.
...
```

没有报错说明 mt7601u.ko 以及所需的 firmware 成功加载了。


找到 wpa_supplicant 相关的日志：

```
...
01-01 08:29:16.090 27437 27437 I wpa_supplicant: Successfully initialized wpa_supplicant
...
```

说明 wpa_supplicant 成功运行，此时也可以尝试使用 iw、ip 等命令直接查看和操作网卡状态，测试是否正常。

最后在 Android 图形界面测试 WiFi 连接。

# 题外话

这次适配没有修改 BoardConfig.mk 中的 BOARD_WIFI_VENDOR 相关选项：

```
...
BOARD_WIFI_VENDOR := common
WPA_SUPPLICANT_VERSION := VER_0_8_X
BOARD_WPA_SUPPLICANT_DRIVER := NL80211
BOARD_WPA_SUPPLICANT_PRIVATE_LIB := lib_driver_cmd_common
BOARD_HOSTAPD_DRIVER        := NL80211
BOARD_HOSTAPD_PRIVATE_LIB   := lib_driver_cmd_common
...
```

```common```表示启用 hardware/aw/wlan 下的代码，而有的厂商提供了自己的 wpa_supplicant_8_lib 封装，在适配这些 Wifi Dodge 的时候可能需要修改 BOARD_WIFI_VENDOR 到对应的厂商名字。
