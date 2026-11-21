# SM-T560 (gtelwifi) LineageOS 16.0 輕量版 + Wi-Fi 熱點

讓 Samsung Galaxy Tab E 9.6 Wi-Fi（**SM-T560**，代號 `gtelwifi`）跑不含 GApps 的 LineageOS 16.0（Android 9），並修好 Wi-Fi 熱點（SoftAP），用來搭配 DiPlay（v0.2.21 以上）執行無線 CarPlay。

本倉庫只放**裝置倉庫清單與修改 patch**，不含 Android 原始碼與編譯產物。

## 裝置

| 項目 | 規格 |
|---|---|
| 型號 | SM-T560（`gtelwifi`）。**不適用** SM-T560NU（`gtelwifiue`，高通 MSM8916） |
| SoC | Spreadtrum SC7730SE，Mali-400 |
| RAM | 1.5 GB |
| Wi-Fi | Broadcom BCM43430（僅 2.4 GHz） |
| 螢幕 | 800×1280，9.6 吋 |

## 相對上游的修改

上游設備樹來自 GitHub [`gtel3g`](https://github.com/gtel3g) 組織的 `lineage-16.0` 分支。

- **Wi-Fi 熱點**
  - 裝置沒有 Wi-Fi HAL，原本開熱點時晶片仍是 STA 韌體，hostapd 回報成功但不發訊號。改由 init 在 hostapd / wpa_supplicant 啟動時切換 `bcmdhd_apsta.bin` / `bcmdhd_sta.bin`。
  - 熱點開著時直接開 Wi-Fi，原本 Wi-Fi 會卡在介面 DOWN。修改 `WifiNative`，在沒有 HAL 時先拆熱點再啟動 Wi-Fi。
  - 熱點 overlay（`wlan0`/`softap0`）、`net.tethering.noprovisioning=true`、韌體路徑的 SELinux 標籤。
- **開機白畫面**：補上 `ro.sf.lcd_density=213`（否則採用驅動回報的錯誤 DPI 20320，介面全部跑出畫面外）。
- **記憶體**：啟用 512 MB zram；保留 `ro.config.low_ram=false`（`low_ram=true` 會停用第三方通知監聽等功能），改用 Lineage 的 mini 產品設定精簡內建 App。
- **刷機**：OTA 機型檢查加入 `gtel3g`，常見的 SM-T560 TWRP 其實是 T561 版本。

詳細原因與檔案位置見 [Readme_Architecture.md](Readme_Architecture.md)。

## 實機測試結果

- ✅ 開機、觸控、螢幕顯示
- ✅ Wi-Fi 連線、上網
- ✅ Wi-Fi 熱點（手機可連線），熱點與 Wi-Fi 互相切換
- ✅ DiPlay 0.2.22 無線 CarPlay（iPhone，1280×800 H.264 硬體解碼）

## 已知限制

- **只有 userdebug 版**。SELinux 是三星開機程式傳入的 Permissive；正式 user 版會強制開啟 SELinux，已知約 125 種存取會被擋（顯示、Wi-Fi、相機等），尚未補齊規則。
- **2.4 GHz 單天線**：CarPlay 實測約 17–20 fps、1–5 Mbps，畫質受頻寬限制。
- **熱點與 Wi-Fi 無法同時開啟**（開熱點時 Wi-Fi 會關閉）。無線 CarPlay 不需要平板連網，不影響使用。

## 編譯

### 環境

Ubuntu 18.04（作者使用 Distrobox 容器），需要：Python 2.7（預設 `python`）、Python 3.6（`repo` 用）、OpenJDK 8、`repo`、`git-lfs`、ImageMagick、`ccache`。完整 build 約需 16 GB RAM、200 GB 以上磁碟空間。

### 步驟

```bash
mkdir lineage-gtelwifi && cd lineage-gtelwifi
repo init -u https://github.com/LineageOS/android.git -b lineage-16.0 --depth=1

# 放入裝置倉庫清單
git clone https://github.com/je991707-star/aosp-go-sm-t560.git ../aosp-go-sm-t560
mkdir -p .repo/local_manifests
cp ../aosp-go-sm-t560/.repo/local_manifests/gtelwifi.xml .repo/local_manifests/

repo sync -c -j6 --force-sync --no-clone-bundle --no-tags
(cd external/chromium-webview/prebuilt/arm && git lfs pull)

# 套用修改
PATCHES=$(realpath ../aosp-go-sm-t560/patches)
for p in device/samsung/gtelwifi device/samsung/scx35-common frameworks/opt/net/wifi; do
  (cd $p && repo start gtelwifi-go . && git am "$PATCHES/$p"/*.patch) || break
done
```

```bash
export USE_CCACHE=1 CCACHE_EXEC=/usr/bin/ccache
export OMP_NUM_THREADS=6   # Lineage 內核編譯用 -j$(nproc)，以此限制執行緒
ccache -M 30G
source build/envsetup.sh
lunch lineage_gtelwifi-userdebug
make -j6 bacon
```

產出：`out/target/product/gtelwifi/lineage-16.0-*-UNOFFICIAL-gtelwifi.zip`

## 刷機

已編譯好的刷機包（userdebug、test-keys 簽章）在 [Releases](https://github.com/je991707-star/aosp-go-sm-t560/releases)，下載後請核對 SHA-256。

1. 刷入 TWRP（gtel3g 版本可用，回報代號 `gtel3g`）。
2. 第一次刷：Wipe → Advanced Wipe 清除 Dalvik/ART Cache、Cache、System、Data。
3. 刷入 zip，重新開機。第一次開機約 5–10 分鐘。

## 授權

本倉庫的文件與 patch 以 [MIT](LICENSE) 授權。編譯所用的原始碼與檔案各自保有原授權：LineageOS / AOSP 多為 Apache 2.0，Linux 內核為 GPLv2（原始碼見 [`gtel3g/android_kernel_samsung_gtel3g`](https://github.com/gtel3g/android_kernel_samsung_gtel3g) `lineage-16.0` 分支），三星閉源 vendor blobs 的權利屬原廠所有。

## 致謝

- [LineageOS](https://github.com/LineageOS)
- [`gtel3g`](https://github.com/gtel3g) 組織的 SC7730SE 設備樹、內核與 vendor blobs
