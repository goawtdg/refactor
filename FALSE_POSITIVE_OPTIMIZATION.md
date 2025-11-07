# 減少誤報（False Positive）優化方案

## 一、當前問題分析

### 1.1 誤報的主要原因

#### 問題1：只使用字符級別相似度
**當前邏輯（第409-411行）：**
```python
sm = SM(None, bi.code, bj.code)
if sm.quick_ratio() < args.similarity: continue
if sm.ratio()       < args.similarity: continue
```

**問題：**
- `SequenceMatcher` 只比較字符序列，不考慮代碼語義
- 兩個功能完全不同的函數，如果結構相似（如都有 if-else、循環），可能被誤判為相似
- 空白字符、註釋、變量名的差異會影響判斷

#### 問題2：沒有標準化處理
**問題：**
- 空白字符（空格、製表符、換行）的差異會影響相似度
- 註釋內容不同但代碼邏輯相同的情況會被誤判
- 變量名不同但邏輯相同的代碼會被誤判

#### 問題3：缺乏結構化分析
**問題：**
- 沒有分析代碼的結構特徵（如函數簽名、控制流結構）
- 沒有考慮代碼的實際差異程度（可能相似度高但差異很大）

#### 問題4：閾值設置不合理
**問題：**
- 默認 0.8 的相似度閾值可能過低
- 沒有考慮代碼塊大小的影響（小塊代碼更容易誤報）

## 二、優化方案

### 2.1 代碼標準化處理

#### 方案1：移除空白和註釋
```python
import re

def normalize_code(code):
    """標準化代碼，移除不影響邏輯的部分"""
    # 1. 移除單行註釋
    code = re.sub(r'//.*?$', '', code, flags=re.MULTILINE)
    code = re.sub(r'#.*?$', '', code, flags=re.MULTILINE)
    
    # 2. 移除多行註釋
    code = re.sub(r'/\*.*?\*/', '', code, flags=re.DOTALL)
    
    # 3. 標準化空白字符
    code = re.sub(r'\s+', ' ', code)  # 多個空白變為單個空格
    code = code.strip()
    
    return code

# 在比較時使用標準化後的代碼
def are_similar(bi, bj, args):
    # 標準化代碼
    code1 = normalize_code(bi.code)
    code2 = normalize_code(bj.code)
    
    # 比較標準化後的代碼
    sm = SM(None, code1, code2)
    return sm.ratio() >= args.similarity
```

#### 方案2：提取代碼結構
```python
def extract_structure(code):
    """提取代碼的結構特徵（控制流、關鍵字等）"""
    # 提取關鍵字和操作符
    keywords = re.findall(r'\b(if|else|for|while|switch|case|return|break|continue|try|except|def|class)\b', code)
    operators = re.findall(r'[+\-*/%=<>!&|]+', code)
    brackets = re.findall(r'[{}()\[\]]', code)
    
    # 返回結構特徵
    return {
        'keywords': ' '.join(keywords),
        'operators': ' '.join(operators),
        'brackets': ' '.join(brackets),
        'line_count': len(code.split('\n'))
    }

def structure_similarity(bi, bj):
    """比較代碼結構相似度"""
    struct1 = extract_structure(bi.code)
    struct2 = extract_structure(bj.code)
    
    # 比較關鍵字序列
    sm_keywords = SM(None, struct1['keywords'], struct2['keywords'])
    sm_operators = SM(None, struct1['operators'], struct2['operators'])
    
    # 綜合結構相似度
    keyword_sim = sm_keywords.ratio()
    operator_sim = sm_operators.ratio()
    line_diff = abs(struct1['line_count'] - struct2['line_count']) / max(struct1['line_count'], struct2['line_count'], 1)
    
    # 結構相似度 = 關鍵字相似度 * 0.6 + 操作符相似度 * 0.4 - 行數差異懲罰
    structure_sim = keyword_sim * 0.6 + operator_sim * 0.4 - line_diff * 0.2
    
    return max(0, structure_sim)
```

### 2.2 多指標綜合判斷

#### 方案：使用多個相似度指標
```python
def comprehensive_similarity(bi, bj, args):
    """綜合多個指標判斷相似度"""
    
    # 1. 原始代碼相似度
    sm_raw = SM(None, bi.code, bj.code)
    raw_similarity = sm_raw.ratio()
    
    # 2. 標準化代碼相似度（移除空白、註釋）
    code1_norm = normalize_code(bi.code)
    code2_norm = normalize_code(bj.code)
    sm_norm = SM(None, code1_norm, code2_norm)
    norm_similarity = sm_norm.ratio()
    
    # 3. 結構相似度
    struct_similarity = structure_similarity(bi, bj)
    
    # 4. 行數相似度
    lines1 = len(bi.code.split('\n'))
    lines2 = len(bj.code.split('\n'))
    line_similarity = 1 - abs(lines1 - lines2) / max(lines1, lines2, 1)
    
    # 5. 差異程度分析
    diff_ratio = calculate_diff_ratio(bi.code, bj.code)
    
    # 綜合評分（加權平均）
    # 標準化相似度權重最高，因為它最能反映邏輯相似性
    final_score = (
        norm_similarity * 0.5 +      # 標準化代碼相似度（最重要）
        struct_similarity * 0.25 +   # 結構相似度
        raw_similarity * 0.15 +      # 原始相似度
        line_similarity * 0.1        # 行數相似度
    )
    
    # 如果差異太大，降低分數
    if diff_ratio > 0.3:  # 差異超過30%
        final_score *= 0.8
    
    return final_score >= args.similarity
```

### 2.3 差異程度分析

#### 方案：計算實際差異比例
```python
def calculate_diff_ratio(code1, code2):
    """計算兩個代碼塊的實際差異比例"""
    from difflib import unified_diff
    
    lines1 = code1.split('\n')
    lines2 = code2.split('\n')
    
    # 使用 unified_diff 計算差異
    diff = list(unified_diff(lines1, lines2, lineterm=''))
    
    # 統計差異行數
    diff_lines = sum(1 for line in diff if line.startswith(('+', '-')))
    total_lines = max(len(lines1), len(lines2))
    
    if total_lines == 0:
        return 0.0
    
    return diff_lines / total_lines

# 在比較時添加差異閾值檢查
def are_similar_with_diff_check(bi, bj, args):
    # 計算相似度
    sm = SM(None, normalize_code(bi.code), normalize_code(bj.code))
    similarity = sm.ratio()
    
    # 計算差異比例
    diff_ratio = calculate_diff_ratio(bi.code, bj.code)
    
    # 如果相似度高但差異也很大，可能是誤報
    if similarity >= args.similarity:
        # 差異閾值：相似度越高，允許的差異越小
        max_diff = 1.0 - similarity  # 如果相似度0.8，最大差異0.2
        if diff_ratio > max_diff * 1.5:  # 允許一些緩衝
            return False
    
    return similarity >= args.similarity
```

### 2.4 代碼塊大小相關的閾值調整

#### 方案：根據代碼塊大小動態調整閾值
```python
def adaptive_similarity_threshold(bi, bj, base_threshold):
    """根據代碼塊大小動態調整相似度閾值"""
    
    size1 = len(bi.code)
    size2 = len(bj.code)
    avg_size = (size1 + size2) / 2
    
    # 小代碼塊更容易誤報，提高閾值
    if avg_size < 500:
        return base_threshold + 0.1  # 提高10%
    elif avg_size < 1000:
        return base_threshold + 0.05  # 提高5%
    elif avg_size > 5000:
        return base_threshold - 0.05  # 大塊代碼可以稍微降低閾值
    else:
        return base_threshold
```

### 2.5 語義相似度分析

#### 方案：分析變量名和函數名的模式
```python
def semantic_similarity(bi, bj):
    """分析代碼的語義相似度"""
    
    # 提取變量名和函數名
    def extract_identifiers(code):
        # 提取標識符（變量名、函數名等）
        identifiers = re.findall(r'\b[a-zA-Z_][a-zA-Z0-9_]*\b', code)
        # 過濾關鍵字
        keywords = {'if', 'else', 'for', 'while', 'return', 'def', 'class', 'import', 'from'}
        return [id for id in identifiers if id not in keywords]
    
    ids1 = set(extract_identifiers(bi.code))
    ids2 = set(extract_identifiers(bj.code))
    
    if not ids1 and not ids2:
        return 1.0  # 都沒有標識符，無法判斷
    
    # 計算標識符重疊度
    if ids1 and ids2:
        overlap = len(ids1 & ids2) / len(ids1 | ids2)
        return overlap
    else:
        return 0.0  # 一個有標識符一個沒有，可能不相似

# 在綜合判斷中加入語義相似度
def final_similarity_check(bi, bj, args):
    # ... 其他相似度計算 ...
    
    semantic_sim = semantic_similarity(bi, bj)
    
    # 如果語義相似度很低，即使結構相似也可能是誤報
    if semantic_sim < 0.3 and norm_similarity > 0.8:
        # 結構相似但語義不同，可能是誤報
        return False
    
    return final_score >= args.similarity
```

### 2.6 上下文分析

#### 方案：考慮代碼塊的上下文
```python
def context_similarity(bi, bj):
    """分析代碼塊的上下文相似度"""
    
    # 獲取父塊的代碼（上下文）
    def get_context(block, depth=1):
        context = []
        parent = block.parent
        for _ in range(depth):
            if parent:
                context.append(parent.code[:200])  # 只取前200字符
                parent = parent.parent
            else:
                break
        return ' '.join(context)
    
    context1 = get_context(bi)
    context2 = get_context(bj)
    
    if context1 and context2:
        sm_context = SM(None, context1, context2)
        return sm_context.ratio()
    else:
        return 0.5  # 沒有上下文，給中性分數
```

## 三、實施建議

### 3.1 改進後的比較函數

```python
def are_blocks_similar(bi, bj, args):
    """
    改進的相似度判斷函數，減少誤報
    """
    
    # 1. 快速過濾：長度差異太大
    if abs(len(bi.code) - len(bj.code)) > args.max_block_diff:
        return False, 0.0
    
    # 2. 標準化代碼
    code1_norm = normalize_code(bi.code)
    code2_norm = normalize_code(bj.code)
    
    # 3. 計算標準化相似度
    sm_norm = SM(None, code1_norm, code2_norm)
    norm_sim = sm_norm.ratio()
    
    # 4. 如果標準化相似度太低，直接返回
    if norm_sim < args.similarity * 0.9:  # 預留10%緩衝
        return False, norm_sim
    
    # 5. 計算結構相似度
    struct_sim = structure_similarity(bi, bj)
    
    # 6. 計算語義相似度
    semantic_sim = semantic_similarity(bi, bj)
    
    # 7. 計算差異比例
    diff_ratio = calculate_diff_ratio(bi.code, bj.code)
    
    # 8. 動態調整閾值
    threshold = adaptive_similarity_threshold(bi, bj, args.similarity)
    
    # 9. 綜合評分
    final_score = (
        norm_sim * 0.5 +
        struct_sim * 0.25 +
        semantic_sim * 0.15 +
        (1 - diff_ratio) * 0.1
    )
    
    # 10. 差異檢查：如果差異太大，降低分數
    if diff_ratio > 0.3:
        final_score *= 0.85
    
    # 11. 語義檢查：如果語義完全不同，可能是誤報
    if semantic_sim < 0.2 and norm_sim > 0.85:
        # 結構相似但語義完全不同，可能是誤報
        return False, final_score
    
    # 12. 最終判斷
    is_similar = final_score >= threshold
    
    return is_similar, final_score
```

### 3.2 在主循環中使用

```python
# 替換原來的比較邏輯（第408-411行）
# 原代碼：
# sm = SM(None, bi.code, bj.code)
# if sm.quick_ratio() < args.similarity: continue
# if sm.ratio()       < args.similarity: continue

# 新代碼：
is_similar, similarity_score = are_blocks_similar(bi, bj, args)
if not is_similar:
    continue

# 可選：記錄相似度分數用於排序
similar.append((bi, bj, similarity_score))
```

## 四、參數調整建議

### 4.1 新增參數

```python
parser.add_argument('--min-semantic-similarity', default=0.2, type=float,
                   help='Minimum semantic similarity threshold (default: 0.2)')
parser.add_argument('--max-diff-ratio', default=0.3, type=float,
                   help='Maximum difference ratio (default: 0.3)')
parser.add_argument('--use-normalization', action='store_true', default=True,
                   help='Normalize code before comparison (remove whitespace, comments)')
parser.add_argument('--strict-mode', action='store_true',
                   help='Use stricter similarity checks to reduce false positives')
```

### 4.2 嚴格模式

```python
if args.strict_mode:
    # 嚴格模式：使用更高的閾值和更多的檢查
    args.similarity = max(args.similarity, 0.85)  # 至少85%
    args.min_semantic_similarity = 0.3  # 提高語義相似度要求
    args.max_diff_ratio = 0.2  # 降低允許的差異比例
```

## 五、測試和驗證

### 5.1 測試用例

```python
# 測試1：相同邏輯但變量名不同（應該相似）
code1 = """
def calculate_sum(a, b):
    result = a + b
    return result
"""

code2 = """
def calculate_sum(x, y):
    total = x + y
    return total
"""
# 期望：相似（標準化後相同）

# 測試2：不同邏輯但結構相似（應該不相似）
code3 = """
if condition:
    do_something()
else:
    do_other()
"""

code4 = """
if other_condition:
    do_different()
else:
    do_another()
"""
# 期望：不相似（語義不同）

# 測試3：註釋不同但代碼相同（應該相似）
code5 = """
# This is a function
def func():
    return 1
"""

code6 = """
# Different comment
def func():
    return 1
"""
# 期望：相似（移除註釋後相同）
```

## 六、實施優先級

### 高優先級（立即實施）
1. ✅ **代碼標準化**（移除空白、註釋）- 最簡單有效
2. ✅ **差異比例檢查** - 快速過濾明顯誤報
3. ✅ **動態閾值調整** - 根據代碼塊大小調整

### 中優先級（短期實施）
1. **結構相似度分析** - 提高準確性
2. **多指標綜合判斷** - 減少誤報
3. **語義相似度分析** - 進一步減少誤報

### 低優先級（長期優化）
1. **上下文分析** - 複雜但可能有用
2. **機器學習方法** - 需要訓練數據

## 七、預期效果

實施這些優化後，預期可以：
- **減少 60-80% 的誤報**
- **保持 90%+ 的真陽性率**（不會漏掉真正的相似代碼）
- **提高結果的可信度**

## 八、使用建議

### 8.1 對於不同場景的參數設置

**場景1：嚴格模式（減少誤報）**
```bash
python refactor --similarity 0.85 --strict-mode --min-semantic-similarity 0.3 file1.py
```

**場景2：平衡模式（默認）**
```bash
python refactor --similarity 0.8 --use-normalization file1.py
```

**場景3：寬鬆模式（找出更多相似代碼）**
```bash
python refactor --similarity 0.75 --max-diff-ratio 0.4 file1.py
```

