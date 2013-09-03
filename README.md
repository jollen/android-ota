## 課程名稱：Android OTA 介紹
##  課程大綱
- Android 更新模式介紹
-- Fastboot
-- Recovery mode
-- OTA
- Android Image 製作
-- Engineering image
-- Recovery deploy
-- OTA update image
- Android Image 客製化與 OTA
-- OTA 更新模式
-- OTA Image 的客製化
-- 實作示範

## Android 更新模式介紹

有 3 種更新 Android 軟體的方式：

- Full update image
- Update files
- Wipe and re-flash partitions（使用 fastboot）

Full image 的做法，是製做出更新的 ROM，即 update.zip，再將手機切換至 Recovery mode 來更新。Update files 則是使用 bsdiff 來更新 /system 裡的部份檔案。

最後一個做法，是 re-flash 特定的 partition，例如更新存放 boot.img  的 partition；這種做法是使用一個稱為 fastboot 的工具。這種方式，手機上 Second bootloader 必須實現 Fastboot protocol。



### Release Tools

release tools 能產生 signed update image，這個工具位於 build/tools/releasetools/，並透過 Android build system 來產生 update.zip。

如何產生 update.zip：

```
make target-files-package
```

target-files-package 也稱為 TFP。完成編譯後，可以找到以下二個 image file：

- out/target/product/mokoid/obj/PACKAGING/apkcerts_intermediates/full-apkcerts-eng.txt
- out/target/product/mokoid/obj/PACKAGING/target_files_intermediates/full-target_files-eng.zip

上述的 image 是利用 mkbookimg 來製作。針對使用 U-Boot 的裝置，請記得利用 mkimage 來製作 U-Boot 格式的 image file。

### android.os.RecoverySystem

這是 Android framework 所提供的標準 API，用來驗證與安裝 update image。以 OTA 方式更新 Android 的話，就是使用這個 API 來驗證 update image。

在下載 Update image 後，要先利用 android.os.RecoverySystem 將 RC command files 寫至 /cache/recovery，再將手機重開機至 Recovery mode。

### Recovery Console

RC 是 Recovery Console 的意思，這其實是一個精簡的開機環境，有點像以往的 Embedded Linux 的 Initial Root Filesystem。進入 Recovery Console 後，它會自動驗證 update image，接著執行更新命令。

### Updater

RC command files 採用一個稱為 Edify 的語言撰寫。請注意，RC 的 command file 並不是使用典型的 shell script 語法撰寫。

### OTA

OTA 是 Over-The-Air 的意思，它會從製造商所提供的 Server 下載 update image，並透過 Recovery Console 來更新手機軟體。

市面上的 Android 裝置大多採用這種做法，因為對一般使用者來說，這是最簡單的方式。要使用 OTA，需要有以下的基礎建設：

- 存放 update image 的 Server
- 檢查並下載 update image 的 App（the Fetcher）
- 通知用戶有軟體更新的 Notification UI，一般是與上述的 App 整合在一起

要實現 OTA，需要 Server 與 App，這二個部份都沒有 Open Source，必須廠商自行開發。不過，去年（2012）出現了一個 OTA 的免費服務：

```
```

它提供了免費的 Server，以及 Open source 的手機端 App。只要將自已的 update image 放到 otaupdater.com 上，再配合它的 App，就可以在手機加入「第三方軟體更新」的機制。

例如，知名的 CM ROM 現在就能用這種方式更新，相當的方便。除了一些社群所製作的 update image 外，其它手機廠，都是使用自已的 Server 以及 update App。

### OTA Server

Server 的部份沒有 Open Source 的解決方案，只能自行設計與實作。不過，它的原理並不複雜：

- 使用 HTTP 下載 update image
- Server 與 Device 需要定義簡單的協定（OTA Protocol）

從 Server 的角度來看，一個簡單的 RESTful Web Service，以及簡易的 Protocol 就能滿足 OTA 的需求。RESTful Web Service 使用 JSON 格式來回應 App 的請求，在 Android App 裡處理 JSON 也是很容易的事情。

### OTA Update 步驟

OTA update 的流程如下：

1. 安裝 Update Client 端 App，通常是每日檢查一次更新
2. Client 端由 Server 下載 update ROM
3. 將 update ROM 儲存至 /sdcard 目錄下
4. 將裝置重開機至 Recovery Mode
5. 可選擇性地清除 data partition 或 cache

研發重點：

- OTA Server 的架構，一般是以 HTTP Server 為主
- OTA Client 端開發，重點在使用 HTTP 下載 update ROM
- 製作 update ROM
- 過去有許多可行的架構方式，現在建議以 RESTful 架構為主

### Boot Image

Boot Image 是利用 system/core/mkbootimg 工具，將 kernel image 與 ramdisk image 再打包而成。有時也會將 second bootloader image 加入。

Android Build System 會自動幫我們調用 mkbootimg，並製作出 boot.img。使用 TFP 時，也會自動調用 mkbootimg，以製作出 update image。

### Flash Partitioning

要了解 update.zip 的更新原理，必須先研究 flash 的 partition 結構。我目前使用的工程手機，將 flash 切割如下：

```
# mount -t
```

AOSP 階段的 partiton layout 通常只有 3 個：

- boot
- system
- data

Production 時，會增加至少 3 個 partition：

- recovery
- misc
- cache

實際的規劃方式，因不同的廠商以及裝置型號，也會有所不同，要依現況為主。此外，不管是採用什麼更新方式，都無法重新分割 partition。所以 Parititon Layout 在 Production 前就要定義好，後續無法再修改。

### OTA Server

可以自行開發，或使用知名的 OTA Update Center。使用 OTA Update Center 前，需要修改 build.prop 檔案，加入 ROM image 的下載連結。oOTA Update Center 也提供一個用戶端 App。

OTA Update Center 的特點：

- 提供 Server 服務
- 每日檢查更新一次
- 自動下載並更新 ROM

將以下設定加入 build.prop：

```
otaupdater.otaid=(write your ota id here without spaces or brackets)
otaupdater.otaver=(write your ota version here without spaces or brackets)
otaupdater.otatime=(write the date+time here as: 20120820-1516 without spaces or backets)
```
支援 quirky sdcard 的裝置，可再加入以下設定：

```
otaupdater.sdcard.os=(sdcard name (e.g. sdcard2 for /sdcard2) in the main system here without spaces or brackets)
otaupdater.sdcard.recovery=(sdcard name (e.g. sdcard2 for /sdcard2) in recovery here without spaces or brackets)
```

#
otaupdater.otaid=SmoothROM7
otaupdater.otaver=201309
otaupdater.otatime=20120831-1516


# Edify Script 簡介

Edify 是撰寫 Android update script 的語言，它的語法非常簡單。在 bootable/recovery/{edify,edifyscripting,updater} 目錄下可以找到 Edify。Edify 的幾個基本函數：

- apply_patch(source_filename, target_filename, target_sha1, target_size, sha1, patch, [[sha1, patch], ...])
- getprop
- install_zip
- package_extract_dir
- package_extract_file
- read_file
- run_program

Edify script 並非直接撰寫，而是透過 build/tools/releasetools/edify_generator.py 來生成。

Edify 的說明文件位於 bootable/recovery/edify/README

### Mount Point 的對應

Recovery 怎麼知道 partition 與 mount point 的對應關係呢？方式是透過 recovery.fstab 文件。recovery.fstab 的寫法，與 Linux 的 fstab 相同。

RC 與 releasetools 都會用到 recovery.fstab 文件。

## Android Image 客製化與 OTA

### TFP

前面已介紹過，可以使用 target-files-package 這個 Makefile rule 來製作 update image。如果有額外的文件，想要另外加到 update image 裡，可以自行擴充 releasetoos。

如果 APK 是 signed，在 update image 裡的新版 APK，也必須使用同樣的 key 來 sign APK。否則新的 APK 將無法讀取原來的 app data。

APK 的 key 是利用 LOCAL_CERTIFICATE 來設定，這部份是在該 APK 的 Android.mk 裡處理。AOSP 提供 4 個 key，位於 build/target/product/security，廠商可自行新增 key：

- devkey (Development 階段使用)
- testkey (default、用於 engineering build 測試)
- platform (platform package)
- shared (與 Home/Contacts 共享資料)
- media (在 media/download 裡的 package)

TFP 會透過 ota_from_target_files 來製作最後的 update image。

Production 的裝置，不能使用上述由 AOSP 提供的 key。ota_from_target_files 的 --package_key 參數，用來指定 key。如何指定 production key？releasetools 的 sign_target_files_apks 工具會幫我們處理這個工作。

### TARGET_RELEASETOOLS_EXTENSIONS

這是用於 BoardConfig.mk 的變數，用來撰寫 release tools 的 extension。可參考 ota_from_target_files。

### 

$ reboot recovery

### BCB

RC 與 bootloader 的共用 /misc partition，這個 partition 存放一種稱為 Bootloader Control Block (BCB) 的資料。BCB 分為 3 個區塊：

- BCB.command[32]：給 bootloader 的 command
- BCB.status[32]：bootloader 回傳的 status
- BCB.recovery[1024]：RC 的 command line

BCB.command 裡的資料，是由 Linux kernel 所寫入，基本上是一行 'boot-recovery' 命令；另外，Linux kernel 也要將 BCB.recovery 清空。BCB.command 是 bootloader 選擇 Boot Image 的依據。也有廠商用來實作 Dual OS 開機。

＄（TARGET_DEVICE_DIR)/recovery/res 裡存放一個 8-bit PNG 圖檔。

### Updater

Android Build System 會將 updater 打包到 update.zip。bootable/recovery/install.c

### SYSTEM/build.prop

### META/otakeys.txt


## Asus Nexus 7 Unlock

首先，要把你的 Nexus 7 Unlock，只要利用 Fastbook 即可輕鬆完成：

```
$ ./fastboot oem unlock
```


