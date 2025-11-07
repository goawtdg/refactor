# GitHub 搜索與測試指南

## 一、在 GitHub 上搜索適合測試的項目

### 1.1 搜索策略

#### 策略1：搜索 Fork 關係的項目（最容易找到相似代碼）

**GitHub 搜索語法：**
```
# 搜索特定項目的 fork
fork:true repo:bitcoin/bitcoin

# 搜索特定主題的項目
topic:cryptocurrency language:python

# 搜索特定語言的熱門項目
language:python stars:>1000
```

**推薦搜索：**
- `fork:true repo:bitcoin/bitcoin` - Bitcoin 的 fork 項目（如 Dogecoin）
- `fork:true repo:torvalds/linux` - Linux 內核的 fork
- `fork:true repo:protocolbuffers/protobuf` - Protobuf 的 fork

#### 策略2：搜索相似名稱的項目

```
# 搜索名稱相似的項目
name:json language:c++
name:http language:go
name:driver language:c
```

#### 策略3：搜索特定類型的項目（容易有重複代碼）

**驅動程序項目：**
```
language:c topic:driver stars:>100
```

**網絡庫項目：**
```
language:go topic:network stars:>500
language:c topic:networking stars:>500
```

**加密貨幣項目：**
```
topic:cryptocurrency language:python
topic:blockchain language:go
```

**JSON 庫：**
```
topic:json language:cpp stars:>1000
```

### 1.2 推薦的測試項目類型

基於工具作者測試過的項目，以下類型最容易找到相似代碼：

#### 類型1：Fork 項目（最推薦）
- **Bitcoin/Dogecoin** - 已知有大量相似代碼
- **Linux 內核驅動** - 不同硬件的驅動代碼相似度高
- **Protocol Buffers** - 不同語言的實現

#### 類型2：庫項目（多版本實現）
- **JSON 庫** - 不同語言的 JSON 解析器
- **HTTP 客戶端/服務器** - 不同實現的網絡庫
- **數據庫驅動** - 不同數據庫的驅動代碼

#### 類型3：大型項目（內部重複）
- **操作系統內核** - Linux, FreeBSD 等
- **編譯器** - GCC, Clang 等
- **Web 框架** - Django, Flask 等

## 二、具體測試步驟

### 步驟1：克隆項目

```bash
# 創建測試目錄
mkdir refactor_test
cd refactor_test

# 克隆要測試的項目
git clone https://github.com/bitcoin/bitcoin.git
git clone https://github.com/dogecoin/dogecoin.git

# 或者只測試單個項目
git clone https://github.com/nlohmann/json.git
```

### 步驟2：準備文件列表

```bash
# 方式1：分析特定目錄
cd bitcoin
find . -name "*.cpp" -o -name "*.h" | head -100 > ../bitcoin_files.txt

# 方式2：分析整個項目（可能很慢）
find . -name "*.py" > ../all_python_files.txt

# 方式3：只分析源代碼目錄
find src -name "*.cpp" -o -name "*.h" > ../src_files.txt
```

### 步驟3：運行 refactor

```bash
# 測試單個項目
cat bitcoin_files.txt | python ../refactor/refactor -o bitcoin_analysis.html

# 測試兩個相關項目（Fork 關係）
find bitcoin dogecoin -name "*.cpp" -o -name "*.h" | \
  python ../refactor/refactor --all-files -o bitcoin_dogecoin.html

# 快速測試（只分析大塊代碼）
cat bitcoin_files.txt | python ../refactor/refactor \
  --min-block-size 2000 \
  --similarity 0.85 \
  -o quick_test.html
```

## 三、推薦的測試項目列表

### 3.1 容易找到相似代碼的項目對

#### 加密貨幣項目（Fork 關係）
```bash
# Bitcoin 和其 Fork
git clone https://github.com/bitcoin/bitcoin.git
git clone https://github.com/dogecoin/dogecoin.git
git clone https://github.com/litecoin-project/litecoin.git

# 測試命令
find bitcoin dogecoin -name "*.cpp" -o -name "*.h" | \
  python refactor --all-files -o crypto_comparison.html
```

#### JSON 庫（不同實現）
```bash
git clone https://github.com/nlohmann/json.git
git clone https://github.com/taocpp/json.git

# 測試命令
find nlohmann/json taocpp/json -name "*.hpp" -o -name "*.cpp" | \
  python refactor --all-files -o json_libs.html
```

#### 網絡庫（Go 語言）
```bash
git clone https://github.com/osrg/gobgp.git
git clone https://github.com/cloudflare/bgpkit.git

# 測試命令
find gobgp -name "*.go" | python refactor -o gobgp_analysis.html
```

### 3.2 大型單項目測試

#### Linux 內核驅動
```bash
git clone --depth 1 https://github.com/torvalds/linux.git
cd linux/drivers/net/ethernet

# 測試命令（只測試驅動目錄）
find . -name "*.c" -o -name "*.h" | \
  python ../../../../refactor/refactor -o drivers.html
```

#### Protobuf
```bash
git clone https://github.com/protocolbuffers/protobuf.git

# 測試命令
find protobuf/src -name "*.cc" -o -name "*.h" | \
  python refactor -o protobuf_analysis.html
```

#### ImGUI
```bash
git clone https://github.com/ocornut/imgui.git

# 測試命令
find imgui -name "*.cpp" -o -name "*.h" | \
  python refactor -o imgui_analysis.html
```

### 3.3 Python 項目測試

#### Web 框架
```bash
git clone https://github.com/django/django.git
git clone https://github.com/pallets/flask.git

# 測試命令
find django flask -name "*.py" | \
  python refactor --all-files -o web_frameworks.html
```

#### 數據科學庫
```bash
git clone https://github.com/pandas-dev/pandas.git
git clone https://github.com/numpy/numpy.git

# 測試命令（只測試核心模塊）
find pandas/pandas numpy/numpy -path "*/core/*" -name "*.py" | \
  python refactor --all-files -o data_science.html
```

## 四、GitHub 搜索技巧

### 4.1 使用 GitHub 高級搜索

訪問：https://github.com/search/advanced

**搜索條件示例：**

1. **語言**：Python, C++, Go, C, JavaScript
2. **星標數**：> 1000（熱門項目）
3. **主題標籤**：driver, network, json, crypto
4. **Fork 數**：> 100（活躍的 fork）

### 4.2 使用 GitHub CLI (gh)

```bash
# 安裝 GitHub CLI
# https://cli.github.com/

# 搜索項目
gh search repos --language python --topic json --stars ">1000"

# 搜索 fork
gh search repos --fork bitcoin/bitcoin

# 列出項目的 fork
gh repo list bitcoin/bitcoin --fork
```

### 4.3 使用 GitHub API

```bash
# 搜索相似項目
curl "https://api.github.com/search/repositories?q=language:python+topic:json+stars:>1000" | \
  jq '.items[] | {name: .full_name, stars: .stargazers_count}'
```

## 五、測試腳本

### 5.1 自動化測試腳本

創建 `test_refactor.sh`：

```bash
#!/bin/bash

# 配置
REPO_URL=$1
REPO_NAME=$(basename $REPO_URL .git)
OUTPUT_FILE="${REPO_NAME}_analysis.html"

echo "克隆項目: $REPO_URL"
git clone --depth 1 $REPO_URL

echo "分析項目: $REPO_NAME"
find $REPO_NAME -type f \( -name "*.py" -o -name "*.cpp" -o -name "*.c" -o -name "*.go" \) | \
  python refactor/refactor \
    --min-block-size 1000 \
    --similarity 0.8 \
    -o $OUTPUT_FILE

echo "報告已生成: $OUTPUT_FILE"
echo "清理臨時文件..."
rm -rf $REPO_NAME

echo "完成！"
```

**使用方法：**
```bash
chmod +x test_refactor.sh
./test_refactor.sh https://github.com/nlohmann/json.git
```

### 5.2 批量測試腳本

創建 `batch_test.sh`：

```bash
#!/bin/bash

REPOS=(
  "https://github.com/nlohmann/json.git"
  "https://github.com/ocornut/imgui.git"
  "https://github.com/osrg/gobgp.git"
)

for repo in "${REPOS[@]}"; do
  echo "測試: $repo"
  ./test_refactor.sh $repo
  echo "---"
done
```

## 六、評估測試結果

### 6.1 檢查報告質量

打開生成的 HTML 報告，檢查：

1. **相似代碼塊數量**：
   - 大型項目（如 Linux 驅動）：應該找到 100+ 相似塊
   - 中型項目（如 JSON 庫）：應該找到 50-200 相似塊
   - 小型項目：應該找到 10-50 相似塊

2. **相似度準確性**：
   - 檢查報告中的代碼塊是否真的相似
   - 是否有誤報（不相似的代碼被標記為相似）

3. **性能**：
   - 記錄處理時間
   - 檢查內存使用

### 6.2 與已知結果對比

參考 `examples/` 目錄中的示例報告：
- `drivers.html` - Linux 驅動（~400 相似塊）
- `json.html` - JSON 庫（~350 相似塊）
- `crypto.html` - Bitcoin/Dogecoin（~270 相似塊）
- `gobgp.html` - Go BGP（~250 相似塊）
- `protobuf.html` - Protobuf（~215 相似塊）
- `imgui.html` - ImGUI（~30 相似塊）

### 6.3 測試不同參數組合

```bash
# 測試1：默認參數
python refactor -o test1.html file1.py

# 測試2：降低相似度
python refactor --similarity 0.7 -o test2.html file1.py

# 測試3：降低最小塊大小
python refactor --min-block-size 800 -o test3.html file1.py

# 測試4：跨文件比較
python refactor --all-files -o test4.html file1.py file2.py
```

## 七、快速測試命令

### 7.1 一鍵測試（推薦項目）

```bash
# 測試 Bitcoin/Dogecoin（已知有大量相似代碼）
git clone --depth 1 https://github.com/bitcoin/bitcoin.git bitcoin_test
git clone --depth 1 https://github.com/dogecoin/dogecoin.git dogecoin_test
find bitcoin_test dogecoin_test -name "*.cpp" -o -name "*.h" | \
  head -200 | \
  python refactor/refactor --all-files -o bitcoin_dogecoin_test.html
rm -rf bitcoin_test dogecoin_test
```

### 7.2 測試小型項目（快速驗證）

```bash
# 測試 ImGUI（較小的項目）
git clone --depth 1 https://github.com/ocornut/imgui.git
find imgui -name "*.cpp" -o -name "*.h" | \
  python refactor/refactor -o imgui_test.html
rm -rf imgui
```

## 八、注意事項

1. **處理時間**：大型項目可能需要很長時間，建議先用 `-p` 查看會識別多少代碼塊
2. **磁盤空間**：克隆大型項目需要足夠的磁盤空間
3. **網絡**：確保網絡連接穩定，可以快速克隆項目
4. **臨時文件**：工具會在 `/tmp` 創建臨時文件，確保有足夠空間

## 九、推薦的測試流程

1. **小規模測試**：先用小型項目（如 ImGUI）驗證工具正常工作
2. **中等規模**：測試中型項目（如 JSON 庫）
3. **大規模測試**：測試大型項目（如 Linux 驅動、Bitcoin）
4. **對比測試**：測試 Fork 關係的項目（如 Bitcoin/Dogecoin）

## 十、GitHub 搜索示例

### 搜索 Fork 項目
```
site:github.com fork bitcoin
site:github.com fork:true repo:torvalds/linux
```

### 搜索特定主題
```
site:github.com language:python topic:json stars:>1000
site:github.com language:c topic:driver stars:>500
```

### 搜索相似名稱
```
site:github.com "json library" language:cpp
site:github.com "http client" language:go
```

