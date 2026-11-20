# SM-T560 (gtelwifi) LineageOS 16.0 Go + SoftAP 專案架構

## 根目錄（本 git 倉庫追蹤範圍）
- `CLAUDE.md`：專案規則（繁中回應、todo、更動須同步本文件並 commit）。
- `AGENT.md`：執行環境（`distrobox enter aosp8`）、硬體限制（`-j6`、`ccache 30G`）、搜尋範圍與分階段目標。
- `.gitignore`：白名單模式，忽略一切後只放行根目錄文件與 `.repo/local_manifests/`，原始碼樹、`out/`、`build.log`、`sync.log` 都不追蹤。
- `.repo/local_manifests/gtelwifi.xml`：裝置相關倉庫清單（見下）。

## 原始碼基底
- `repo init -u https://github.com/LineageOS/android.git -b lineage-16.0 --depth=1`（Android 9，AOSP tag `android-9.0.0_r46`）。
- 編譯目標 `lineage_gtelwifi-userdebug`，產物為 `make bacon` 的刷機包。`vendor/lineage/bootanimation/Android.mk` 在 kati 階段要求主機有 ImageMagick（`mogrify`），缺少時直接中止。
- 裝置子倉庫的修改放在 `repo start` 建立的 `gtelwifi-go` 分支（`device/samsung/gtelwifi`、`device/samsung/scx35-common`），避免 `repo sync` 覆蓋。
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
| `external/stlport` | `LineageOS/android_external_stlport` | lineage-15.1 | 舊 blobs 依賴的 libstlport；無 16.0 分支，上游 manifest 同樣用 15.1 |

注意：`gtelwifiue`（SM-T560NU，高通 MSM8916）的倉庫不適用本機。

## 記憶體 / 輕量化策略
- `scx35-common/common.mk` 設定 `ro.config.low_ram=false`、`ro.statsd.enable=false`，並以 `PRODUCT_ALWAYS_PREOPT_EXTRACTED_APK`、`PRODUCT_ART_TARGET_INCLUDE_DEBUG_BUILD := false`、`PRODUCT_MINIMIZE_JAVA_DEBUG_INFO` 縮小 ART 佔用；dalvik heap 用 `tablet-7in-hdpi-1024-dalvik-heap.mk`，另有 zram（`swap.rc`）。
- **不繼承 `go_defaults.mk`**：上游 `d2df462` 已移除 `go_defaults_512.mk`（Go 的 LMK/ART/dexopt 政策針對 512MB 機種），`71f291b` 為保留傳統多工畫面而覆寫 low_ram。Android 9 中 `low_ram=true` 另會讓 `NotificationManagerService` 停用第三方通知監聽/管理服務（`canUseManagedServices`）、Settings 隱藏特殊存取權項目並改用 Go 版多工畫面；它不會隱藏熱點選單（Wi-Fi 服務只用來縮小 log 緩衝）。
- 輕量化改由：無 GApps + `lineage_gtelwifi.mk` 繼承 `common_mini_tablet_wifionly.mk`（相較 full 版去掉 Recorder 與字典 overlay）。

## Wi-Fi / SoftAP
- `BoardConfig.mk`：`BOARD_WLAN_DEVICE := bcmdhd`、`BOARD_WLAN_DEVICE_REV := bcm4343`、wpa_supplicant/hostapd 皆 NL80211 + `lib_driver_cmd_bcmdhd`，STA/AP 韌體路徑 `/vendor/etc/wifi/bcmdhd_{sta,apsta}.bin`，切換參數 `/sys/module/dhd/parameters/firmware_path`。
- 本機沒有獨立 vendor 分割區，`vendor/samsung/gtelwifi/gtelwifi-vendor-blobs.mk` 把韌體複製到 `system/vendor/etc/wifi/`，執行時 `/vendor` 指向 `/system/vendor`，與上述路徑一致。
- `scx35-common/common.mk` 提供 `wpa_supplicant`、`wpa_supplicant.conf`、`hostapd`；`hidl.mk` 提供 `android.hardware.wifi@1.0-service`；`rootdir/init.wifi.rc` 將 `firmware_path` chown 給 wifi。
- `scx35-common/sepolicy/genfs_contexts`：`/module/dhd/parameters/firmware_path` 標為 `sysfs_wlan_fwpath`（平台只標記 `/module/wlan/parameters/fwpath`），讓 `hal_wifi` 在 enforcing 下可寫入以切換 AP 韌體。
- `gtelwifi/overlay/frameworks/base/core/res/res/values/config.xml`：`config_tether_wifi_regexs` = `wlan0`、`softap0`；`config_tether_upstream_types`（`integer-array`）= 0, 1, 5, 7。gtelwifi overlay 在 `DEVICE_PACKAGE_OVERLAYS` 中排在 scx35-common 之前，優先生效。
- `gtelwifi.mk`：`net.tethering.noprovisioning=true`（跳過電信商熱點授權檢查）。
- `gtelwifi.mk` 繼承 `hardware/broadcom/wlan/bcmdhd/config/config-bcm.mk`。

## 核心網路選項（`gtelwifi_defconfig`）
- 已開啟：`IP_ADVANCED_ROUTER`、`IP_MULTIPLE_TABLES`、`NETFILTER`、`NF_NAT`、`NF_CONNTRACK(_IPV4)`、`NF_NAT_IPV4`、`IP_NF_IPTABLES`、`IP_NF_FILTER`、`IP_NF_MANGLE`、`IP_NF_TARGET_MASQUERADE`、`NETFILTER_XT_MATCH_QUOTA2`。
- Linux 3.10 沒有 `CONFIG_IP_NF_NAT`，IPv4 NAT 表由 `CONFIG_NF_NAT_IPV4` 提供。
