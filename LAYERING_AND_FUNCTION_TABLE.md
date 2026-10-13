---
title: 分层架构 & 函数表模式 - 实战指南
---

# 分层架构和函数表模式 - 完整实战指南

**这是解耦和提升代码可维护性的关键！**

---

## 第一部分：分层架构基础

### 为什么需要分层

**问题 - 没有分层的代码**:

```
main.c (所有逻辑混在一起)
├─ UART 初始化
├─ ADC 初始化
├─ 状态机
├─ 命令处理
├─ 硬件寄存器操作
├─ 业务逻辑
└─ 错误处理
```

**问题**:
- ❌ 修改硬件时整个系统受影响
- ❌ 不同开发者容易冲突
- ❌ 难以单元测试
- ❌ 代码难以复用

**解决 - 分层架构**:

```
APP 层 (app_freq_detector.c)
   └─ 依赖
BSW 业务层 (bsw_freq_analyzer.c、bsw_state_machine.c)
   └─ 依赖
驱动层 (uart_driver.c、adc_driver.c)
   └─ 依赖
底层 (hal_init.c、hal_interrupt.c)
```

**好处**:
- ✅ 清晰的依赖关系（单向）
- ✅ 修改硬件只改底层，业务层不变
- ✅ 可独立开发和测试
- ✅ 易于多人协作

---

## 第二部分：解决 Switch 地狱 - 函数表模式

### 问题 1: 命令处理的 Switch 爆炸

**❌ 不好的做法 - Switch 中添加所有逻辑**:

```c
void handle_command(uint8_t cmd, uint8_t *p_data, uint16_t len)
{
    switch (cmd)
    {
        case 0x01:  // 启动命令
            if (len >= 2)
            {
                uint16_t delay = (p_data[0] << 8) | p_data[1];
                if (delay > 0 && delay < 60000)
                {
                    start_system(delay);
                    send_ack(0x01);
                }
                else
                {
                    send_error(ERROR_INVALID_PARAM);
                }
            }
            break;
            
        case 0x02:  // 停止命令
            if (check_permissions())
            {
                stop_system();
                send_ack(0x02);
            }
            break;
            
        // ... 还有 20 多个 case，每个都有复杂逻辑
    }
}

// 问题：
// 1. 函数太大（可能 500+ 行）
// 2. 难以维护和测试
// 3. 添加新命令要修改这个大函数
// 4. 命令的参数检查分散在各处
```

**✅ 好的做法 - 函数表模式**:

```c
// ============================================================================
// 第一步：定义命令处理函数的原型
// ============================================================================

typedef ErrorCode_t (*cmd_handler_t)(const uint8_t *p_data, uint16_t len);

// ============================================================================
// 第二步：实现每个命令的处理函数（独立、清晰）
// ============================================================================

/**
 * @brief 处理启动命令（0x01）
 * 为什么分离：
 * - 逻辑独立，易于理解
 * - 可以单独编写和测试
 * - 修改不影响其他命令
 */
static ErrorCode_t cmd_start_handler(const uint8_t *p_data, uint16_t len)
{
    // 参数检查
    if (len < 2)
        return ERROR_INVALID_LENGTH;
    
    // 提取参数
    uint16_t delay = (p_data[0] << 8) | p_data[1];
    
    // 参数验证
    if (delay == 0 || delay >= 60000)
        return ERROR_INVALID_PARAM;
    
    // 执行操作
    ErrorCode_t result = start_system(delay);
    
    // 返回结果（由分发函数处理响应）
    return result;
}

/**
 * @brief 处理停止命令（0x02）
 */
static ErrorCode_t cmd_stop_handler(const uint8_t *p_data, uint16_t len)
{
    // 权限检查
    if (!check_permissions())
        return ERROR_PERMISSION_DENIED;
    
    return stop_system();
}

/**
 * @brief 处理查询状态命令（0x03）
 */
static ErrorCode_t cmd_query_handler(const uint8_t *p_data, uint16_t len)
{
    return query_system_status();
}

// ... 其他命令处理函数

// ============================================================================
// 第三步：构建命令处理表
// ============================================================================

/**
 * @brief 命令表项
 */
typedef struct {
    uint8_t cmd_id;             // 命令 ID
    cmd_handler_t handler;      // 处理函数
    const char *p_name;         // 命令名称（调试、日志用）
    uint16_t min_data_len;      // 最小数据长度
} CmdEntry_t;

/**
 * @brief 命令处理表 - 集中管理
 * 为什么用表格：
 * 1. 一眼看清所有支持的命令
 * 2. 添加新命令只需添加表项
 * 3. 无需修改分发函数
 * 4. 便于生成协议文档
 */
static const CmdEntry_t g_cmd_table[] =
{
    // cmd_id, handler,              name,           min_len
    { 0x01,    cmd_start_handler,   "START",        2 },
    { 0x02,    cmd_stop_handler,    "STOP",         0 },
    { 0x03,    cmd_query_handler,   "QUERY",        0 },
    // 新增命令时只需添加：
    // { 0x04,    cmd_new_handler,     "NEW_CMD",      1 },
};

#define CMD_TABLE_SIZE  (sizeof(g_cmd_table) / sizeof(g_cmd_table[0]))

// ============================================================================
// 第四步：简洁的命令分发函数
// ============================================================================

/**
 * @brief 命令分发
 * @param cmd 命令 ID
 * @param p_data 命令数据
 * @param len 数据长度
 * @return ErrorCode_t - 处理结果
 * 
 * 为什么这样做：
 * - 分发函数简单清晰，无复杂逻辑
 * - 所有命令处理由表管理
 * - 易于维护和扩展
 */
ErrorCode_t dispatch_command(uint8_t cmd, const uint8_t *p_data, uint16_t len)
{
    // 查表找到对应的处理函数
    for (uint16_t i = 0; i < CMD_TABLE_SIZE; i++)
    {
        if (g_cmd_table[i].cmd_id == cmd)
        {
            const CmdEntry_t *p_entry = &g_cmd_table[i];
            
            // 检查最小长度
            if (len < p_entry->min_data_len)
                return ERROR_INVALID_LENGTH;
            
            // 调用处理函数
            ErrorCode_t result = p_entry->handler(p_data, len);
            
            // 发送响应
            if (result == ERROR_SUCCESS)
                send_response(cmd, RESPONSE_ACK);
            else
                send_response(cmd, result);
            
            return result;
        }
    }
    
    // 未找到命令
    send_response(cmd, ERROR_UNKNOWN_COMMAND);
    return ERROR_UNKNOWN_COMMAND;
}

// 使用：在 UART 接收中断中
void uart_rx_complete_handler(uint8_t *p_frame, uint16_t len)
{
    uint8_t cmd = p_frame[0];
    uint8_t *p_data = &p_frame[1];
    uint16_t data_len = len - 1;
    
    // 分发到对应的处理函数
    dispatch_command(cmd, p_data, data_len);
}
```

**对比**:

| 方面 | Switch 方法 | 函数表方法 |
|------|-----------|---------|
| 函数长度 | 很长（500+ 行） | 短小（每个函数 20-30 行） |
| 添加新命令 | 修改分发函数 | 只加表项 |
| 代码复用 | 困难 | 容易 |
| 单元测试 | 难（不能隔离） | 容易（独立测试） |
| 可读性 | 差 | 好 |
| 维护性 | 差 | 好 |

---

## 第三部分：虚接口模式 - 设备适配

### 场景：不同的通信设备（UART、SPI、I2C）

**❌ 不好的做法 - 到处 if 语句**:

```c
// 配置阶段
#define COMM_DEVICE_TYPE 0  // 0=UART, 1=SPI, 2=I2C

void init_communication(void)
{
    if (COMM_DEVICE_TYPE == 0)
        uart_init();
    else if (COMM_DEVICE_TYPE == 1)
        spi_init();
    else
        i2c_init();
}

void send_data(uint8_t *p_data, uint16_t len)
{
    if (COMM_DEVICE_TYPE == 0)
        uart_send(p_data, len);
    else if (COMM_DEVICE_TYPE == 1)
        spi_transmit(p_data, len);
    else
        i2c_write(p_data, len);
}

// 问题：
// 1. 到处都要加 if 语句
// 2. 修改设备需要改很多地方
// 3. 业务代码耦合硬件选择
```

**✅ 好的做法 - 虚接口**:

```c
// ============================================================================
// 第一步：定义统一的设备接口
// ============================================================================

/**
 * @brief 设备操作接口（类似 C++ 的虚函数表）
 */
typedef struct {
    // 函数指针
    ErrorCode_t (*init)(void);
    ErrorCode_t (*open)(void);
    void (*close)(void);
    ErrorCode_t (*send)(const uint8_t *p_data, uint16_t len);
    ErrorCode_t (*recv)(uint8_t *p_data, uint16_t *p_len);
    
    // 元数据
    const char *p_name;
} CommDeviceInterface_t;

// ============================================================================
// 第二步：每个设备实现接口
// ============================================================================

// UART 实现
static ErrorCode_t uart_iface_init(void) { return uart_init(&g_uart_config); }
static ErrorCode_t uart_iface_open(void) { return uart_start(); }
static void uart_iface_close(void) { uart_stop(); }
static ErrorCode_t uart_iface_send(const uint8_t *p_data, uint16_t len)
{
    return uart_send(p_data, len);
}
static ErrorCode_t uart_iface_recv(uint8_t *p_data, uint16_t *p_len)
{
    return uart_receive(p_data, p_len);
}

// UART 接口实例
static const CommDeviceInterface_t g_uart_device =
{
    .init = uart_iface_init,
    .open = uart_iface_open,
    .close = uart_iface_close,
    .send = uart_iface_send,
    .recv = uart_iface_recv,
    .p_name = "UART",
};

// SPI 实现（类似）
static const CommDeviceInterface_t g_spi_device =
{
    .init = spi_iface_init,
    .open = spi_iface_open,
    .close = spi_iface_close,
    .send = spi_iface_send,
    .recv = spi_iface_recv,
    .p_name = "SPI",
};

// I2C 实现（类似）
static const CommDeviceInterface_t g_i2c_device =
{
    .init = i2c_iface_init,
    .open = i2c_iface_open,
    .close = i2c_iface_close,
    .send = i2c_iface_send,
    .recv = i2c_iface_recv,
    .p_name = "I2C",
};

// ============================================================================
// 第三步：通用设备管理层
// ============================================================================

/**
 * @brief 通用设备对象 - 应用层只与这个交互
 */
typedef struct {
    const CommDeviceInterface_t *p_iface;  // 指向具体设备
} CommDevice_t;

static CommDevice_t g_comm_device;

// 设备操作 - 应用代码只调用这些
ErrorCode_t comm_device_init(const CommDeviceInterface_t *p_iface)
{
    if (p_iface == NULL || p_iface->init == NULL)
        return ERROR_INVALID_PARAM;
    
    g_comm_device.p_iface = p_iface;
    return p_iface->init();
}

ErrorCode_t comm_device_send(const uint8_t *p_data, uint16_t len)
{
    if (g_comm_device.p_iface == NULL || g_comm_device.p_iface->send == NULL)
        return ERROR_NOT_INITIALIZED;
    
    return g_comm_device.p_iface->send(p_data, len);
}

ErrorCode_t comm_device_recv(uint8_t *p_data, uint16_t *p_len)
{
    if (g_comm_device.p_iface == NULL || g_comm_device.p_iface->recv == NULL)
        return ERROR_NOT_INITIALIZED;
    
    return g_comm_device.p_iface->recv(p_data, p_len);
}

// ============================================================================
// 第四步：应用代码 - 完全不知道硬件细节
// ============================================================================

int main(void)
{
    // 为什么：只需改这一行就能换设备
    // UART：comm_device_init(&g_uart_device);
    // SPI：comm_device_init(&g_spi_device);
    // I2C：comm_device_init(&g_i2c_device);
    
    ErrorCode_t result = comm_device_init(&g_uart_device);
    if (result != ERROR_SUCCESS)
        return 1;
    
    // 业务代码 - 无需关心用的什么设备
    uint8_t tx_data[] = "Hello";
    comm_device_send(tx_data, sizeof(tx_data));
    
    uint8_t rx_data[64];
    uint16_t rx_len = sizeof(rx_data);
    comm_device_recv(rx_data, &rx_len);
    
    return 0;
}
```

**要换设备**:
- ✅ 只改 `main()` 中的一行
- ✅ 所有业务逻辑代码无需改动
- ✅ 驱动层代码完全独立

---

## 第四部分：状态机模式 - 管理复杂流程

### 场景：设备有多个状态，不同状态不同行为

**❌ 不好的做法 - 嵌套 if/else**:

```c
typedef enum { IDLE, INIT, RUN, ERROR } State;
static State g_state = IDLE;

void process_event(int event)
{
    if (g_state == IDLE)
    {
        if (event == START)
        {
            g_state = INIT;
            init_system();
        }
    }
    else if (g_state == INIT)
    {
        if (event == INITIALIZED)
        {
            g_state = RUN;
            start_operation();
        }
        else if (event == ERROR)
        {
            g_state = ERROR;
            handle_error();
        }
    }
    else if (g_state == RUN)
    {
        // ... 更多嵌套
    }
    // 问题：难以维护、易出错
}
```

**✅ 好的做法 - 状态表**:

```c
// ============================================================================
// 第一步：定义状态和事件
// ============================================================================

typedef enum {
    STATE_IDLE,         // 空闲
    STATE_INITIALIZING, // 初始化中
    STATE_RUNNING,      // 运行中
    STATE_ERROR,        // 错误
    STATE_MAX,
} SystemState_t;

typedef enum {
    EVENT_START,
    EVENT_STOP,
    EVENT_ERROR,
    EVENT_RECOVER,
} SystemEvent_t;

// ============================================================================
// 第二步：每个状态的处理函数
// ============================================================================

/**
 * @brief IDLE 状态处理
 */
static SystemState_t handle_idle(SystemEvent_t event)
{
    switch (event)
    {
        case EVENT_START:
            log_info("IDLE -> INITIALIZING");
            return STATE_INITIALIZING;
        default:
            return STATE_IDLE;
    }
}

/**
 * @brief INITIALIZING 状态处理
 */
static SystemState_t handle_initializing(SystemEvent_t event)
{
    switch (event)
    {
        case EVENT_START:
            log_info("INITIALIZING -> RUNNING");
            return STATE_RUNNING;
        case EVENT_ERROR:
            log_error("INITIALIZING -> ERROR");
            return STATE_ERROR;
        default:
            return STATE_INITIALIZING;
    }
}

/**
 * @brief RUNNING 状态处理
 */
static SystemState_t handle_running(SystemEvent_t event)
{
    switch (event)
    {
        case EVENT_STOP:
            log_info("RUNNING -> IDLE");
            return STATE_IDLE;
        case EVENT_ERROR:
            log_error("RUNNING -> ERROR");
            return STATE_ERROR;
        default:
            return STATE_RUNNING;
    }
}

/**
 * @brief ERROR 状态处理
 */
static SystemState_t handle_error(SystemEvent_t event)
{
    switch (event)
    {
        case EVENT_RECOVER:
            log_info("ERROR -> IDLE");
            return STATE_IDLE;
        default:
            return STATE_ERROR;
    }
}

// ============================================================================
// 第三步：状态处理表
// ============================================================================

typedef SystemState_t (*state_handler_t)(SystemEvent_t);

/**
 * @brief 状态处理表 - 集中管理所有状态转换
 * 好处：
 * 1. 一眼看清所有状态和转换规则
 * 2. 易于绘制状态图
 * 3. 添加新状态只需加表项
 */
static const state_handler_t g_state_handlers[STATE_MAX] =
{
    handle_idle,          // STATE_IDLE
    handle_initializing,  // STATE_INITIALIZING
    handle_running,       // STATE_RUNNING
    handle_error,         // STATE_ERROR
};

// ============================================================================
// 第四步：状态机核心
// ============================================================================

static SystemState_t g_current_state = STATE_IDLE;

/**
 * @brief 处理事件并转换状态
 */
void fsm_on_event(SystemEvent_t event)
{
    if (g_current_state >= STATE_MAX)
    {
        g_current_state = STATE_IDLE;
        return;
    }
    
    // 为什么：通过表查询当前状态的处理函数
    state_handler_t handler = g_state_handlers[g_current_state];
    
    if (handler != NULL)
    {
        SystemState_t next_state = handler(event);
        
        if (next_state != g_current_state)
        {
            // 状态转换发生
            on_state_transition(g_current_state, next_state, event);
            g_current_state = next_state;
        }
    }
}

SystemState_t fsm_get_state(void)
{
    return g_current_state;
}

// 使用
int main(void)
{
    fsm_on_event(EVENT_START);      // IDLE -> INITIALIZING
    fsm_on_event(EVENT_START);      // INITIALIZING -> RUNNING
    fsm_on_event(EVENT_STOP);       // RUNNING -> IDLE
    fsm_on_event(EVENT_ERROR);      // 无状态转换（IDLE 不处理 ERROR）
    
    return 0;
}
```

---

## 最佳实践总结

### 何时用函数表

| 场景 | 用法 | 好处 |
|------|------|------|
| 命令处理 | 按 cmd_id 查表调用处理函数 | 解耦、易扩展 |
| 设备驱动 | 不同设备实现同一接口 | 设备独立、可切换 |
| 状态机 | 按状态查表调用处理函数 | 规则清晰、易维护 |
| 事件系统 | 按事件类型查表调用回调 | 低耦合、模块化 |
| 协议处理 | 按消息类型查表调用解析器 | 易添加新消息 |

### 函数表设计要点

```c
// 1. 定义清晰的函数原型
typedef ErrorCode_t (*handler_t)(const uint8_t *p_data, uint16_t len);

// 2. 实现独立的处理函数
static ErrorCode_t handle_cmd_01(...) { ... }
static ErrorCode_t handle_cmd_02(...) { ... }

// 3. 构建表格（集中管理）
static const struct {
    uint8_t id;
    handler_t handler;
    const char *name;
} g_handler_table[] = {
    { 0x01, handle_cmd_01, "CMD_01" },
    { 0x02, handle_cmd_02, "CMD_02" },
};

// 4. 简洁的查表分发函数
ErrorCode_t dispatch(uint8_t id, ...) {
    for (int i = 0; i < TABLE_SIZE; i++) {
        if (g_handler_table[i].id == id)
            return g_handler_table[i].handler(...);
    }
    return ERROR_NOT_FOUND;
}
```

### 检查清单

- [ ] 是否有大的 switch/if-else？→ 改用函数表
- [ ] 是否有到处的 if 判断硬件类型？→ 改用虚接口
- [ ] 是否有复杂的状态转换嵌套？→ 改用状态表
- [ ] 处理函数是否独立且简洁（< 50 行）？→ 是
- [ ] 表格是否集中、易于查看？→ 是
- [ ] 新增功能是否不需要修改分发函数？→ 是
- [ ] 是否避免了循环依赖？→ 是

