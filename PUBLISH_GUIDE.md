# FPGA Hardware Design Guide - 发布检查清单

## 📋 文件完整性检查

### ✅ 必须文件（缺一不可）

| 文件 | 位置 | 状态 | 说明 |
|------|------|------|------|
| SKILL.md | 根目录 | ✅ | 主skill文件，必须包含YAML frontmatter |
| README.md | 根目录 | ✅ | GitHub仓库首页显示 |
| LICENSE | 根目录 | ✅ | 许可证文件（MIT推荐）|

### ✅ 可选但建议的文件

| 文件/文件夹 | 位置 | 状态 | 说明 |
|------------|------|------|------|
| references/ | 根目录 | ✅ | 参考文档文件夹 |
| .gitignore | 根目录 | ⬜ | Git忽略文件 |
| CHANGELOG.md | 根目录 | ⬜ | 版本更新记录 |

---

## 🔍 SKILL.md 规范检查

### YAML Frontmatter（必须）

```yaml
---
name: fpga-hardware-design-guide                    ✅ 已设置
description: >                                        ✅ 已设置
  Personal FPGA hardware design guide based on real project experience.
  Covers pipeline design, timing optimization, SystemVerilog coding patterns,
  and practical debugging techniques. Use this skill when:
  (1) designing FPGA modules with timing constraints,
  (2) implementing video processing or data path designs,
  (3) optimizing for resource utilization and timing closure,
  (4) reviewing RTL code for hardware implementation,
  (5) debugging synthesis and implementation issues.
license: MIT                                          ✅ 已设置
compatibility: Works with Claude Code...              ✅ 已设置
metadata:                                             ✅ 已设置
  author: peng
  version: "1.0"
allowed-tools: Read Write Edit Bash                   ✅ 已设置
---
```

### 内容结构检查

- [x] 有清晰的概述/介绍
- [x] 分章节组织内容
- [x] 包含代码示例
- [x] 提到参考文件位置
- [x] 总长度 < 500行 ✅（实际约200行）

---

## 📁 目录结构验证

正确的结构：
```
fpga-hardware-design-guide/           ← 仓库根目录
├── .git/                            ← Git仓库（自动生成）
├── .gitignore                       ← （可选但建议）
├── LICENSE                          ✅ 已创建
├── README.md                        ✅ 已创建
├── SKILL.md                         ✅ 已创建
└── references/                      ✅ 已创建
    ├── design-patterns.md          ✅ 已创建
    ├── device-selection.md         ✅ 已创建
    └── troubleshooting.md          ✅ 已创建
```

**❌ 常见错误结构：**
```
❌ fpga-hardware-design-guide/
  └── fpga-hardware-design-guide/   ← 不要嵌套文件夹
      ├── SKILL.md
      └── ...

❌ fpga-hardware-design-guide/
    ├── skill/                      ← 不要放在子文件夹
    │   └── SKILL.md
    └── README.md
```

---

## 🚀 发布步骤详解

### 步骤 1：创建 GitHub 仓库

**1.1 访问 GitHub**
- 打开浏览器：https://github.com
- 登录你的账号

**1.2 创建新仓库**
- 点击右上角 **+** 号 → **New repository**
- **Repository name**: `fpga-hardware-design-guide`
  - ⚠️ 必须和 skill name 完全一致！
  - 使用小写字母和连字符
- **Description**: 
  ```
  Personal FPGA hardware design guide based on real project experience. Covers pipeline design, timing optimization, and practical debugging.
  ```
- **Visibility**: ✅ **Public**（必须公开，否则别人无法安装）
- ✅ **Initialize this repository with**: 
  - ☑️ Add a README file（可选，我们也可以手动上传）
- 点击 **Create repository**

**1.3 记录仓库地址**
创建成功后，地址将是：
```
https://github.com/YOUR_USERNAME/fpga-hardware-design-guide
```

---

### 步骤 2：上传文件到 GitHub

#### 方法 A：网页上传（推荐，避免网络问题）

**2.1 上传 SKILL.md**
1. 在新创建的仓库页面，点击 **"Add file"** → **"Upload files"**
2. 拖拽或选择 `SKILL.md` 文件
3. Commit message: `Add main SKILL.md`
4. 点击 **Commit changes**

**2.2 创建 references 文件夹并上传文件**

**创建文件夹方法：**
1. 点击 **"Add file"** → **"Create new file"**
2. 在文件名框输入：`references/design-patterns.md`
   - 这会同时创建 `references` 文件夹和 `design-patterns.md` 文件
3. 打开本地的 `design-patterns.md`，复制全部内容
4. 粘贴到 GitHub 编辑框
5. Commit message: `Add design patterns reference`
6. 点击 **Commit changes**

**重复上述步骤上传其他文件：**
- `references/troubleshooting.md`
- `references/device-selection.md`
- `LICENSE`（如果创建仓库时没选）
- `README.md`（如果创建仓库时没选）

**2.3 验证文件结构**
上传完成后，仓库页面应该显示：
```
fpga-hardware-design-guide/
├── LICENSE
├── README.md
├── SKILL.md
└── references/
    ├── design-patterns.md
    ├── device-selection.md
    └── troubleshooting.md
```

#### 方法 B：Git 命令行（需要稳定网络）

如果你能稳定访问 GitHub：

```bash
# 1. 进入skill目录
cd E:\peng\AI\opencode\test1\fpga-hardware-design-guide

# 2. 初始化Git仓库
git init

# 3. 添加所有文件
git add .

# 4. 提交
git commit -m "Initial release of FPGA Hardware Design Guide"

# 5. 添加远程仓库（替换YOUR_USERNAME）
git remote add origin https://github.com/YOUR_USERNAME/fpga-hardware-design-guide.git

# 6. 推送到GitHub
git branch -M main
git push -u origin main
```

---

### 步骤 3：提交到 skills.sh

#### 方法 1：通过 skills.sh 网站（推荐）

**3.1 访问 skills.sh**
- 打开：https://skills.sh
- 寻找 **"Submit a Skill"**、**"Add Skill"** 或 **"Publish"** 按钮

**3.2 填写提交表单**

| 字段 | 填写内容 | 示例 |
|------|---------|------|
| **Repository URL** | GitHub仓库地址 | `https://github.com/peng/fpga-hardware-design-guide` |
| **Skill Name** | 和仓库名一致 | `fpga-hardware-design-guide` |
| **Description** | 从SKILL.md复制 | `Personal FPGA hardware design guide...` |
| **Category** | 选择分类 | `Hardware` 或 `FPGA` |
| **Tags** | 关键词 | `fpga, xilinx, hardware, timing` |

**3.3 等待审核**
- 提交后通常需要1-3天审核
- 审核通过后会显示在 skills.sh 上

#### 方法 2：使用 npx skills publish

```bash
npx skills publish https://github.com/YOUR_USERNAME/fpga-hardware-design-guide
```

---

## ⚠️ 常见错误及避免方法

### 错误 1：文件位置错误

**❌ 错误示例：**
```
my-skill/
└── my-skill/
    ├── SKILL.md
    └── references/
```

**✅ 正确结构：**
```
my-skill/
├── SKILL.md
└── references/
```

### 错误 2：SKILL.md 缺少 frontmatter

**❌ 错误：**
```markdown
# My Skill
没有YAML frontmatter
```

**✅ 正确：**
```markdown
---
name: my-skill
description: ...
---

# My Skill
```

### 错误 3：仓库是私有的

**❌ 错误：**
- 创建时选择了 **Private**
- 结果：别人无法访问，无法安装

**✅ 正确：**
- 必须选择 **Public**

### 错误 4：文件名大小写错误

**❌ 错误：**
- `skill.md`（小写）
- `Skill.md`（首字母大写）

**✅ 正确：**
- `SKILL.md`（全大写）

### 错误 5：name 和仓库名不一致

**❌ 错误：**
- 仓库名：`fpga-hardware-design-guide`
- SKILL.md name：`fpga-guide`

**✅ 正确：**
- 仓库名：`fpga-hardware-design-guide`
- SKILL.md name：`fpga-hardware-design-guide`

---

## ✅ 发布前最终检查

### 文件检查
- [ ] GitHub仓库已创建
- [ ] 仓库为 **Public**
- [ ] SKILL.md 在根目录
- [ ] SKILL.md 包含 YAML frontmatter
- [ ] references/ 文件夹已上传
- [ ] README.md 已添加
- [ ] LICENSE 已添加

### 内容检查
- [ ] name 和仓库名一致
- [ ] description 清晰完整
- [ ] 没有语法错误
- [ ] 文件总大小 < 1MB

### 功能检查
- [ ] 可以通过 URL 访问仓库
- [ ] 可以看到所有文件列表
- [ ] 可以查看 SKILL.md 内容

---

## 🧪 测试安装

发布前，先测试别人能否安装：

**方法 1：本地测试**
```bash
# 尝试安装自己的skill
npx skills add https://github.com/YOUR_USERNAME/fpga-hardware-design-guide --skill fpga-hardware-design-guide
```

**方法 2：请朋友测试**
- 把GitHub仓库链接发给朋友
- 让他们尝试安装

---

## 📞 如果出问题

### 别人看不到 skill？

**检查清单：**
1. 仓库是 Public 吗？
2. SKILL.md 在根目录吗？
3. 有 YAML frontmatter 吗？
4. name 和仓库名一致吗？

### 安装失败？

**常见原因：**
- 网络问题（GitHub访问不了）
- 文件路径错误
- 缺少 frontmatter

**解决方案：**
- 检查仓库地址是否正确
- 检查文件结构
- 使用本地路径测试

---

## 🎯 快速开始（简化版）

如果你只想快速发布，按这个顺序：

1. **创建GitHub仓库**（Public，加README）
2. **上传 SKILL.md**（必须有 frontmatter）
3. **创建 references/ 文件夹**，上传3个参考文件
4. **访问 skills.sh**，提交表单
5. **等待审核通过**

---

*按照这份清单操作，100%不会出错！*
