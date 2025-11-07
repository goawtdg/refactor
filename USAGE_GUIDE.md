# 使用指南：如何識別相似代碼塊

## 快速開始

### 1. 基本使用 - 分析單個或多個文件

```bash
# 分析單個文件
python refactor file1.py

# 分析多個文件
python refactor file1.py file2.py file3.c

# 分析整個目錄（使用 find 命令）
find . -name "*.py" | python refactor
```

### 2. 從標準輸入讀取文件列表

```bash
# 方式1：使用 find 命令
find . -type f -name "*.py" | python refactor

# 方式2：使用 grep 過濾
find . -type f | grep -E '\.(py|cpp|h)$' | python refactor

# 方式3：手動輸入文件列表
echo -e "file1.py\nfile2.py\nfile3.py" | python refactor
```

### 3. 自動過濾文件類型

使用 `-f` 選項自動過濾支持的編程語言文件：

```bash
# 從 stdin 讀取並自動過濾（只處理支持的語言）
find . -type f | python refactor -f
```

**支持的語言**：`c, cpp, h, hh, hpp, py, go, rs, java, js, swift, php`

## 常用參數說明

### 調整相似度閾值

```bash
# 降低相似度要求（找到更多相似塊，默認 0.8）
python refactor --similarity 0.7 file1.py file2.py

# 提高相似度要求（只找非常相似的塊）
python refactor --similarity 0.9 file1.py file2.py
```

### 調整最小代碼塊大小

```bash
# 降低最小塊大小（分析更小的代碼塊，默認 1500 字符）
python refactor --min-block-size 500 file1.py

# 提高最小塊大小（只分析大塊代碼，加快速度）
python refactor --min-block-size 3000 file1.py
```

### 跨文件比較

```bash
# 比較所有文件（即使文件名差異很大）
python refactor --all-files file1.py file2.c file3.go

# 默認只比較文件名相似的文件（如 udp.c 和 udp6.c）
```

### 跨縮進層級比較

```bash
# 比較不同縮進層級的代碼塊（默認只比較同一層級）
python refactor --all-indents file1.py
```

### 指定輸出文件

```bash
# 自定義輸出文件名
python refactor -o my_report.html file1.py file2.py
```

### 調試模式

```bash
# 顯示解析過程的調試信息
python refactor -d file1.py

# 只解析，不比較（查看識別到的代碼塊）
python refactor -p file1.py

# 顯示所有識別到的代碼塊內容
python refactor -b file1.py
```

## 實際使用場景

### 場景1：分析單個 Python 項目

```bash
# 分析項目中所有 Python 文件
find /path/to/project -name "*.py" | python refactor -o project_analysis.html
```

### 場景2：找出複製粘貼但未完全更新的代碼

```bash
# 使用較低的相似度閾值，找出可能未完全更新的代碼
find . -name "*.py" | python refactor --similarity 0.75 --min-block-size 800
```

### 場景3：比較兩個相關項目

```bash
# 比較兩個項目的相似代碼（如 fork 的項目）
find project1 project2 -name "*.py" | python refactor --all-files -o comparison.html
```

### 場景4：分析大型代碼庫

```bash
# 只分析大塊代碼，加快處理速度
find . -name "*.cpp" | python refactor --min-block-size 2000 --max-block-diff 1000
```

### 場景5：分析特定目錄

```bash
# 只分析 src 目錄下的文件
find src -type f | python refactor -f -o src_analysis.html
```

## 查看結果

### HTML 報告

運行完成後，會生成一個 HTML 報告文件：

```bash
# 默認文件名：refactor-<進程ID>.html
# 或使用 -o 指定文件名
python refactor -o report.html file1.py

# 在瀏覽器中打開
# Linux/Mac:
open report.html
# 或
xdg-open report.html

# Windows:
start report.html
```

### HTML 報告內容

報告包含：
- **側邊對比視圖**：左右對比顯示相似的代碼塊
- **文件位置**：顯示每個代碼塊所在的文件和行號
- **差異高亮**：
  - 綠色背景：新增的代碼
  - 紅色背景：刪除的代碼
  - 黃色分隔符：有差異的行

### 使用調試選項查看解析過程

```bash
# 查看識別到的代碼塊數量
python refactor -p file1.py
# 輸出：123 blocks

# 查看所有代碼塊的詳細內容
python refactor -b file1.py
# 會顯示每個代碼塊的文件位置、縮進層級和完整代碼
```

## 參數組合示例

### 快速掃描（適合大型項目）

```bash
find . -name "*.py" | python refactor \
  --min-block-size 2000 \
  --max-block-diff 800 \
  --similarity 0.85 \
  -o quick_scan.html
```

### 深度分析（找出所有相似代碼）

```bash
find . -name "*.py" | python refactor \
  --min-block-size 500 \
  --max-block-diff 200 \
  --similarity 0.7 \
  --all-files \
  --all-indents \
  -o deep_analysis.html
```

### 只分析同一文件內的相似代碼

```bash
# 不使用 --all-files，默認只比較文件名相似的文件
# 同一文件內的代碼塊會被比較
python refactor file1.py file2.py -o same_file_analysis.html
```

## 工作流程建議

### 步驟1：快速預覽

```bash
# 先用默認參數快速掃描
find . -name "*.py" | python refactor -o preview.html
```

### 步驟2：根據結果調整參數

如果找到太多結果：
- 提高 `--min-block-size`
- 提高 `--similarity`

如果找到太少結果：
- 降低 `--min-block-size`
- 降低 `--similarity`
- 使用 `--all-files` 和 `--all-indents`

### 步驟3：深入分析

```bash
# 針對特定目錄或文件類型進行深度分析
find src/utils -name "*.py" | python refactor \
  --similarity 0.75 \
  --min-block-size 800 \
  -o utils_analysis.html
```

## 注意事項

1. **處理時間**：大型項目可能需要較長時間，建議先用 `-p` 查看會識別多少代碼塊

2. **內存使用**：處理大量文件時可能消耗較多內存

3. **臨時文件**：工具會在 `/tmp` 目錄創建臨時文件（Windows 上可能失敗）

4. **文件編碼**：確保文件是 UTF-8 編碼，否則可能跳過

5. **Windows 兼容性**：目前工具主要針對 Unix/Linux 設計，Windows 上可能需要修改

## 輸出示例

運行成功後，終端會顯示：

```
[|] processing file1.py
[/] processing file2.py
[-] comparing blocks 1234 - 5678
[\] post processing
42 similar blocks found
> report.html
```

然後打開 `report.html` 查看詳細的相似代碼對比報告。

