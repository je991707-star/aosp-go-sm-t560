# SM-T560 (gtelwifi) LineageOS 16.0 Go + SoftAP 專案架構

## 根目錄（本 git 倉庫追蹤範圍）
- `README.md`：GitHub 首頁說明（裝置、修改摘要、測試結果、已知限制、編譯與刷機步驟），細節連回本文件。
- `CLAUDE.md`：專案規則（繁中回應、todo、更動須同步本文件並 commit）。
- `AGENT.md`：執行環境（`distrobox enter aosp8`）、硬體限制（`-j6`、`ccache 30G`）、搜尋範圍與分階段目標。
- `.gitignore`：白名單模式，忽略一切後只放行根目錄文件與 `.repo/local_manifests/`，原始碼樹、`out/`、`build.log`、`sync.log` 都不追蹤。
- `.repo/local_manifests/gtelwifi.xml`：裝置相關倉庫清單（見下）。
- `patches/<子倉庫路徑>/`：各子倉庫 `gtelwifi-go` 分支相對上游的修改，以 `git format-patch m/lineage-16.0..gtelwifi-go` 匯出（`device/samsung/gtelwifi` 3 個、`device/samsung/scx35-common` 2 個、`frameworks/opt/net/wifi` 1 個）。從零重建時：依「原始碼基底」`repo init`、放入 local manifest、`repo sync` 後，在各子倉庫執行 `repo start gtelwifi-go .` 與 `git am <根目錄>/patches/<子倉庫路徑>/*.patch`。子倉庫修改後需重新匯出 patch。

## 原始碼基底
- `repo init -u https://github.com/LineageOS/android.git -b lineage-16.0 --depth=1`（Android 9，AOSP tag `android-9.0.0_r46`）。
- 編譯目標 `lineage_gtelwifi-userdebug`，產物為 `make bacon` 的刷機包。`vendor/lineage/bootanimation/Android.mk` 在 kati 階段要求主機有 ImageMagick（`mogrify`），缺少時直接中止。
- `vendor/lineage/config/BoardConfigKernel.mk` 對內核編譯加上 `-j$(nproc)`（設備樹無法覆寫），編譯前需 `export OMP_NUM_THREADS=6` 讓 `nproc` 回報 6，維持 `-j6` 上限。
- 子倉庫的修改放在 `repo start` 建立的 `gtelwifi-go` 分支（`device/samsung/gtelwifi`、`device/samsung/scx35-common`、`frameworks/opt/net/wifi`），避免 `repo sync` 覆蓋。
- `vendorsetup.sh` 在 `source build/envsetup.sh` 時會把設備樹 `patches/` 直接套用到 `frameworks/base`、`packages/providers/MediaProvider` 工作目錄（未 commit，屬上游設計，勿提交）。
- `external/chromium-webview/prebuilt/*` 以 Git LFS 存放 `webview.apk`，容器須安裝 `git-lfs`；只對 `arm` 執行過 `git lfs pull`（裝置為 32-bit ARM），其餘架構保留 LFS 指標檔。

## 裝置倉庫（`.repo/local_manifests/gtelwifi.xml`）
所有 `gtel3g/*` 倉庫皆來自 GitHub `gtel3g` 組織，分支 `lineage-16.0`，`clone-depth="1"`。`github` remote 沿用 LineageOS 主 manifest 的定義（重複定義會讓 `repo init` 失敗）。

| 路徑 | 倉庫 | 分支 | 用途 |
|---|---|---|---|
| `device/samsung/gtelwifi` | `gtel3g/android_device_samsung_gtelwifi` | lineage-16.0 | 設備樹；`lineage_gtelwifi.mk`（繼承 `common_mini_tablet_wifionly.mk`）→ `gtelwifi.mk` → `scx35-common/common.mk`；`vendorsetup.sh` 於 `envsetup` 時自動套用 `patches/`（frameworks_base、MediaProvider） |
| `vendor/samsung/gtelwifi` | `gtel3g/android_vendor_samsung_gtelwifi` | lineage-16.0 | 閉源 blobs；含 `proprietary/etc/wifi/bcmdhd_sta.bin`、`bcmdhd_apsta.bin`（SoftAP 韌體）、`nvram_net.txt` |
| `kernel/samsung/gtel3g` | `gtel3g/android_kernel_samsung_gtel3g` | lineage-16.0 | SC7730SE 核心（Linux 3.10.108），`gtelwifi_defconfig`；路徑由 `BoardConfig.mk` 的 `TARGET_KERNEL_SOURCE` 決定 |
| `device/samsung/scx35-common` | `gtel3g/android_device_samsung_scx35-common` | lineage-16.0 | SPRD scx35 平台共用設定（`BoardConfigCommon.mk`、`common.mk`、`sepolicy/`、`overlay/`） |
| `hardware/sprd` | `gtel3g/android_hardware_sprd` | lineage-16.0 | SPRD HAL（gralloc、healthd charger 等） |
| `prebuilts/gcc/linux-x86/arm/arm-eabi-4.8` | AOSP `platform/prebuilts/gcc/linux-x86/arm/arm-eabi-4.8` | tag android-8.1.0_r81 | 內核編譯器；`scx35-common/BoardConfigCommon.mk` 的 `KERNEL_TOOLCHAIN` 寫死此路徑，Android 9 manifest 已不含（缺少時 `TARGET_KERNEL_BINARIES` 失敗） |
| `external/stlport` | `LineageOS/android_external_stlport` | lineage-15.1 | 舊 blobs 依賴的 libstlport；無 16.0 分支，上游 manifest 同樣用 15.1 |

注意：`gtelwifiue`（SM-T560NU，高通 MSM8916）的倉庫不適用本機。

`device/samsung/gtelwifi/BoardConfig.mk` 的 `TARGET_OTA_ASSERT_DEVICE := SM-T560,gtelwifi,gtelwifixx,gtel3g`：刷機包 updater-script 只接受這些 `ro.product.device`；常見的 SM-T560 用 TWRP 是 gtel3g（SM-T561）版本，回報 `gtel3g`，未列入時刷機會以 E3004 / Error 7 中止。

## 顯示與系統屬性（`device/samsung/gtelwifi/system.prop`）
- 由 `BoardConfig.mk` 的 `TARGET_SYSTEM_PROP += device/samsung/gtelwifi/system.prop` 掛入（`scx35-common` 已先 `+=` 自己的 system.prop，所以預設的裝置 system.prop 不會自動套用，必須明確加入）。
- `ro.sf.lcd_density=213`、`ro.sf.lcd_width=129`、`ro.sf.lcd_height=207`（與 gtel3g 相同面板）。未設定時 SurfaceFlinger 會採用 SPRD hwcomposer 回報的錯誤 DPI（20320），所有 dp 被放大到畫面外，開機後只看得到設定精靈的背景色（白畫面）。
- `ro.config.swappiness=100`、`ro.config.zram.disksize=512M`：供 `scx35-common/system/etc/init/swap.rc` 設定 zram；未設定時 zram 不會啟用（Swap 0）。

## 記憶體 / 輕量化策略
- `scx35-common/common.mk` 設定 `ro.config.low_ram=false`、`ro.statsd.enable=false`，並以 `PRODUCT_ALWAYS_PREOPT_EXTRACTED_APK`、`PRODUCT_ART_TARGET_INCLUDE_DEBUG_BUILD := false`、`PRODUCT_MINIMIZE_JAVA_DEBUG_INFO` 縮小 ART 佔用；dalvik heap 用 `tablet-7in-hdpi-1024-dalvik-heap.mk`，另有 zram（`swap.rc`）。
- **不繼承 `go_defaults.mk`**：上游 `d2df462` 已移除 `go_defaults_512.mk`（Go 的 LMK/ART/dexopt 政策針對 512MB 機種），`71f291b` 為保留傳統多工畫面而覆寫 low_ram。Android 9 中 `low_ram=true` 另會讓 `NotificationManagerService` 停用第三方通知監聽/管理服務（`canUseManagedServices`）、Settings 隱藏特殊存取權項目並改用 Go 版多工畫面；它不會隱藏熱點選單（Wi-Fi 服務只用來縮小 log 緩衝）。
- 輕量化改由：無 GApps + `lineage_gtelwifi.mk` 繼承 `common_mini_tablet_wifionly.mk`（相較 full 版去掉 Recorder 與字典 overlay）。

## Wi-Fi / SoftAP
- `BoardConfig.mk`：`BOARD_WLAN_DEVICE := bcmdhd`、`BOARD_WLAN_DEVICE_REV := bcm4343`、wpa_supplicant/hostapd 皆 NL80211 + `lib_driver_cmd_bcmdhd`，STA/AP 韌體路徑 `/vendor/etc/wifi/bcmdhd_{sta,apsta}.bin`，切換參數 `/sys/module/dhd/parameters/firmware_path`。
- 本機沒有獨立 vendor 分割區，`vendor/samsung/gtelwifi/gtelwifi-vendor-blobs.mk` 把韌體複製到 `system/vendor/etc/wifi/`，執行時 `/vendor` 指向 `/system/vendor`，與上述路徑一致。
- `scx35-common/common.mk` 提供 `wpa_supplicant`、`wpa_supplicant.conf`、`hostapd`；`hidl.mk` 提供 `android.hardware.wifi@1.0-service`；`rootdir/init.wifi.rc` 將 `firmware_path` chown 給 wifi。
- 裝置 HIDL manifest（`scx35-common/configs/manifest.xml`）沒有註冊 `android.hardware.wifi@1.0::IWifi`，框架以「Vendor HAL not supported」模式運作，HAL 的韌體切換（`DriverTool::ChangeFirmwareMode`）不會執行。舊版 bcmdhd（1.141，Linux 3.10）沒有 nl80211 vendor 事件群組，因此不註冊 HAL。
- 兩套韌體：`bcmdhd_sta.bin` 為 7.119.3「Station/P2P」（無法建立 AP，hostapd 雖回報 AP-ENABLED 但驅動 `device role select failed`、無 beacon）；`bcmdhd_apsta.bin` 為 7.10.323.56「SoftAP」。驅動 `dhd_get_fw_mode()` 在路徑含 `_apsta` 時進入 HOSTAP 模式，且每次上電重新讀取 `firmware_path`。
- `frameworks/opt/net/wifi/.../WifiNative.java` `setupInterfaceForClientMode()`：無 Vendor HAL 時先 `teardownInterface()` 現有 AP 介面，再啟動 wpa_supplicant。原流程先啟動 supplicant（晶片仍在 AP 模式、不會重新上電）才拆 AP，hostapd 結束時 `wl_android_wifi_off` 讓晶片斷電，導致熱點開啟中直接開 Wi-Fi 後 wlan0 一直 DOWN。開熱點的路徑（`setupInterfaceForSoftApMode`）原本就先拆 STA 再啟動 hostapd，不受影響。
- `scx35-common/rootdir/init.wifi.rc`：`on property:init.svc.hostapd=running` 寫入 apsta 路徑、`on property:init.svc.wpa_supplicant=running` 寫回 sta 路徑；服務啟動到 `wl_android_wifi_on` 約 70–110ms，觸發一定先完成。
- `scx35-common/sepolicy/genfs_contexts`：`/module/dhd/parameters/firmware_path` 標為 `sysfs_wlan_fwpath`（平台只標記 `/module/wlan/parameters/fwpath`）；`scx35-common/sepolicy/init.te` 允許 init 寫入 `sysfs_wlan_fwpath`。
- `gtelwifi/overlay/frameworks/base/core/res/res/values/config.xml`：`config_tether_wifi_regexs` = `wlan0`、`softap0`；`config_tether_upstream_types`（`integer-array`）= 0, 1, 5, 7。gtelwifi overlay 在 `DEVICE_PACKAGE_OVERLAYS` 中排在 scx35-common 之前，優先生效。
- `gtelwifi.mk`：`net.tethering.noprovisioning=true`（跳過電信商熱點授權檢查）。
- `gtelwifi.mk` 繼承 `hardware/broadcom/wlan/bcmdhd/config/config-bcm.mk`。

## 核心網路選項（`gtelwifi_defconfig`）
- 已開啟：`IP_ADVANCED_ROUTER`、`IP_MULTIPLE_TABLES`、`NETFILTER`、`NF_NAT`、`NF_CONNTRACK(_IPV4)`、`NF_NAT_IPV4`、`IP_NF_IPTABLES`、`IP_NF_FILTER`、`IP_NF_MANGLE`、`IP_NF_TARGET_MASQUERADE`、`NETFILTER_XT_MATCH_QUOTA2`。
- Linux 3.10 沒有 `CONFIG_IP_NF_NAT`，IPv4 NAT 表由 `CONFIG_NF_NAT_IPV4` 提供。
