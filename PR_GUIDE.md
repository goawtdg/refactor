# 如何提交改进并创建 PR

## 一、准备工作

### 1.1 Fork 原项目（如果还没有）

1. 访问原项目：https://github.com/forhadahmed/refactor
2. 点击右上角的 **Fork** 按钮
3. 等待 Fork 完成，您会得到一个 `您的用户名/refactor` 仓库

### 1.2 克隆您的 Fork 到本地

```bash
# 克隆您的 fork
git clone https://github.com/您的用户名/refactor.git
cd refactor

# 添加上游仓库（原项目）作为远程仓库
git remote add upstream https://github.com/forhadahmed/refactor.git

# 验证远程仓库
git remote -v
# 应该看到：
# origin    https://github.com/您的用户名/refactor.git (fetch)
# origin    https://github.com/您的用户名/refactor.git (push)
# upstream  https://github.com/forhadahmed/refactor.git (fetch)
# upstream  https://github.com/forhadahmed/refactor.git (push)
```

## 二、提交您的改进

### 2.1 创建新分支（推荐）

```bash
# 创建一个新分支用于您的改进
git checkout -b improve-false-positive-detection

# 或者使用更描述性的分支名
git checkout -b fix-cross-platform-compatibility
```

### 2.2 进行代码修改

在您的编辑器中修改代码，例如：
- 修复误报问题
- 改进跨平台兼容性
- 性能优化

### 2.3 提交更改

```bash
# 查看修改的文件
git status

# 添加修改的文件
git add refactor/refactor
git add refactor/README.md
# 或者添加所有修改
git add .

# 提交更改（使用清晰的提交信息）
git commit -m "feat: 添加代码标准化以减少误报

- 实现代码标准化函数，移除空白和注释
- 添加结构特征提取和语义相似度计算
- 使用多指标综合判断相似度
- 修复 IndexError 和字符串迭代错误"
```

**提交信息格式建议：**
- `feat:` - 新功能
- `fix:` - 修复 bug
- `perf:` - 性能优化
- `docs:` - 文档更新
- `refactor:` - 代码重构

### 2.4 推送到您的 Fork

```bash
# 推送到您的 fork（第一次推送需要设置上游）
git push -u origin improve-false-positive-detection

# 之后的推送
git push
```

## 三、创建 Pull Request

### 3.1 在 GitHub 上创建 PR

1. **访问您的 Fork**
   - 打开：https://github.com/您的用户名/refactor

2. **点击 "Compare & pull request" 按钮**
   - GitHub 通常会在您推送新分支后显示这个按钮
   - 或者点击 **Pull requests** 标签页，然后点击 **New pull request**

3. **选择分支**
   - **base repository**: `forhadahmed/refactor` (原项目)
   - **base branch**: `main` 或 `master` (原项目的主分支)
   - **compare**: `您的用户名/refactor` → `improve-false-positive-detection` (您的分支)

4. **填写 PR 信息**

**标题示例：**
```
feat: 添加代码标准化和结构分析以减少误报
```

**描述示例：**
```markdown
## 改进内容

本次 PR 包含以下改进：

### 🎯 主要功能
- ✅ 实现代码标准化函数，移除空白字符和注释后再比较
- ✅ 添加结构特征提取（关键字、操作符）
- ✅ 添加语义相似度计算（变量名、函数名）
- ✅ 使用多指标综合判断，减少误报

### 🐛 Bug 修复
- ✅ 修复第193行 IndexError（添加 indent 检查）
- ✅ 修复第283行逻辑错误（返回 None 而不是 0）
- ✅ 修复第388行字符串迭代错误

### 📊 预期效果
- 减少 60-80% 的误报
- 保持 90%+ 的真阳性率

## 测试
- [x] 测试了 Python 文件
- [x] 测试了 C/C++ 文件
- [x] 验证了误报减少效果

## 相关 Issue
Closes #123 (如果有相关 issue)
```

5. **提交 PR**
   - 点击 **Create pull request**

### 3.2 PR 链接格式

创建 PR 后，您会得到一个链接，格式为：
```
https://github.com/forhadahmed/refactor/pull/123
```
其中 `123` 是 PR 的编号。

## 四、在 README 中链接到 PR

### 4.1 更新 README.md

在 README 的适当位置添加 PR 链接：

```markdown
# Refactor Tool

> 这是 [原始项目](https://github.com/forhadahmed/refactor) 的改进版本

## 改进内容

本版本包含以下改进：

### 减少误报优化
- 实现代码标准化，移除空白和注释
- 添加结构特征和语义相似度分析
- 使用多指标综合判断

**相关 PR：** [#123](https://github.com/forhadahmed/refactor/pull/123)

### 跨平台兼容性
- 使用 Python 标准库替代 shell 命令
- 支持 Windows、Linux、macOS

**相关 PR：** [#124](https://github.com/forhadahmed/refactor/pull/124)

### Bug 修复
- 修复 IndexError 和字符串迭代错误
- 改进错误处理

**相关 PR：** [#125](https://github.com/forhadahmed/refactor/pull/125)
```

### 4.2 在 README 顶部添加说明

```markdown
# Refactor Tool

[![PR](https://img.shields.io/badge/PR-%23123-blue)](https://github.com/forhadahmed/refactor/pull/123)

> 这是 [原始项目](https://github.com/forhadahmed/refactor) 的改进版本
> 
> 主要改进：
> - 🎯 [减少误报优化](https://github.com/forhadahmed/refactor/pull/123)
> - 🖥️ [跨平台兼容性改进](https://github.com/forhadahmed/refactor/pull/124)
> - 🐛 [Bug 修复](https://github.com/forhadahmed/refactor/pull/125)
```

### 4.3 在贡献部分添加

```markdown
## 贡献

如果您想向原项目贡献代码，可以：

1. Fork 本项目
2. 创建新分支进行改进
3. 提交 Pull Request

### 已提交的 PR

- [#123 - 减少误报优化](https://github.com/forhadahmed/refactor/pull/123)
- [#124 - 跨平台兼容性改进](https://github.com/forhadahmed/refactor/pull/124)
- [#125 - Bug 修复](https://github.com/forhadahmed/refactor/pull/125)
```

## 五、保持 Fork 同步

### 5.1 同步上游更改

```bash
# 获取上游仓库的最新更改
git fetch upstream

# 切换到主分支
git checkout main  # 或 master

# 合并上游更改
git merge upstream/main  # 或 upstream/master

# 推送到您的 fork
git push origin main
```

### 5.2 更新您的 PR 分支

```bash
# 切换到您的 PR 分支
git checkout improve-false-positive-detection

# 合并最新的主分支
git merge main

# 推送到 GitHub（PR 会自动更新）
git push
```

## 六、PR 最佳实践

### 6.1 提交信息规范

```bash
# 好的提交信息
git commit -m "feat: 添加代码标准化功能

- 实现 standardize_code 函数
- 移除空白字符和注释
- 提高相似度判断准确性"

# 不好的提交信息
git commit -m "更新代码"  # 太模糊
```

### 6.2 PR 描述模板

```markdown
## 问题描述
简要描述要解决的问题或添加的功能

## 解决方案
描述您的解决方案

## 改进内容
- [ ] 功能1
- [ ] 功能2

## 测试
- [ ] 单元测试通过
- [ ] 集成测试通过
- [ ] 手动测试完成

## 截图/示例
（如果有的话）

## 相关 Issue
Closes #123
```

### 6.3 代码审查准备

- ✅ 确保代码可以正常运行
- ✅ 添加必要的注释
- ✅ 遵循项目的代码风格
- ✅ 更新相关文档
- ✅ 添加测试（如果有测试框架）

## 七、常见问题

### Q1: 如何修改已提交的 PR？

```bash
# 修改代码后
git add .
git commit -m "fix: 修复某个问题"
git push
# PR 会自动更新
```

### Q2: 如何关闭 PR？

在 GitHub PR 页面点击 **Close pull request**，或者在 PR 描述中添加：
```
/closes
```

### Q3: 如何引用 Issue？

在 PR 描述中使用：
```
Closes #123
Fixes #123
Resolves #123
```

### Q4: 如何添加 PR 标签？

通常由仓库维护者添加，但您可以在 PR 描述中建议：
```markdown
建议标签：enhancement, bug-fix
```

## 八、完整示例流程

```bash
# 1. Fork 并克隆
git clone https://github.com/您的用户名/refactor.git
cd refactor
git remote add upstream https://github.com/forhadahmed/refactor.git

# 2. 创建分支
git checkout -b improve-false-positive-detection

# 3. 修改代码
# ... 在编辑器中修改 ...

# 4. 提交
git add .
git commit -m "feat: 添加代码标准化以减少误报"

# 5. 推送
git push -u origin improve-false-positive-detection

# 6. 在 GitHub 上创建 PR
# 访问 https://github.com/您的用户名/refactor
# 点击 "Compare & pull request"
# 填写 PR 信息并提交

# 7. 在 README 中添加 PR 链接
# 编辑 README.md，添加 PR 链接
git add README.md
git commit -m "docs: 在 README 中添加 PR 链接"
git push
```

## 九、PR 链接的 Markdown 格式

### 基本链接
```markdown
[PR #123](https://github.com/forhadahmed/refactor/pull/123)
```

### 带描述的链接
```markdown
[减少误报优化 PR](https://github.com/forhadahmed/refactor/pull/123)
```

### 在表格中
```markdown
| 改进内容 | PR 链接 |
|---------|---------|
| 减少误报 | [#123](https://github.com/forhadahmed/refactor/pull/123) |
| 跨平台支持 | [#124](https://github.com/forhadahmed/refactor/pull/124) |
```

### 使用徽章
```markdown
[![PR](https://img.shields.io/badge/PR-%23123-blue)](https://github.com/forhadahmed/refactor/pull/123)
```

## 十、总结

1. **Fork 原项目** → 获得您自己的副本
2. **创建分支** → 在新分支上工作
3. **提交更改** → 使用清晰的提交信息
4. **推送分支** → 推送到您的 fork
5. **创建 PR** → 在 GitHub 上创建 Pull Request
6. **更新 README** → 添加 PR 链接
7. **等待审查** → 维护者会审查您的 PR

完成这些步骤后，您的改进就会出现在原项目的 PR 列表中，维护者可以审查并合并您的代码！

