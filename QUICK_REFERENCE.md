---
title: Embedded C Conventions - Quick Reference (Team Version - Chinese Comments)
---

# 嵌入式 C 代码规范 - 快速参考（团队版 - 中文注释）

## 1️⃣ 魔法数字 → 宏定义

```c
❌ delay(1000);
✅ #define INIT_DELAY_MS (1000)
   delay(INIT_DELAY_MS);

❌ if (value > 2048) { ... }
✅ #define ADC_THRESHOLD (2048)
   if (value > ADC_THRESHOLD) { ... }
```

**宏命名规则：** `MODULE_DESCRIPTION` (全大写，带模块前缀)
- `UART_BAUD_RATE`, `TIMER_PRESCALE`, `GPIO_PIN_COUNT`

---

## 2️⃣ 大括号必须换行

```c
❌ if (ready) {          ✅ if (ready)
   ...                     {
}                              ...
                           }

❌ void init() { ... }    ✅ void init()
                            { ... }

✅ else if (condition)  // 这是对的！
   {
```

---

## 3️⃣ 变量和函数命名

| 类型 | 规则 | 示例 |
|------|------|------|
| 全局变量 | `g_module_name` | `g_uart_state` |
| 静态变量 | `s_module_name` | `s_timer_config` |
| 局部变量 | `lowercase_underscore` | `error_count` |
| 指针 | `p_type_name` | `p_uart_config` |
| 函数 | `module_action()` | `uart_send_byte()` |
| 宏/常量 | `UPPERCASE_UNDERSCORE` | `BUFFER_SIZE` |
| 结构体 typedef | `ModuleName_t` | `UartConfig_t` |
| 布尔变量 | `is_/has_/should_` | `is_ready`, `has_error` |

---

## 4️⃣ 注释规范（中文注释）

### .h 文件 - 必须注释函数声明

```c
/**
 * @brief 初始化 UART 外设
 * @param p_config 指向 UartConfig_t 配置的指针
 * @return ErrorCode_t - 成功返回 ERROR_SUCCESS
 * @note 初始化期间最多阻塞 100ms
 */
ErrorCode_t uart_init(const UartConfig_t *p_config);

/**
 * @brief 通过 UART 发送字节
 * @param byte 要发送的字节（0-255）
 * @return ErrorCode_t - 成功或缓冲区溢出
 */
ErrorCode_t uart_send_byte(uint8_t byte);
```

### 宏定义 - 标注每个常量的含义

```c
// ============================================================================
// UART 配置宏定义
// ============================================================================

#define UART_BAUD_RATE_115200      (8)      // 波特率 115200 的 BRR 值
#define UART_INIT_DELAY_MS         (100)    // 电压稳定等待时间
#define UART_BUFFER_SIZE           (256)    // 接收缓冲区大小
```

### 枚举 - 标注每个状态值

```c
typedef enum {
    UART_STATE_UNINITIALIZED,    // 未初始化
    UART_STATE_READY,            // 就绪，可收发
    UART_STATE_TRANSMITTING,     // DMA 正在发送
    UART_STATE_ERROR             // 错误状态
} UartState_t;
```

### 结构体 - 标注每个字段

```c
typedef struct {
    uint32_t baud_rate;          // 波特率（9600、115200 等）
    uint16_t buffer_size;        // 接收缓冲区大小
    uint8_t data_bits;           // 数据位（8 或 9）
} UartConfig_t;
```

### .c 文件 - 只注释关键部分（中文）

```c
❌ 不要注释明显的代码：
int x = 0;              // 初始化 x 为 0（太明显）
i++;                    // i 自增（太明显）

✅ 注释关键部分（为什么）：
// 为什么：禁用中断防止临界段竞态条件
DISABLE_INTERRUPTS();
TIMER->CR1 = value;
ENABLE_INTERRUPTS();

// 为什么：轮询而非忙等，允许调度器运行
uint32_t deadline = get_ticks() + UART_READY_TIMEOUT_MS;
while (!is_uart_ready() && get_ticks() < deadline)
{
    yield_to_scheduler();
}
```

### 全局/静态变量 - 说明理由（中文）

```c
/**
 * @brief DMA 发送缓冲区
 * @rationale: DMA 控制器需要固定内存地址
 * @thread-safe: ISR 中通过禁用中断保护
 */
static uint8_t s_uart_tx_buffer[256];

/**
 * @brief UART 状态标志
 * @rationale: 需要在函数调用间保持状态
 * @note: 仅通过 uart_get_state() 访问
 */
static uint8_t s_is_initialized = 0;
```

---

## 5️⃣ 错误处理规范

```c
// 定义错误码
typedef enum {
    ERROR_SUCCESS = 0,
    ERROR_INVALID_PARAM = 1,
    ERROR_TIMEOUT = 2,
    ERROR_HARDWARE_FAILURE = 3,
    ERROR_BUFFER_OVERFLOW = 4,
} ErrorCode_t;

// 调用时检查返回值
ErrorCode_t result = uart_send_byte(data);
if (result != ERROR_SUCCESS)
{
    if (result == ERROR_BUFFER_OVERFLOW)
    {
        // 处理缓冲区满
    }
}
```

---

## 6️⃣ AI 开发工作流

### ✅ AI 应该做的

- ✅ 在 .h/.c 文件中插入中文代码注释
- ✅ 在对话窗口总结变更
- ✅ 宏、枚举、结构体都要标注
- ✅ 函数声明要有文档头

### ❌ AI 不应该做的

- ❌ **主动生成 .md 文档**（除非明确要求）
- ❌ 创建单独的文档文件
- ❌ 留下没有注释的代码

### 📋 完成后在对话中总结

```
## 变更总结

**文件修改:**
- src/uart_driver.c - UART 驱动实现
- include/uart_driver.h - 函数声明

**新增功能:**
- uart_init() 初始化函数
- uart_send_byte() 发送函数

**关键设计:**
- DMA 非阻塞传输
- 中断安全缓冲区

**测试:**
- 验证 115200 波特率
- 缓冲区测试到 512 字节
```

---

## 7️⃣ 结构体设计

```c
// 按大小排序（大的在前）以最小化填充
typedef struct {
    uint32_t baud_rate;         // 4 bytes
    uint32_t base_address;      // 4 bytes
    uint16_t buffer_size;       // 2 bytes
    uint8_t data_bits;          // 1 byte
    uint8_t stop_bits;          // 1 byte
} UartConfig_t;
```

---

## 8️⃣ 函数设计规范

### 参数个数限制

```c
// ❌ 参数过多（难以使用）
ErrorCode_t uart_config(uint32_t baud, uint8_t data_bits, uint8_t stop_bits,
                       uint8_t parity, uint16_t timeout, uint8_t retries);

// ✅ 改为结构体（参数清晰）
typedef struct {
    uint32_t baud_rate;      // 波特率
    uint8_t data_bits;       // 数据位
    uint8_t stop_bits;       // 停止位
    uint8_t parity;          // 奇偶校验
    uint16_t timeout_ms;     // 超时时间
    uint8_t retries;         // 重试次数
} UartConfig_t;

ErrorCode_t uart_init(const UartConfig_t *p_config);
```

### 函数长度和嵌套

```c
// ❌ 嵌套过深
ErrorCode_t process(uint8_t *p_data, uint16_t len)
{
    if (len > 0) {
        if (validate(p_data)) {
            if (parse(p_data)) {
                if (execute(p_data)) {  // 4 层嵌套！
                    return ERROR_SUCCESS;
                }
            }
        }
    }
    return ERROR_FAILED;
}

// ✅ 提前返回，减少嵌套
ErrorCode_t process(uint8_t *p_data, uint16_t len)
{
    if (len == 0)
        return ERROR_INVALID;       // 提前返回
    
    if (!validate(p_data))
        return ERROR_VALIDATE;      // 提前返回
    
    if (!parse(p_data))
        return ERROR_PARSE;         // 提前返回
    
    if (!execute(p_data))
        return ERROR_EXECUTE;       // 提前返回
    
    return ERROR_SUCCESS;           // 只有成功到这里
}
```

---

## 9️⃣ 竞态条件和中断安全

### Volatile 关键字 - ISR 访问的变量

```c
// ============================================================================
// ISR 访问的变量必须 volatile
// ============================================================================

/**
 * @brief UART 接收缓冲写指针
 * @isr: uart_rx_isr() 修改
 * @thread-safe: ISR 独占写，主任务读
 */
static volatile uint16_t s_uart_rx_write_idx = 0;

/**
 * @brief DMA 状态标志
 * @isr: dma_complete_isr() 设置
 * @thread-safe: ISR 设置，主任务轮询
 */
static volatile uint8_t s_dma_is_complete = 0;
```

### 临界段保护

```c
// ❌ 竞态条件：ISR 可能中断修改
s_dma_ptr = p_data;
s_dma_len = len;

// ✅ 临界段保护：禁用中断
// 为什么：防止 DMA ISR 在修改过程中中断
DISABLE_INTERRUPTS();
{
    s_dma_ptr = p_data;
    s_dma_len = len;
    s_dma_status = DMA_BUSY;
}
ENABLE_INTERRUPTS();
```

---

## 🔟 缓冲区和内存安全

### 缓冲区溢出防护

```c
// 宏定义缓冲区大小
#define UART_TX_BUFFER_SIZE     (512)   // 发送缓冲大小
#define PACKET_MAX_LEN          (1024)  // 包最大长度

// ❌ 危险：未检查大小
void write_buffer(uint8_t *p_data, uint16_t len)
{
    memcpy(s_buffer, p_data, len);  // 溢出风险！
}

// ✅ 安全：检查所有边界
/**
 * @brief 写缓冲区
 * @param len 长度（1-PACKET_MAX_LEN）
 * @return ERROR_BUFFER_OVERFLOW 如果空间不足
 */
ErrorCode_t write_buffer(const uint8_t *p_data, uint16_t len)
{
    // 参数检查
    if (p_data == NULL || len == 0 || len > PACKET_MAX_LEN)
        return ERROR_INVALID;
    
    // 为什么：必须检查缓冲空间防止溢出
    uint16_t available = UART_TX_BUFFER_SIZE - s_write_idx;
    if (available < len)
        return ERROR_BUFFER_OVERFLOW;
    
    // 现在安全
    memcpy(&s_buffer[s_write_idx], p_data, len);
    s_write_idx += len;
    return ERROR_SUCCESS;
}
```

### 指针检查

```c
// ❌ 危险：未检查指针
void process(uint8_t *p_data)
{
    p_data[0] = 0xFF;  // 如果 p_data == NULL 会崩溃
}

// ✅ 安全：检查指针
ErrorCode_t process(uint8_t *p_data)
{
    // 为什么：NULL 指针解引用导致硬故障
    if (p_data == NULL)
        return ERROR_INVALID_PARAM;
    
    p_data[0] = 0xFF;
    return ERROR_SUCCESS;
}
```

---

## 1️⃣1️⃣ 模块组织和 API 设计

### 清晰的公共 API

```c
// 模块前缀 - 所有公共函数都有
sensor_error_t sensor_init(void);
sensor_error_t sensor_read_temperature(int16_t *p_temp);
sensor_error_t sensor_read_humidity(uint16_t *p_humidity);
void sensor_close(void);

// 一致的 API 设计：
// - 初始化/关闭配对
// - 读操作返回错误码 + 输出指针
// - 查询操作无返回值
// - 函数名清晰说明功能
```

### 资源生命周期管理

```c
// 为什么：获取资源时任何失败都要释放已获取的资源
ErrorCode_t device_open(void)
{
    // 获取资源 1
    ErrorCode_t result = mutex_acquire(g_device_mutex, 100);
    if (result != ERROR_SUCCESS)
        return result;  // 未分配任何资源
    
    // 获取资源 2
    result = uart_init(&g_config);
    if (result != ERROR_SUCCESS)
    {
        // 为什么：反向顺序释放
        mutex_release(g_device_mutex);
        return result;
    }
    
    return ERROR_SUCCESS;
}

void device_close(void)
{
    // 为什么：反向顺序释放（LIFO）防止依赖问题
    uart_close();
    mutex_release(g_device_mutex);
}
```

---

## 📋 代码审查检查清单 - 扩展版

- [ ] **魔法数字**：所有数字都用宏定义代替
- [ ] **大括号**：所有 `{` `}` 都在新行
- [ ] **参数个数**：最多 3-4 个参数（否则用结构体）
- [ ] **函数长度**：最多 ~100 行代码
- [ ] **嵌套深度**：最多 3 层（否则提前返回）
- [ ] **参数检查**：入口检查所有指针和大小
- [ ] **缓冲溢出**：所有写操作都检查大小
- [ ] **返回值检查**：所有错误返回都被检查
- [ ] **Volatile 变量**：ISR 访问的变量都标记 volatile
- [ ] **临界段保护**：共享数据都有保护
- [ ] **资源泄漏**：获取/释放配对
- [ ] **API 一致性**：类似操作有类似的名字和签名
- [ ] **文档完整**：@brief/@param/@return/@note/@isr/@rationale

---

## 🚀 何时应用这个 Skill

- **代码审查**：合并前检查是否符合规范
- **新功能开发**：从一开始就写符合规范的代码
- **AI 辅助开发**：让 AI 按规范生成代码
- **团队评审**：确保整个项目代码一致

---

## 1️⃣2️⃣ 分层架构和函数表模式【新增】

### 为什么要分层

```
❌ 不分层：所有东西混在一起
main.c
 ├─ UART 驱动
 ├─ ADC 驱动  
 ├─ 状态机
 ├─ 业务逻辑
 └─ 硬件寄存器操作

✅ 分层：清晰的依赖
APP 层 (应用逻辑)
   ↓
业务层 (算法、状态机)
   ↓
驱动层 (设备驱动)
   ↓
底层 HAL (硬件初始化)
```

### 函数表 - 解决 Switch 爆炸

```c
// ❌ 不好：巨大的 Switch
void handle_command(uint8_t cmd, uint8_t *p_data, uint16_t len)
{
    switch (cmd)
    {
        case 0x01:  // 启动命令
            if (len >= 2) { /* ... 20 行逻辑 ... */ }
            break;
        case 0x02:  // 停止命令
            if (...) { /* ... 20 行逻辑 ... */ }
            break;
        // ... 还有 20 多个 case（可能 500+ 行）
    }
}

// ✅ 好：函数表 + 独立处理函数
typedef ErrorCode_t (*cmd_handler_t)(const uint8_t *p_data, uint16_t len);

// 每个命令的独立处理函数
static ErrorCode_t cmd_start(const uint8_t *p_data, uint16_t len) { /* 20 行 */ }
static ErrorCode_t cmd_stop(const uint8_t *p_data, uint16_t len) { /* 15 行 */ }

// 命令处理表 - 集中管理
static const struct {
    uint8_t cmd_id;
    cmd_handler_t handler;
} g_cmd_table[] = {
    { 0x01, cmd_start },   // 新增命令只需加一行！
    { 0x02, cmd_stop },
};

// 清晰的分发函数
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

### 虚接口 - 设备独立

```c
// 定义统一接口
typedef struct {
    ErrorCode_t (*init)(void);
    ErrorCode_t (*send)(const uint8_t *p_data, uint16_t len);
    ErrorCode_t (*recv)(uint8_t *p_data, uint16_t *p_len);
    const char *name;
} CommInterface_t;

// 不同设备实现
static const CommInterface_t g_uart_device = {
    .init = uart_init,
    .send = uart_send,
    .recv = uart_recv,
    .name = "UART",
};

static const CommInterface_t g_spi_device = {
    .init = spi_init,
    .send = spi_transmit,
    .recv = spi_receive,
    .name = "SPI",
};

// 应用代码：改这一行就换设备
CommInterface_t *p_comm = &g_uart_device;  // 用 UART
// CommInterface_t *p_comm = &g_spi_device;  // 用 SPI

void send_data(const uint8_t *p_data, uint16_t len) {
    p_comm->send(p_data, len);  // 业务逻辑无需改
}
```

### 状态机 - 清晰的转换规则

```c
// 定义状态和事件
typedef enum { STATE_IDLE, STATE_RUN, STATE_ERROR } State;
typedef enum { EVENT_START, EVENT_STOP, EVENT_FAIL } Event;

// 每个状态的处理函数
static State handle_idle(Event event) {
    switch (event) {
        case EVENT_START: return STATE_RUN;
        default: return STATE_IDLE;
    }
}

static State handle_run(Event event) {
    switch (event) {
        case EVENT_STOP: return STATE_IDLE;
        case EVENT_FAIL: return STATE_ERROR;
        default: return STATE_RUN;
    }
}

// 状态处理表
typedef State (*state_handler_t)(Event);
static const state_handler_t g_state_handlers[] = {
    handle_idle,   // STATE_IDLE
    handle_run,    // STATE_RUN
    handle_error,  // STATE_ERROR
};

// 处理事件
void on_event(Event event) {
    State next = g_state_handlers[g_current_state](event);
    if (next != g_current_state) {
        // 转换发生
        on_state_changed(g_current_state, next);
        g_current_state = next;
    }
}
```

### 检查清单

- [ ] **避免巨大 Switch**：switch 超过 10 个 case → 改用函数表
- [ ] **避免到处 if 硬件判断**：有多个 if 判断设备 → 改用虚接口
- [ ] **避免状态嵌套**：状态转换超过 2 层嵌套 → 改用状态表
- [ ] **处理函数独立**：每个函数 < 50 行，可单独测试
- [ ] **表格集中管理**：新增功能无需修改分发函数
- [ ] **无循环依赖**：依赖都是单向的（从上到下）
- [ ] **设备可切换**：改一行配置就能换设备

---

**Version:** 3.0 | **Updated:** 2026-10-13 | **新增分层架构规范**

