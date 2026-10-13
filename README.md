# 🎯 Embedded C Conventions - 嵌入式C编程规范

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Language: C](https://img.shields.io/badge/Language-C-blue.svg)](https://en.wikipedia.org/wiki/C_(programming_language))
[![Standards: MISRA C](https://img.shields.io/badge/Standards-MISRA%20C-green.svg)](https://misra.org.uk/)
[![Cursor Skill](https://img.shields.io/badge/Cursor-Skill-FF4D4F.svg)](https://cursor.com/)

一套专为嵌入式系统开发设计的 C 编程规范体系。通过清晰的架构、严格的安全规范和解耦设计，提升代码质量和团队协作效率。

**中文文档 | 实战指南 | 22项检查清单 | 13个规范模块**

---

## 📋 目录

- [核心特性](#核心特性)
- [快速开始](#快速开始)
- [规范模块](#规范模块)
- [文件说明](#文件说明)
- [使用指南](#使用指南)
- [实战示例](#实战示例)
- [最佳实践](#最佳实践)
- [贡献指南](#贡献指南)

---

## 🌟 核心特性

### ✅ 13 个规范模块

涵盖嵌入式开发的全方位需求：

| 类别 | 规范 | 目标 |
|------|------|------|
| **基础规范** | 魔法数字、大括号、命名 | 代码一致性 |
| **文档规范** | 注释、函数文档 | 代码可读性 |
| **安全规范** | 错误处理、竞态条件、缓冲溢出 | 代码可靠性 |
| **架构规范** | 分层设计、函数表模式 | 代码可维护性 |
| **资源管理** | 内存安全、资源配对 | 代码稳定性 |

### 🔒 安全第一

- 缓冲区溢出防护
- 竞态条件检测
- ISR 中断安全
- 资源泄漏防护
- 指针安全验证

### 🏗️ 解耦架构

- 清晰的分层模型
- 函数表模式消除 switch 地狱
- 虚接口支持设备切换
- 状态机集中管理

### 📖 中文文档

- 完整的规范定义
- 22 项代码审查清单
- 实战示例和最佳实践
- 快速参考卡片

---

## 🚀 快速开始

### 方法 1: 在 Cursor IDE 中使用

如果你使用 [Cursor IDE](https://cursor.com/)（基于 VS Code 的 AI-first 编辑器）：

```bash
# 1. 克隆本仓库到 Cursor Skills 文件夹
git clone https://github.com/YOUR_USERNAME/embedded-c-conventions.git \
  ~/.cursor/skills/embedded-c-conventions

# 2. 在 Cursor 中使用
/embedded-c-conventions
请按规范审查这段代码
```

### 方法 2: 作为参考文档

```bash
# 克隆仓库
git clone https://github.com/YOUR_USERNAME/embedded-c-conventions.git
cd embedded-c-conventions

# 阅读文档
# - INDEX.md: 快速导航
# - QUICK_REFERENCE.md: 规范速查表
# - SKILL.md: 完整规范定义
# - LAYERING_AND_FUNCTION_TABLE.md: 分层架构详解
```

---

## 📚 规范模块

### 🔵 基础规范（8个）

| # | 规范 | 描述 |
|---|------|------|
| 1️⃣ | 魔法数字 → 宏定义 | 所有数字常量都用宏，便于维护 |
| 2️⃣ | 大括号换行 | `{` 和 `}` 必须独占一行 |
| 3️⃣ | 变量命名 | g_/s_/p_ 前缀规范 |
| 4️⃣ | 注释规范 | 中文注释，函数有完整文档 |
| 5️⃣ | 全局变量 | 每个都有 @rationale 理由 |
| 6️⃣ | 错误处理 | 统一的 ErrorCode_t 返回码 |
| 7️⃣ | 结构体设计 | 字段分组、按大小排序 |
| 8️⃣ | AI 工作流 | 代码注释规范、对话总结 |

**示例**:
```c
// ✅ 好的做法
#define UART_RX_BUFFER_SIZE     (256)
#define UART_TIMEOUT_MS         (100)

static uint8_t s_uart_buffer[UART_RX_BUFFER_SIZE];  // g_ = 全局，s_ = 静态
static volatile uint16_t s_rx_index = 0;             // volatile = ISR 访问

ErrorCode_t uart_init(const UartConfig_t *p_config);  // p_ = 指针
```

### 🟢 安全规范（5个）

| # | 规范 | 描述 |
|---|------|------|
| 9️⃣ | 函数设计 | 参数 ≤4 个、长度 ≤100 行、嵌套 ≤3 层 |
| 🔟 | 竞态条件 | volatile、临界段、@isr 文档 |
| 1️⃣1️⃣ | 缓冲溢出 | 尺寸验证、指针检查、边界保护 |
| 1️⃣2️⃣ | 资源泄漏 | init/close 配对、失败路径清理 |
| 1️⃣3️⃣ | 分层架构 | 函数表、虚接口、状态机 |

**示例 - 缓冲溢出防护**:
```c
// ❌ 危险
void uart_recv(uint8_t *p_data, uint16_t len)
{
    memcpy(s_buffer, p_data, len);  // 没检查大小！
}

// ✅ 安全
ErrorCode_t uart_recv(uint8_t *p_data, uint16_t len)
{
    if (p_data == NULL || len == 0 || len > UART_MAX_SIZE)
        return ERROR_INVALID_PARAM;
    
    if (s_buffer_space < len)
        return ERROR_BUFFER_OVERFLOW;
    
    memcpy(s_buffer, p_data, len);
    return ERROR_SUCCESS;
}
```

---

## 📁 文件说明

```
embedded-c-conventions/
├── README.md                              # 项目说明（本文件）
├── INDEX.md                               # 📖 快速导航和文件索引
├── SKILL.md                               # 📋 13个规范的完整定义（核心）
├── QUICK_REFERENCE.md                     # ⚡ 规范速查卡片
└── LAYERING_AND_FUNCTION_TABLE.md         # 🏗️ 分层架构和函数表实战指南
```

### 各文件用途

| 文件 | 大小 | 用途 | 首选用户 |
|------|------|------|---------|
| **INDEX.md** | 5.2 KB | 快速导航 | 所有人 - 推荐首先阅读 |
| **QUICK_REFERENCE.md** | 16 KB | 规范速查表 | 快速查阅、代码审查 |
| **SKILL.md** | 48.8 KB | 完整规范定义 | 深入学习、理解背景 |
| **LAYERING_AND_FUNCTION_TABLE.md** | 18.7 KB | 实战指南 | 架构设计、代码重构 |

---

## 💡 使用指南

### 场景 1: 快速审查代码

```
打开：QUICK_REFERENCE.md
检查内容：
- 变量命名是否正确
- 是否有魔法数字
- 函数是否过长（>100行）
- 是否检查返回值
```

### 场景 2: 深入学习规范

```
1. 先读 INDEX.md 了解全貌
2. 再读 QUICK_REFERENCE.md 看代码示例
3. 最后读 SKILL.md 理解完整背景
```

### 场景 3: 解决架构问题

```
问题：代码中有巨大的 switch 语句
解决：打开 LAYERING_AND_FUNCTION_TABLE.md
      参考"命令处理"部分的函数表模式

问题：多个设备驱动代码重复
解决：参考"虚接口模式"部分

问题：状态转换逻辑复杂
解决：参考"状态机模式"部分
```

### 场景 4: 团队代码审查

```
使用提供的 22 项检查清单（见 SKILL.md 末尾）
✓ 基础规范 (8 项)
✓ 函数设计 (5 项)
✓ 并发安全 (4 项)
✓ 内存安全 (4 项)
✓ 架构设计 (6 项)
✓ 模块接口 (3 项)
```

---

## 🔍 实战示例

### 示例 1: 解决 Switch 地狱

**❌ 问题代码** (500+ 行的 switch)：
```c
void handle_command(uint8_t cmd, uint8_t *p_data, uint16_t len)
{
    switch (cmd)
    {
        case 0x01:
            // ... 20 行处理逻辑
        case 0x02:
            // ... 20 行处理逻辑
        // ... 还有 20 多个 case
    }
}
```

**✅ 解决方案** (函数表模式)：
```c
typedef ErrorCode_t (*cmd_handler_t)(const uint8_t *p_data, uint16_t len);

static ErrorCode_t cmd_start(const uint8_t *p_data, uint16_t len) { ... }
static ErrorCode_t cmd_stop(const uint8_t *p_data, uint16_t len) { ... }

static const struct {
    uint8_t cmd_id;
    cmd_handler_t handler;
} g_cmd_table[] = {
    { 0x01, cmd_start },
    { 0x02, cmd_stop },
};

ErrorCode_t dispatch_command(uint8_t cmd, const uint8_t *p_data, uint16_t len)
{
    for (int i = 0; i < sizeof(g_cmd_table)/sizeof(g_cmd_table[0]); i++)
    {
        if (g_cmd_table[i].cmd_id == cmd)
            return g_cmd_table[i].handler(p_data, len);
    }
    return ERROR_UNKNOWN;
}
```

👉 **详见** `LAYERING_AND_FUNCTION_TABLE.md` 第二部分

### 示例 2: 缓冲区溢出防护

**❌ 危险代码**:
```c
void process_data(uint8_t *p_data, uint16_t len)
{
    memcpy(s_buffer, p_data, len);  // 没检查！
}
```

**✅ 安全代码**:
```c
ErrorCode_t process_data(const uint8_t *p_data, uint16_t len)
{
    // 参数检查
    if (p_data == NULL || len == 0 || len > MAX_SIZE)
        return ERROR_INVALID_PARAM;
    
    // 缓冲检查
    if (available_space < len)
        return ERROR_BUFFER_OVERFLOW;
    
    // 安全操作
    memcpy(s_buffer, p_data, len);
    return ERROR_SUCCESS;
}
```

👉 **详见** `QUICK_REFERENCE.md` 第 10 节或 `SKILL.md` 规范 #11

---

## 🎓 最佳实践

### 1. 代码审查清单

在审查代码前，检查以下要点：

```
[ ] 魔法数字都用宏定义了吗？
[ ] 括号格式是否正确（{} 都在新行）？
[ ] 变量命名是否一致（g_/s_/p_ 前缀）？
[ ] 函数是否有文档注释（@brief/@param/@return）？
[ ] 参数是否验证了（NULL、范围、大小）？
[ ] 返回错误码是否被检查了？
[ ] 是否有 volatile 标记的 ISR 访问变量？
[ ] 是否有大的 switch 或 if-else？→ 考虑用函数表
```

### 2. 分层架构建议

```
APP 层 (应用逻辑)
   ↓ 依赖
业务层 (算法、状态机)
   ↓ 依赖
驱动层 (设备驱动)
   ↓ 依赖
底层 HAL (硬件初始化)
```

### 3. 命名规范速记

```c
g_global_var           // 全局变量
s_static_var           // 静态变量
p_pointer              // 指针参数
is_flag                // 布尔标志
MAX_SIZE               // 常量宏
MODULE_function_name() // 函数命名
```

### 4. 文件组织建议

```
module/
├── module.h              # 公共接口
├── module.c              # 实现
├── module_internal.h     # 内部定义
└── module_test.c         # 单元测试
```

---

## 📊 适用场景

✅ **强烈推荐**
- 嵌入式 MCU 开发（STM32、ARM、RISC-V 等）
- 实时操作系统（RTOS）应用
- 硬件驱动开发
- 固件开发
- 多人协作项目

✅ **可以参考**
- IoT 设备开发
- Linux 内核模块
- 系统编程
- C 语言最佳实践学习

---

## 🤝 贡献指南

欢迎贡献改进！

### 贡献方式

1. **反馈问题**
   - 提交 Issue 描述遇到的问题
   - 提供具体的代码示例

2. **提交改进**
   - Fork 本仓库
   - 创建特性分支 (`git checkout -b feature/improvement`)
   - 提交更改 (`git commit -am 'Add improvement'`)
   - 推送分支 (`git push origin feature/improvement`)
   - 创建 Pull Request

3. **补充文档**
   - 添加新的规范模块
   - 提供更多实战示例
   - 改进中文翻译

### 贡献建议

- 保持代码示例简洁清晰
- 中文注释和英文变量名结合
- 遵循现有的文档格式
- 提供 "问题 → 解决方案" 对比

---

## 📖 参考资源

- [MISRA C](https://misra.org.uk/) - C 代码安全标准
- [Linux Kernel Coding Style](https://www.kernel.org/doc/html/latest/process/coding-style.html) - Linux 内核编程风格
- [SEI CERT C](https://wiki.sei.cmu.edu/confluence/display/c/SEI+CERT+C+Coding+Standard) - 安全编码标准
- [Embedded C Programming](https://en.wikipedia.org/wiki/Embedded_C) - 嵌入式 C 背景知识

---

## 📝 许可证

本项目采用 MIT 许可证。详见 [LICENSE](LICENSE) 文件。

---

## 👥 作者

- **开发者**: chengyuchung and you
- **维护者**: 
- **贡献者**: all

---

## 💬 交流

- 📧 Email: 
- 💬 讨论: [Discussions](https://github.com/chengyuchung/embedded-c-conventions/discussions)
- 🐛 问题: [Issues](https://github.com/chengyuchung/embedded-c-conventions/issues)

---

## 🎯 下一步

1. **快速上手**: 打开 `INDEX.md` 开始浏览
2. **深入学习**: 阅读 `QUICK_REFERENCE.md` 了解具体规范
3. **实战应用**: 参考 `LAYERING_AND_FUNCTION_TABLE.md` 解决架构问题
4. **完整掌握**: 学习 `SKILL.md` 中的所有 13 个规范模块

---

## ⭐ 如果有帮助，请给个 Star！

```
git clone https://github.com/YOUR_USERNAME/embedded-c-conventions.git
cd embedded-c-conventions
# 阅读 README.md 和 INDEX.md 开始！
```

---

**最后更新**: 2026-10-13 | **版本**: 3.0 | **状态**: ✅ 稳定版本

