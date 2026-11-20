# SM-T560 (gtelwifi) LineageOS 16.0 Go + SoftAP 專案架構

## 根目錄（本 git 倉庫追蹤範圍）
- `CLAUDE.md`：專案規則（繁中回應、todo、更動須同步本文件並 commit）。
- `AGENT.md`：執行環境（`distrobox enter aosp8`）、硬體限制（`-j6`、`ccache 30G`）、搜尋範圍與分階段目標。
- `.gitignore`：白名單模式，忽略一切後只放行根目錄文件與 `.repo/local_manifests/`，原始碼樹、`out/`、`build.log`、`sync.log` 都不追蹤。
- `.repo/local_manifests/gtelwifi.xml`：裝置相關倉庫清單（見下）。

## 原始碼基底
- `repo init -u https://github.com/LineageOS/android.git -b lineage-16.0 --depth=1`（Android 9，AOSP tag `android-9.0.0_r46`）。
- 編譯目標 `lineage_gtelwifi-userdebug`，產物為 `make bacon` 的刷機包。

## 裝置倉庫（`.repo/local_manifests/gtelwifi.xml`）
所有 `gtel3g/*` 倉庫皆來自 GitHub `gtel3g` 組織，分支 `lineage-16.0`，`clone-depth="1"`。`github` remote 沿用 LineageOS 主 manifest 的定義（重複定義會讓 `repo init` 失敗）。

| 路徑 | 倉庫 | 分支 | 用途 |
|---|---|---|---|
| `device/samsung/gtelwifi` | `gtel3g/android_device_samsung_gtelwifi` | lineage-16.0 | 設備樹；`lineage_gtelwifi.mk` → `gtelwifi.mk` → `scx35-common/common.mk`；`vendorsetup.sh` 於 `envsetup` 時自動套用 `patches/`（frameworks_base、MediaProvider） |
| `vendor/samsung/gtelwifi` | `gtel3g/android_vendor_samsung_gtelwifi` | lineage-16.0 | 閉源 blobs；含 `proprietary/etc/wifi/bcmdhd_sta.bin`、`bcmdhd_apsta.bin`（SoftAP 韌體）、`nvram_net.txt` |
| `kernel/samsung/gtel3g` | `gtel3g/android_kernel_samsung_gtel3g` | lineage-16.0 | SC7730SE 核心，`gtelwifi_defconfig`；路徑由 `BoardConfig.mk` 的 `TARGET_KERNEL_SOURCE` 決定 |
| `device/samsung/scx35-common` | `gtel3g/android_device_samsung_scx35-common` | lineage-16.0 | SPRD scx35 平台共用設定（`BoardConfigCommon.mk`、`common.mk`） |
| `hardware/sprd` | `gtel3g/android_hardware_sprd` | lineage-16.0 | SPRD HAL（gralloc、healthd charger 等） |
| `external/stlport` | `LineageOS/android_external_stlport` | lineage-15.1 | 舊 blobs 依賴的 libstlport；無 16.0 分支，上游 manifest 同樣用 15.1 |

注意：`gtelwifiue`（SM-T560NU，高通 MSM8916）的倉庫不適用本機。

## Wi-Fi / SoftAP 現況（設備樹原始設定）
- `BoardConfig.mk`：`BOARD_WLAN_DEVICE := bcmdhd`、`BOARD_WLAN_DEVICE_REV := bcm4343`、wpa_supplicant/hostapd 皆 NL80211 + `lib_driver_cmd_bcmdhd`，STA/AP 韌體路徑 `/vendor/etc/wifi/bcmdhd_{sta,apsta}.bin`。
- `gtelwifi.mk` 繼承 `hardware/broadcom/wlan/bcmdhd/config/config-bcm.mk`。
