# 專案說明：SM-T560 (gtelwifi) LineageOS 16.0 Go 輕量化 + SoftAP（Wi-Fi 熱點）編譯

## 1. 執行環境架構（極重要，請優先閱讀）
- **你（Claude Code）目前運行在 BazziteOS 宿主機上**，而所有編譯工具（Python 2.7、Python 3.6（`repo` 啟動器需要）、OpenJDK 8、`repo`、`git-lfs`、`make`、`ccache`）都安裝在名為 **`aosp8` 的 Distrobox 容器（Ubuntu 18.04）** 內部。
- 宿主機與容器共享完全相同的檔案系統路徑（`/home/h97m/Projects/aosp-go-sm-t560`）。你可以直接在宿主機讀取或編輯檔案，**但所有終端機指令、git/repo 操作與編譯命令，都必須包裝在 `distrobox enter aosp8` 裡面執行**，範例如下：
  ```bash
  distrobox enter aosp8 -- bash -lc 'cd /home/h97m/Projects/aosp-go-sm-t560 && <你的指令>'
  ```
- **嚴禁**在 BazziteOS 宿主機直接執行 `repo`、`make`、`java` 或 `python`。
- 在容器內部，預設 `python` 必須維持 Python 2.7，`java` 必須維持 OpenJDK 8。

## 2. 硬體規格與記憶體限制
- 宿主機規格：Intel Xeon E3-1231 v3 (4C/8T)｜16GB DDR3 RAM + 約 7.8GB zram Swap｜約 350GB 可用 SSD 空間。
- **嚴格編譯限制**：
  - 執行 `make` / `mka` 時，執行緒數**絕對不可超過 `-j6`**（記憶體不足時降至 `-j4`）。
  - Android 9 已無 Jack Server，不需設定 `ANDROID_JACK_VM_ARGS`。
  - **務必**啟用 `ccache`（設定 30GB 上限以加速後續重編）：
    ```bash
    export USE_CCACHE=1
    export CCACHE_EXEC=/usr/bin/ccache
    export PATH="$HOME/.local/bin:$PATH"
    ccache -M 30G
    ```
- **記憶體不足（OOM）恢復方式**：若 `build.log` 出現 `Out of memory error` 或 `soong`/`ninja` 被 kill，請降速接續編譯：
  ```bash
  make -j4 bacon 2>&1 | tee build.log
  ```

## 3. 程式碼搜尋範圍與 Git 安全規範（配合 `CLAUDE.md`）
- **切勿掃描整個 LineageOS 原始碼樹**（包含數十萬個檔案，極易超時或耗盡上下文）。僅搜尋與修改以下目錄：
  - `.repo/local_manifests/`
  - `device/samsung/gtelwifi/`
  - `device/samsung/scx35-common/`
  - `kernel/samsung/gtel3g/`（SC7730SE 核心，內含 `gtelwifi_defconfig`）
  - `vendor/samsung/gtelwifi/`
  - `hardware/sprd/`
  - 專案根目錄文件（`CLAUDE.md`、`AGENT.md`、`Readme_Architecture.md`、`.gitignore`）
- **Git Commit 安全原則**：
  - 依照 `CLAUDE.md` 執行 git commit 時，**絕對不可**在原始碼根目錄對整個原始碼樹或 `out/` 執行 `git add .`。
  - 專案根目錄的 `.gitignore` 採白名單，只追蹤根目錄文件與 `.repo/local_manifests/`；原始碼修改直接在對應的子倉庫（如 `device/samsung/gtelwifi`、`kernel/samsung/gtel3g`）內建立 commit。

## 4. 目標裝置與系統基底規格
- **目標裝置**：Samsung Galaxy Tab E 9.6 Wi-Fi 版（型號 `SM-T560`，代號 `gtelwifi`）。注意 `gtelwifiue`（SM-T560NU）是高通 MSM8916 機種，倉庫不可混用。
- **硬體平台**：Spreadtrum SC7730SE 處理器、1.5GB RAM、Broadcom BCM43430 Wi-Fi/藍牙二合一晶片
- **系統基底**：`LineageOS/android` 分支 `lineage-16.0`（Android 9，使用 `--depth=1` 淺複製以節省空間）
- **編譯目標**：`lineage_gtelwifi-userdebug`（保留 LineageOS 基底以減少依賴報錯）
- **最終目的**：打造不含 GApps 的輕量化（Android Go 化）系統，且能穩定開啟 Wi-Fi 熱點（SoftAP），讓 **DiPlay（v0.2.21 以上）** 可以在 1.5GB RAM 的裝置上自動開啟熱點並流暢執行無線 CarPlay。

## 5. 分階段編譯與修改流程

### 第一階段：同步原始碼與建立 Local Manifest
1. 使用 GitHub `gtel3g` 組織的 `lineage-16.0` 倉庫（`android_device_samsung_gtelwifi`、`android_vendor_samsung_gtelwifi`、`android_kernel_samsung_gtel3g`、`android_device_samsung_scx35-common`、`android_hardware_sprd`），以及 `LineageOS/android_external_stlport`（`lineage-15.1`，無 16.0 分支）。
2. 在 `aosp8` 容器內初始化：
   ```bash
   repo init -u https://github.com/LineageOS/android.git -b lineage-16.0 --depth=1
   ```
3. 建立 `.repo/local_manifests/gtelwifi.xml`（`github` remote 由 LineageOS 主 manifest 提供，不可重複定義）。
4. 同步原始碼：
   ```bash
   repo sync -c -j6 --force-sync --no-clone-bundle --no-tags
   ```

### 第二階段：Android Go 輕量化並解鎖 Wi-Fi 熱點（SoftAP）
1. **輕量化產品設定**（`device/samsung/gtelwifi/`）：
   - 保留 `lineage_gtelwifi.mk` 與 `AndroidProducts.mk` 的產品名稱 `lineage_gtelwifi`。
   - 在產品設定中加入 Android Go 預設，並不安裝 GApps：
     ```makefile
     $(call inherit-product, $(SRC_TARGET_DIR)/product/go_defaults.mk)
     ```
     若 `go_defaults.mk` 與 LineageOS 設定衝突，則改為手動加入低記憶體屬性（`ro.config.low_ram=true` 等）。
   - *關於 `ro.config.low_ram` 的注意事項*：請確認系統設定中的「熱點與網路共用」選單不會因此被隱藏；若被隱藏，請在 `gtelwifi.mk` 中覆寫為 `ro.config.low_ram=false`，同時保留精簡後的套件組合。

2. **解鎖純 Wi-Fi 平板熱點的框架層 Overlay**（`device/samsung/gtelwifi/overlay/frameworks/base/core/res/res/values/config.xml`）：
   ```xml
   <?xml version="1.0" encoding="utf-8"?>
   <resources>
       <!-- 啟用 Wi-Fi 熱點共享介面 -->
       <string-array translatable="false" name="config_tether_wifi_regexs">
           <item>wlan0</item>
           <item>softap0</item>
       </string-array>
       <!-- 允許無 SIM 卡平板啟動熱點，跳過電信商驗證 -->
       <integer-array translatable="false" name="config_tether_upstream_types">
           <item>0</item>
           <item>1</item>
           <item>5</item>
           <item>7</item>
       </integer-array>
   </resources>
   ```
   - 同時在 `PRODUCT_PROPERTY_OVERRIDES` 中加入 `net.tethering.noprovisioning=true`。

3. **Wi-Fi 驅動、`hostapd` 與 BCM43430 AP 模式韌體設定**（`BoardConfig.mk` / `gtelwifi.mk` / `vendor/samsung/gtelwifi/`）：
   - 確認 `PRODUCT_PACKAGES` 包含 `hostapd`、`wpa_supplicant` 與 `wpa_supplicant.conf`。
   - 確認 `BoardConfig.mk` 已正確設定（現有設備樹的韌體路徑為 `/vendor/etc/wifi/`）：
     ```makefile
     BOARD_WLAN_DEVICE := bcmdhd
     WPA_SUPPLICANT_VERSION := VER_0_8_X
     BOARD_WPA_SUPPLICANT_DRIVER := NL80211
     BOARD_WPA_SUPPLICANT_PRIVATE_LIB := lib_driver_cmd_bcmdhd
     BOARD_HOSTAPD_DRIVER := NL80211
     BOARD_HOSTAPD_PRIVATE_LIB := lib_driver_cmd_bcmdhd
     WIFI_DRIVER_FW_PATH_PARAM := "/sys/module/dhd/parameters/firmware_path"
     WIFI_DRIVER_FW_PATH_STA := "/vendor/etc/wifi/bcmdhd_sta.bin"
     WIFI_DRIVER_FW_PATH_AP := "/vendor/etc/wifi/bcmdhd_apsta.bin"
     ```
   - 檢查 `vendor/samsung/gtelwifi/` 中確實存在 `bcmdhd_apsta.bin`，且編譯時會正確複製到 `/vendor/etc/wifi/`。

4. **核心 NAT 與封包轉發選項**（`kernel/samsung/gtel3g/arch/arm/configs/gtelwifi_defconfig`）：
   - 確保以下核心選項全部開啟（`=y`）：
     ```ini
     CONFIG_IP_ADVANCED_ROUTER=y
     CONFIG_IP_MULTIPLE_TABLES=y
     CONFIG_NETFILTER=y
     CONFIG_NF_NAT=y
     CONFIG_IP_NF_IPTABLES=y
     CONFIG_NF_NAT_IPV4=y   # Linux 3.10 無 CONFIG_IP_NF_NAT，由此選項提供 IPv4 NAT
     CONFIG_IP_NF_TARGET_MASQUERADE=y
     ```

### 第三階段：驗證配置與正式編譯
1. 在 `aosp8` 容器內驗證產品目標是否可順利載入（`vendorsetup.sh` 會自動套用設備樹的 `patches/`）：
   ```bash
   source build/envsetup.sh
   lunch lineage_gtelwifi-userdebug
   ```
2. 開始編譯可刷入的壓縮包，並同步記錄日誌至 `build.log`：
   ```bash
   export USE_CCACHE=1
   export CCACHE_EXEC=/usr/bin/ccache
   make -j6 bacon 2>&1 | tee build.log
   ```
3. 若編譯過程中發生任何錯誤，請讀取 `build.log` 尾端分析原因，修復對應的 Makefile、C/C++ 程式碼、SELinux 規則或依賴缺失，將修改紀錄同步更新至 `Readme_Architecture.md`，然後接續編譯直到成功產出 `out/target/product/gtelwifi/lineage-16.0-*-gtelwifi.zip`。
