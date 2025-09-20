# 開機流程
1. 電源供應器上電，電壓穩定後送出 PWR_OK
2. 主機板/晶片組釋放 CPU 的 RESET，CPU 進入重置狀態
3. CPU 以實模式從 0xFFFFFFF0 取第一條指令（晶片組把 BIOS/UEFI 映射到這裡）
4. BIOS/UEFI 做硬體初始化（POST、記憶體/晶片組/PCIe 等）、挑選開機裝置並載入開機程式
    BIOS：讀取開機碟第 0 個扇區（MBR）到 0x7C00，跳去執行
    UEFI：從 EFI System Partition 載入 .efi 開機程式（例如 bootx64.efi）
5. 開機程式再載入作業系統核心，切換到保護模式/長模式，交棒給核心

# IO機制 (伺服器)
## Kafka: Disk-Level High Throughput Platform
將所有寫入先寫入作業系統的頁面快取（即OS Cache，實際為記憶體操作）。再由OS以順序寫入磁碟，大幅提高寫入效率。
從磁碟讀取資料時，採用DMA達到zero-copy，省去從 OS Cache -> User Space -> Socket Cache 的重複copy，降低CPU與記憶體複製開銷。

# 延伸問題
(1) 0xFFFFFFF0的指令通常是做什麼？為什麼他可以開始執行BIOS？他是做jump嗎？
0xFFFFFFF0是CPU的reset vector，晶片組把主機板上的 SPI Flash（放 BIOS/UEFI）映射到這個位址，所以 CPU 一解開重置就能抓到那裡的指令
通常是一條 FAR JMP（遠跳轉），跳到韌體真正的入口點（仍在同一顆 ROM 裡）

(2) SPI Flash是什麼？

(3) BIOS是韌體？
BIOS（或 UEFI）就是系統韌體，存放在主機板的 SPI Flash 上

(4) 實模式是什麼？
8086 相容模式：16 位元、segment:offset（20 位元實體位址）、無分頁、無保護、幾乎沒有權限機制。可直接跑最早期的 PC 開機碼與 BIOS 介面。

(5) CR0維持預設值，但CR0是做什麼的？CR1又是做什麼的？
CR0：控制保護模式、分頁、快取等大開關。重置後常見狀態是 PE=0、PG=0、CD=1、NW=1、ET=1（也就是實模式、分頁關、快取關）
CR1：在 x86 是保留不用，沒有功能，也不能存取。

(6) 電壓穩定後，是怎麼輸出PWR_OK的？為什麼可以輸出這個訊號？是誰輸出這個訊號？是誰接受這個訊號？
這是 ATX 電源供應器輸出的「電壓穩定」訊號（灰色線）。PSU 監看 3.3/5/12V 都進入容許範圍且穩定後，拉高 PWR_OK。
主機板（PCH/南橋/超級 I/O）接收它，用來決定何時釋放各種重置線，讓 CPU/裝置開始跑。

(7) 硬體初始化算是一個Bootstrap吧？從哪裡開始做Bootstrap？
從 reset vector 取到第一條指令開始就是 bootstrap。那條跳轉會進入 BIOS/UEFI 的主要初始化流程（POST、記憶體控制器、PCIe…），再載入開機程式。

(8) POST是不是只是ping ping看周邊是不是能夠碰到？
不只是探測。它會初始化核心邏輯（時鐘、記憶體控制器、MTRR）、做基本自我測試（CPU/x87、RAM 測試）、設定裝置、發出蜂鳴碼/顯示錯誤等

(9) 主機板/SoC釋放CPU的RESET是什麼意思？CPU為什麼要釋放？為什麼要進入重置？它不是才剛供電嗎？要重置什麼？
RESET 是一條訊號（通常低電位有效）。上電時會先保持「在重置」狀態，讓電壓與時鐘穩定，CPU 內部狀態歸零到已知值（暫存器、快取、管線…）
「釋放」就是把 RESET 解除，CPU 才會開始從 reset vector 執行

(10)RESET訊號的低電位有效是什麼意思？

(11) 是讀取 0xFFFFFFF0 後，jump到BIOS嗎？
0xFFFFFFF0 那裡已經是 BIOS/UEFI ROM 映射區
CPU 取到的就是韌體的指令，而那條指令通常會再 jump 到韌體主入口

(12) 怎麼映射到MBR的？MBR大小多大？
BIOS 會用磁碟 I/O 介面讀「開機裝置」的第 0 扇區到 RAM

(13) 0X7C00是什麼？
傳統 BIOS 會把開機扇區（MBR/Volume Boot Record）載入到實體位址 0x0000:0x7C00（線性 0x7C00），並從那裡開始執行

(14) MBR通常在哪個扇區？為什麼這樣設計？
物理/邏輯的第 0 扇區（CHS 的 0/0/1；LBA 的 0）。固定位置讓早期簡單的 BIOS 只要讀一個已知扇區就能開機

(15) .efi又是什麼？EFI System Partition是什麼東西？bootx64.efi又是什麼？
.efi：UEFI 可執行檔（PE/COFF 格式），由韌體載入到 RAM 執行
ESP（EFI System Partition）：一個 FAT 格式的特殊分割區，放 UEFI 開機程式、驅動、設定。GPT 代碼為 EF00（MBR 類型 0xEF）
bootx64.efi：x86_64 平台上可攜式媒體的預設後備路徑 \EFI\BOOT\BOOTX64.EFI。沒指定其他開機項時，韌體會嘗試載它

(16) PE/COFF格式是什麼？ FAT格式又是什麼？GPT呢？

(17) 怎麼載入作業系統？作業系統放在哪裡？ROM又放在哪裡？記憶體位置在哪？
BIOS：載入 MBR → MBR 讀取更高階的 bootloader → bootloader 從磁碟分割讀 OS 核心到 RAM 指定位址 → 切換模式並跳到核心
UEFI：韌體從 ESP 載入 .efi 開機程式到 RAM → 由它讀取 OS 核心到 RAM → 跳轉
OS 放在儲存裝置的分割區；韌體在主機板的 SPI Flash（記憶體對映到高位址，常會 shadow 到 RAM 加速）；核心載入到它要求/相容的實體位址（由其標頭決定），然後由 bootloader 交棒

(15) 說明PE、PG、CD、NW、ET、NE、WP的意義？是放在CR0嗎？
都是CR0內的bit位
PE（bit0）：Protection Enable，1=進入保護模式
PG（bit31）：Paging，1=啟用分頁（需先備妥頁表並設好 CR3）
CD（bit30）：Cache Disable，1=關閉快取
NW（bit29）：Not Write-through，1=禁止寫通（搭配 CD 控制快取策略）
ET（bit4）：Extension Type，歷史相容位，現代必為 1
NE（bit5）：Numeric Error，1=用 #MF 回報 x87 錯誤（建議開）
WP（bit16）：Write Protect，1=特權態也遵守唯讀頁面的寫保護（建議開）

(16) 為什麼初始化是設定0？為什麼這樣會先跑韌體？
預設 PE=0、PG=0 讓 CPU 以最簡單、相容 8086 的實模式起步，不需要先準備 GDT/頁表
晶片組把 ROM 映射到 CPU 一定能看見的位址（reset vector），所以韌體能立刻被取指並執行

(17) CR3是什麼？
頁表根指標。它存放分頁結構的最高層物理位址（32 位＝頁目錄；PAE/長模式＝PDPT/PML4）。寫 CR3 會觸發 TLB 刷新（啟用 PCID 時可更細緻）

(18) PAE/PDPT/PML4/TLB/PCID 各是什麼？

(19) CR4是什麼？
啟用各種進階機制的開關集合，如 PAE、PSE、PGE、OSFXSR/OSXMMEXCPT（SSE 支援）、OSXSAVE（XSAVE/AVX）、FSGSBASE、PCIDE、SMEP、SMAP、LA57（五級頁表）等

(20) PAE/PSE/PGE/OSFXSR/OSXMMEXCPT(SSE)/OSXSAVE(XSAVE/AVX)/FSGSBASE/PCIDE/SMEP/SMAP/LA57各自是什麼？什麼是五級頁表？哪五級？

(21) x86的CR1有什麼功用？
沒有。保留不用。嘗試用 MOV 存取 CR1 會觸發 #UD（非法指令）

(22) IA32_EFER.LME是什麼？長模式是什麼？
IA32_EFER 是一個 MSR；LME（bit 8）是 Long Mode Enable。流程通常是：設 CR0.PE=1、CR4.PAE=1、備妥 64 位頁表、設 EFER.LME=1，再設 CR0.PG=1 → 長模式啟用（EFER.LMA 會反映已進入
長模式＝x86-64 的 64-bit/相容模式：64 位暫存器與位址（分段幾乎無效，僅 FS/GS 有基底），用 4/5 級頁表，支援 RIP-relative 等新特性

(23) 什麼是MSR ？ RIP-relative是什麼？

(24) ARM的SCTLR是什麼？
ARM 的系統控制暫存器。AArch32 時代叫 SCTLR（CP15 c1），AArch64 是 SCTLR_ELx。它控制 MMU（M 位）、資料/指令快取（C/I 位）、對齊檢查（A）、記憶體執行保護（WXN）、大小端等。功能上有點像 x86 的 CR0+CR4 的集合

(25) AArch32是什麼？CP15 c1是什麼？SCTLR_ELx是什麼？MMU的M位是什麼意思？C/I位是什麼意思？A位是什麼？WXN是什麼？
