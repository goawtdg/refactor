# 優化方向詳細指南

## 一、性能優化（高影響）

### 1.1 算法複雜度優化

#### 問題：O(n²) 雙重循環比較

**當前代碼（第385-421行）：**
```python
for i in range(0, L):
    for j in range(i+1, L):
        # 比較每個代碼塊對
```

**問題：**
- 對於 1000 個代碼塊，需要比較 499,500 次
- 每次比較都調用 `SequenceMatcher`，計算成本高

**優化方案1：早期過濾優化**
```python
# 在比較前先建立索引，快速過濾
# 按長度分組，只比較長度相近的塊
length_groups = {}
for b in blist:
    length = len(b.code)
    key = length // args.max_block_diff  # 分組鍵
    if key not in length_groups:
        length_groups[key] = []
    length_groups[key].append(b)

# 只在同組內比較
for group in length_groups.values():
    for i in range(len(group)):
        for j in range(i+1, len(group)):
            # 比較邏輯
```

**優化方案2：使用哈希預過濾**
```python
# 使用代碼塊的哈希值快速過濾明顯不同的塊
from hashlib import md5

def get_code_hash(code):
    # 移除空白字符後計算哈希
    normalized = ''.join(code.split())
    return md5(normalized.encode()).hexdigest()

# 只比較哈希值相似的塊
```

**預期提升：** 減少 50-80% 的比較次數

#### 問題：重複計算 SequenceMatcher

**當前代碼（第409-411行）：**
```python
sm = SM(None, bi.code, bj.code)
if sm.quick_ratio() < args.similarity: continue
if sm.ratio()       < args.similarity: continue
```

**問題：**
- `quick_ratio()` 和 `ratio()` 都計算相似度，但 `ratio()` 更準確但更慢
- 如果 `quick_ratio()` 已經低於閾值，不需要再計算 `ratio()`

**優化方案：**
```python
sm = SM(None, bi.code, bj.code)
quick_ratio = sm.quick_ratio()
if quick_ratio < args.similarity: continue

# 只有 quick_ratio 接近閾值時才計算精確的 ratio
if quick_ratio < args.similarity * 0.95:  # 緩衝區
    continue

ratio = sm.ratio()
if ratio < args.similarity: continue
```

**預期提升：** 減少 30-50% 的計算時間

### 1.2 數據結構優化

#### 問題：skip_children 中使用 list.index()

**當前代碼（第336-343行）：**
```python
def skip_children(b):
    global skip
    for ch in b.children:
      if ch in blist:
        skip += [blist.index(ch)]  # O(n) 操作
```

**問題：**
- `blist.index(ch)` 是 O(n) 操作
- 在循環中調用，總複雜度為 O(n²)

**優化方案：**
```python
# 建立 block 到索引的映射
block_to_index = {b: i for i, b in enumerate(blist)}

def skip_children(b):
    global skip
    for ch in b.children:
        if ch in block_to_index:
            skip.append(block_to_index[ch])
            skip_children(ch)
```

**預期提升：** 將 O(n²) 降為 O(n)

#### 問題：ancestor 函數重複計算

**當前代碼（第345-352行）：**
```python
def ancestor(a, b):
    p = a.parent
    while p:
       if b == p: return 1
       p = p.parent
    return 0
```

**問題：**
- 在雙重循環中重複調用
- 可以預先計算所有祖先關係

**優化方案：**
```python
# 預先建立祖先關係緩存
ancestor_cache = {}

def is_ancestor(a, b):
    key = (id(a), id(b))
    if key in ancestor_cache:
        return ancestor_cache[key]
    
    p = a.parent
    while p:
        if b == p:
            ancestor_cache[key] = True
            return True
        p = p.parent
    ancestor_cache[key] = False
    return False
```

**預期提升：** 減少 70-90% 的重複計算

### 1.3 字符串操作優化

#### 問題：字符串拼接效率低

**當前代碼（第89-92行）：**
```python
def add_code(self, c):
    self.code += c  # 字符串拼接，每次創建新對象
    if self.parent: self.parent.add_code(c)
```

**問題：**
- Python 字符串是不可變的，`+=` 操作會創建新字符串對象
- 對於大代碼塊，會產生大量臨時對象

**優化方案：**
```python
class block:
    def __init__(self, ...):
        self.code_parts = []  # 使用列表
    
    def add_code(self, c):
        self.code_parts.append(c)
        if self.parent:
            self.parent.add_code(c)
    
    @property
    def code(self):
        return ''.join(self.code_parts)  # 只在需要時拼接
```

**預期提升：** 減少 30-50% 的內存分配

### 1.4 並行處理優化

#### 問題：單線程處理

**當前狀態：** 所有比較都在單線程中順序執行

**優化方案：使用多進程並行比較**
```python
from multiprocessing import Pool, cpu_count
from functools import partial

def compare_blocks(args_tuple):
    i, j, bi, bj, config = args_tuple
    # 比較邏輯
    if should_skip(bi, bj, config):
        return None
    sm = SM(None, bi.code, bj.code)
    if sm.ratio() >= config.similarity:
        return (bi, bj)
    return None

# 生成所有需要比較的對
comparisons = [(i, j, blist[i], blist[j], config) 
               for i in range(L) for j in range(i+1, L)]

# 並行處理
with Pool(processes=cpu_count()) as pool:
    results = pool.map(compare_blocks, comparisons)

similar = [r for r in results if r is not None]
```

**預期提升：** 在多核 CPU 上可獲得 4-8 倍加速

## 二、跨平台兼容性優化

### 2.1 替換 os.system() 調用

**當前問題：** 使用 Unix 命令，Windows 不兼容

**優化方案：**
```python
import tempfile
import shutil
import subprocess
from pathlib import Path

# 1. 使用 tempfile 模塊
root = Path(tempfile.gettempdir()) / f'refactor-{os.getpid()}'
root.mkdir(parents=True, exist_ok=True)
(root / 'diffs').mkdir(exist_ok=True)
(root / 'files').mkdir(exist_ok=True)

# 2. 使用 Python 實現 diff（或跨平台調用）
def generate_diff(file1, file2, width):
    """跨平台的 diff 生成"""
    try:
        # 嘗試使用系統 diff 命令
        result = subprocess.run(
            ['diff', '-t', '-y', f'--width={width}', str(file1), str(file2)],
            capture_output=True,
            text=True,
            check=False
        )
        return result.stdout
    except FileNotFoundError:
        # 如果沒有 diff 命令，使用 Python difflib
        with open(file1) as f1, open(file2) as f2:
            from difflib import unified_diff
            return ''.join(unified_diff(
                f1.readlines(), f2.readlines(),
                fromfile=str(file1), tofile=str(file2),
                lineterm=''
            ))

# 3. 使用 shutil 和 Path 操作
def merge_html_files(output_file, html_files):
    """合併 HTML 文件"""
    with open(output_file, 'w', encoding='utf-8') as out:
        for html_file in sorted(html_files):
            with open(html_file, 'r', encoding='utf-8') as f:
                out.write(f.read())

# 4. 清理臨時文件
if not args.no_cleanup:
    shutil.rmtree(root, ignore_errors=True)
```

### 2.2 路徑處理標準化

**優化方案：**
```python
import os.path
from pathlib import Path

def dash(fname):
    """跨平台的文件名轉換"""
    # 使用 Path 處理路徑
    path = Path(fname)
    # 替換所有路徑分隔符
    return str(path).replace(os.sep, '-').replace('/', '-').replace('\\', '-')
```

## 三、代碼結構優化

### 3.1 減少全局變量

**當前問題：** 大量全局變量導致狀態管理混亂

**優化方案：使用類封裝**
```python
class CodeParser:
    def __init__(self, args):
        self.args = args
        self.blocks = []
        self.stack = []
        self.state = code
        self.indent = 0
    
    def process_file(self, filename):
        # 重置狀態
        self.stack = []
        self.state = code
        self.indent = 0
        
        if filename.endswith('.py') or self.args.python:
            self.process_python_file(filename)
        else:
            self.process_c_file(filename)
    
    # ... 其他方法

# 使用
parser = CodeParser(args)
for fn in inputs:
    parser.process_file(fn)
blocks = parser.blocks
```

### 3.2 函數職責分離

**優化方案：**
```python
class BlockComparator:
    def __init__(self, blocks, config):
        self.blocks = blocks
        self.config = config
        self.similar = []
    
    def find_similar(self):
        # 比較邏輯
        pass

class HTMLReporter:
    def __init__(self, similar_blocks, output_file):
        self.similar_blocks = similar_blocks
        self.output_file = output_file
    
    def generate_report(self):
        # HTML 生成邏輯
        pass
```

## 四、內存優化

### 4.1 流式處理大文件

**當前問題：** 整個文件讀入內存

**優化方案：**
```python
def process_cfile_streaming(fn, f):
    """流式處理，不將整個文件讀入內存"""
    # 只讀取需要的部分
    # 使用生成器而不是列表
    pass
```

### 4.2 及時釋放不需要的數據

**優化方案：**
```python
# 在比較完成後，釋放代碼內容
for b in blocks:
    if b not in matched_blocks:
        b.code = None  # 釋放內存
```

## 五、錯誤處理優化

### 5.1 修復已知錯誤

**修復 IndexError（第108行）：**
```python
if not line.rstrip() or (len(line) > indent and line[indent] in ['#','"',"'"]):
```

**修復字符串迭代（第303行）：**
```python
print(b.code, end="")  # 直接打印字符串
# 或
for line in b.code.split('\n'):
    print(line)
```

### 5.2 添加異常處理

**優化方案：**
```python
def safe_process_file(fn):
    try:
        with open(fn, 'r', encoding='utf-8', errors='ignore') as f:
            if fn.endswith('.py') or args.python:
                process_pfile(fn, f)
            else:
                process_cfile(fn, f)
    except FileNotFoundError:
        print(f'[!] File not found: {fn}', file=sys.stderr)
    except PermissionError:
        print(f'[!] Permission denied: {fn}', file=sys.stderr)
    except Exception as e:
        print(f'[!] Error processing {fn}: {e}', file=sys.stderr)
```

## 六、功能增強優化

### 6.1 進度顯示改進

**優化方案：使用 tqdm**
```python
from tqdm import tqdm

# 文件處理進度
for fn in tqdm(inputs, desc="Processing files"):
    process_file(fn)

# 比較進度
total = L * (L - 1) // 2
with tqdm(total=total, desc="Comparing blocks") as pbar:
    for i in range(L):
        for j in range(i+1, L):
            # 比較邏輯
            pbar.update(1)
```

### 6.2 輸出格式多樣化

**優化方案：**
```python
class ReportGenerator:
    def generate_html(self, similar_blocks):
        # HTML 報告
        pass
    
    def generate_json(self, similar_blocks):
        # JSON 報告
        import json
        return json.dumps([
            {
                'file1': b1.file,
                'file2': b2.file,
                'range1': b1.range,
                'range2': b2.range,
                'similarity': similarity
            }
            for b1, b2, similarity in similar_blocks
        ], indent=2)
    
    def generate_terminal(self, similar_blocks):
        # 終端彩色輸出
        from colorama import init, Fore
        init()
        # ...
```

## 七、優化優先級建議

### 高優先級（立即實施）
1. ✅ **修復邏輯錯誤**（IndexError、字符串迭代）
2. ✅ **跨平台兼容性**（替換 os.system）
3. ✅ **skip_children 優化**（使用字典而非 index）
4. ✅ **早期過濾**（按長度分組）

### 中優先級（短期實施）
1. **SequenceMatcher 優化**（避免重複計算）
2. **ancestor 緩存**（預先計算）
3. **字符串拼接優化**（使用列表）
4. **錯誤處理改進**

### 低優先級（長期優化）
1. **並行處理**（多進程）
2. **代碼重構**（類封裝）
3. **功能增強**（JSON 輸出、進度條）

## 八、性能基準測試

### 測試腳本
```python
import time
import cProfile

def benchmark():
    start = time.time()
    
    # 運行 refactor
    # ...
    
    elapsed = time.time() - start
    print(f"處理時間: {elapsed:.2f} 秒")
    print(f"處理速度: {len(blocks) / elapsed:.2f} 塊/秒")

# 使用 cProfile 分析
cProfile.run('benchmark()')
```

### 預期性能提升

| 優化項目 | 預期提升 | 實施難度 |
|---------|---------|---------|
| 早期過濾 | 50-80% | 低 |
| skip_children 優化 | 10-20% | 低 |
| ancestor 緩存 | 5-10% | 中 |
| 並行處理 | 300-700% | 高 |
| 字符串優化 | 20-30% | 中 |
| SequenceMatcher 優化 | 30-50% | 低 |

**總體預期：** 綜合優化後，性能可提升 **2-5 倍**（單線程）或 **10-20 倍**（多線程）

## 九、實施建議

### 階段1：穩定性修復（1-2天）
- 修復所有邏輯錯誤
- 實現跨平台兼容
- 添加基本錯誤處理

### 階段2：性能優化（3-5天）
- 實施早期過濾
- 優化數據結構
- 改進算法效率

### 階段3：架構重構（1-2週）
- 代碼重構為類結構
- 實現並行處理
- 添加測試覆蓋

### 階段4：功能增強（持續）
- 多種輸出格式
- 改進用戶體驗
- 擴展語言支持

