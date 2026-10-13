---
name: embedded-c-conventions
description: Enforce embedded C coding standards for clarity and maintainability. Apply this skill whenever reviewing or writing embedded C code, or when the user mentions code quality, C conventions, naming standards, variable naming, comments, error handling, function design, or code style. Check for magic numbers replaced with macros, proper bracket formatting (braces on new lines), consistent naming conventions, comprehensive comments, proper error handling, global variable management, and consistency with embedded systems best practices.
---

# Embedded C Coding Conventions

This skill enforces coding standards specifically designed for embedded systems development in C. The goal is to maximize code clarity, maintainability, safety, and team collaboration in resource-constrained environments.

## Core Conventions

### 1. Use Macros Instead of Magic Numbers

**Rationale**: Embedded code often contains hardware addresses, timing constants, and configuration values. Using named macros makes the code self-documenting and simplifies maintenance when values need to change.

**Examples**:

❌ **Bad**:
```c
// Hardware initialization
GPIOA->ODR = 0x04;           // Set GPIO pin without explanation
delay(1000);                  // What is this 1000? Milliseconds? Microseconds?
if (adc_value > 2048) { ... } // What is 2048? ADC max? Threshold?
```

✅ **Good**:
```c
#define GPIO_PIN_LED_MASK       (0x04)
#define INIT_DELAY_MS           (1000)
#define ADC_THRESHOLD_VOLTAGE   (2048)

GPIOA->ODR = GPIO_PIN_LED_MASK;
delay(INIT_DELAY_MS);
if (adc_value > ADC_THRESHOLD_VOLTAGE) { ... }
```

**Macro Naming Convention**:
- Use UPPERCASE_WITH_UNDERSCORES
- Prefix with module/subsystem name: `UART_BAUD_RATE`, `TIMER_PRESCALE`, `ADC_SAMPLE_COUNT`
- Group related macros by subsystem

**When NOT to use macros**:
- Loop counters and temporary values in function scope
- Simple integer literals in obvious context (e.g., `i++` where `i` is a loop index)
- Struct/array initialization with clear, one-time values

---

### 2. Bracket Formatting - Braces on New Lines

**Rationale**: Embedded code is often reviewed on small screens and in documentation. New-line braces improve readability and reduce merge conflicts in version control. This is the standard for embedded systems codebases (Linux kernel, MISRA C guidelines).

**Format**:
- Opening brace `{` goes on a new line
- Closing brace `}` goes on its own line
- Applies to: `if`, `else`, `for`, `while`, `do`, `switch`, `function definitions`

**Examples**:

❌ **Bad**:
```c
void init_system() {
    if (uart_ready) {
        configure_uart();
    } else {
        error_handler();
    }
}

for (int i = 0; i < MAX_ITERATIONS; i++) {
    process_data(i);
}
```

✅ **Good**:
```c
void init_system()
{
    if (uart_ready)
    {
        configure_uart();
    }
    else
    {
        error_handler();
    }
}

for (int i = 0; i < MAX_ITERATIONS; i++)
{
    process_data(i);
}
```

**Special Cases**:
- `else if` stays on the same line as the closing brace of the previous `if`:
  ```c
  if (condition_a)
  {
      handle_a();
  }
  else if (condition_b)
  {
      handle_b();
  }
  else
  {
      handle_default();
  }
  ```

- `switch` statement:
  ```c
  switch (state)
  {
      case STATE_INIT:
          initialize();
          break;
      case STATE_RUN:
          execute();
          break;
      default:
          error();
  }
  ```

---

### 3. Variable and Function Naming Conventions

**Rationale**: Consistent naming makes code self-documenting and prevents confusion. Embedded systems often operate across hardware/software boundaries, so clear naming distinguishes between different contexts (local, global, static, hardware registers, etc.).

**Naming Rules by Scope and Type**:

| Entity Type | Convention | Example | Notes |
|------------|-----------|---------|-------|
| Global variables | `g_module_name` | `g_uart_state`, `g_adc_buffer` | Prefix `g_` signals global scope |
| Static variables | `s_module_name` | `s_timer_config`, `s_init_flag` | Prefix `s_` signals static/file scope |
| Local variables | `lowercase_with_underscores` | `temperature_raw`, `error_count` | Descriptive, no prefix needed |
| Pointers | `p_type_name` | `p_uart_config`, `p_sensor_data` | Prefix `p_` indicates pointer |
| Function names | `lowercase_with_underscores` | `uart_send_byte()`, `sensor_read_temperature()` | Module prefix: `MODULE_action()` |
| Constants/Macros | `UPPERCASE_WITH_UNDERSCORES` | `UART_BAUD_RATE`, `ADC_BUFFER_SIZE` | Already covered in section 1 |
| Struct typedef | `ModuleName_t` | `UartConfig_t`, `SensorData_t` | Suffix `_t`, PascalCase with underscore |
| Struct members | `lowercase_with_underscores` | `config->baud_rate`, `data->temperature` | Lowercase, snake_case |
| Boolean variables | `is_/has_/should_` prefix | `is_ready`, `has_error`, `should_retry` | Clear intent |

**Examples**:

❌ **Bad**:
```c
int x, y, z;                    // Meaningless names
void Init();                    // Unclear scope
static int state;               // No indication it's static
int *ptr;                       // No indication it's a pointer
struct Config {
    int bd, fc, pc;             // Abbreviated members
} config;
```

✅ **Good**:
```c
uint16_t temperature_raw;
uint16_t humidity_raw;
uint16_t error_count;

void uart_init();               // Module prefix + action
void sensor_configure();

static uint8_t s_uart_tx_buffer[256];
static bool s_is_initialized;

int *p_adc_data;
void (*p_callback_func)(void);

typedef struct {
    uint32_t baud_rate;
    uint8_t data_bits;
    uint8_t parity;
} UartConfig_t;
```

---

### 4. Comment and Documentation Standards

**Rationale**: Balance between clarity and code readability. Only document what's necessary. Focus comments on **declarations** (.h files) and **critical sections** in implementation.

#### 4.1 Header File (.h) Documentation - REQUIRED

Every declaration in .h files **must** be documented:

```c
/**
 * @brief Initialize UART peripheral with specified configuration
 * @param p_config Pointer to UartConfig_t with baud rate, data bits, parity
 * @return ErrorCode_t - ERROR_SUCCESS on success, error code on failure
 * @note Blocks up to 100ms during initialization
 */
ErrorCode_t uart_init(const UartConfig_t *p_config);

/**
 * @brief Send single byte via UART
 * @param byte Byte to transmit (0-255)
 * @return ErrorCode_t - ERROR_SUCCESS or ERROR_BUFFER_OVERFLOW
 * @note Non-blocking; queues byte for DMA transmission
 */
ErrorCode_t uart_send_byte(uint8_t byte);

/**
 * @brief Get current UART state
 * @return UartState_t Current state (IDLE, TRANSMITTING, ERROR)
 */
UartState_t uart_get_state(void);
```

#### 4.2 Macro, Enum, and Struct Definitions - Document Each Member

**Macro group header (中文注释):**
```c
// ============================================================================
// UART 配置宏定义
// ============================================================================

#define UART_BAUD_RATE_115200      (8)      // 波特率 115200 的 BRR 值 @ 16MHz
#define UART_INIT_DELAY_MS         (100)    // 电压稳定等待时间
#define UART_BUFFER_SIZE           (256)    // 接收缓冲区大小
```

**Enum - document each state (中文注释):**
```c
typedef enum {
    UART_STATE_UNINITIALIZED,    // 未初始化
    UART_STATE_READY,            // 就绪，可收发
    UART_STATE_TRANSMITTING,     // 正在通过 DMA 发送
    UART_STATE_ERROR             // 错误状态
} UartState_t;
```

**Struct - document each field (中文注释):**
```c
typedef struct {
    uint32_t baud_rate;          // 波特率（9600、115200 等）
    uint16_t data_bits;          // 数据位（8 或 9）
    uint8_t stop_bits;           // 停止位（1 或 2）
    uint8_t parity;              // 奇偶校验（0=无、1=偶数、2=奇数）
    void (*on_byte_received)(uint8_t);  // 字节接收回调
} UartConfig_t;
```

#### 4.3 Implementation (.c) - Comment Only Critical Sections

**Only comment when (中文注释示例):**
- ❌ 代码含义清晰时不需要注释
- ✅ 为什么采用特定方案（如硬件时序、临时方案）
- ✅ 非明显的控制流或关键算法逻辑
- ✅ 全局/静态变量访问的原因
- ✅ 有副作用的硬件操作

**Good minimal comments (中文):**
```c
// 为什么：禁用中断以防止临界段中的寄存器更新竞态条件
DISABLE_INTERRUPTS();
{
    TIMER->CR1 = timer_config_value;
    TIMER->CR2 = timer_prescale_value;
}
ENABLE_INTERRUPTS();

// 为什么：轮询硬件状态而非忙轮询，允许调度器运行其他任务
uint32_t deadline = get_ticks() + UART_READY_TIMEOUT_MS;
while (!is_uart_ready() && get_ticks() < deadline)
{
    yield_to_scheduler();
}

// 硬件特定时序：必须先读状态再读数据寄存器
// 以清除待处理中断标志
uint8_t status = UART->SR;
uint8_t data = UART->DR;
```

**Avoid over-commenting (中文):**
```c
❌ int x = 0;        // 初始化 x 为 0（太明显）
❌ i++;              // i 自增（太明显）
❌ result = a + b;   // a 加 b 存入结果（太明显）

✅ 当变量名清晰时，明显的代码不需要注释
```

---

### 5. Global Variable and Static Variable Management

**Rationale**: Global and static variables can cause confusion, race conditions, and make code hard to test. Use them intentionally and document why.

**Rules (中文示例):**
1. **最小化全局变量**：仅用于硬件寄存器、系统状态或共享资源
2. **需要理由注释**：说明为什么必须是全局/静态而不是局部
3. **优先使用文件作用域静态变量**：命名空间隔离，避免污染全局
4. **初始化要明确**：所有全局/静态变量都必须显式初始化

**Examples (中文注释):**

❌ **Bad**:
```c
int buffer[256];              // 全局缓冲区 - 为什么？
int current_index;            // 全局状态 - 没有保护
bool error_flag;              // 全局 - 可从任何地方设置
```

✅ **Good**:
```c
// @brief UART 发送缓冲区（DMA 使用）
// @rationale: DMA 控制器需要固定内存地址，必须是全局
// @thread-safe: 由 ISR 中的中断禁用保护
static uint8_t s_uart_tx_buffer[UART_TX_BUFFER_SIZE] = {0};

// @brief 发送缓冲区写指针
// @rationale: ISR 和主任务都访问，必须是静态以控制可见性
// @thread-safe: 32 位架构上的原子读写
static volatile uint16_t s_uart_tx_index = 0;

// @brief 模块级状态机
// @rationale: 需要在函数调用间持久化，必须是静态
// @note: 仅通过 uart_get_state() 和 uart_set_state() 访问
static enum {
    UART_STATE_UNINITIALIZED,
    UART_STATE_READY,
    UART_STATE_TRANSMITTING,
    UART_STATE_ERROR
} s_uart_state = UART_STATE_UNINITIALIZED;
```

---

### 6. Error Handling and Return Value Conventions

**Rationale**: Embedded systems must handle errors gracefully. Consistent error codes make error handling predictable.

**Error Return Pattern**:
```c
// Define error codes
typedef enum {
    ERROR_SUCCESS           = 0,    // Operation succeeded
    ERROR_INVALID_PARAM     = 1,    // Invalid parameter passed
    ERROR_TIMEOUT           = 2,    // Operation timed out
    ERROR_HARDWARE_FAILURE  = 3,    // Hardware not responding
    ERROR_BUFFER_OVERFLOW   = 4,    // Buffer full
    ERROR_NOT_INITIALIZED   = 5,    // Subsystem not initialized
} ErrorCode_t;

// Function always returns error code, output via pointer
ErrorCode_t uart_read_byte(uint8_t *p_byte);
ErrorCode_t sensor_get_temperature(int16_t *p_temp_celsius);

// Usage:
uint8_t received_byte;
ErrorCode_t result = uart_read_byte(&received_byte);
if (result != ERROR_SUCCESS)
{
    // Handle error based on specific code
    if (result == ERROR_TIMEOUT)
    {
        log_error("UART read timeout");
    }
    else if (result == ERROR_NOT_INITIALIZED)
    {
        log_error("UART not initialized");
    }
}
```

---

### 7. Structure and Type Definition Standards

**Rationale**: Well-organized structures make interfaces clear and catch bugs at compile time.

**Rules**:
1. Group related fields together
2. Order by size (largest first) to minimize padding
3. Use typedef for easier forward declarations
4. Document each field

```c
typedef struct {
    // Configuration (8 bytes)
    uint32_t baud_rate;        // Bits per second (9600, 115200, etc.)
    uint16_t data_bits;        // 8 or 9
    uint8_t stop_bits;         // 1 or 2
    uint8_t parity;            // 0=none, 1=even, 2=odd
    
    // Callbacks (16 bytes on 32-bit systems)
    void (*on_byte_received)(uint8_t);
    void (*on_tx_complete)(void);
    
    // Buffers (dynamic, allocated separately)
    uint8_t *p_tx_buffer;
    uint16_t tx_buffer_size;
    uint8_t *p_rx_buffer;
    uint16_t rx_buffer_size;
} UartConfig_t;
```

---

### 8. AI-Assisted Development Workflow

**Rationale**: When using AI to generate code, follow a specific workflow to maintain team clarity and avoid documentation clutter.

#### 8.1 After AI Generates Code

**DO:**
- ✅ Have AI insert code comments directly into the generated code (.h and .c files)
- ✅ **Summarize changes in the chat** with:
  - What was added/modified
  - Key files affected
  - Functional changes or bug fixes
  - Any manual adjustments made
- ✅ Ask AI to explain complex logic if needed

**DON'T:**
- ❌ Generate markdown documentation files (.md) unless **explicitly requested**
- ❌ Create separate documentation outside the code
- ❌ Leave generated code without inline comments

#### 8.2 Documentation Summary Template (Chat Only)

Keep summaries brief and in the chat window:

```
## Changes Summary

**Files modified:**
- src/uart_driver.c - Added DMA transmit support
- include/uart_driver.h - New function uart_send_dma()

**What was added:**
- DMA controller initialization and configuration
- Non-blocking transmit via DMA

**Key functions:**
- uart_send_dma() - Queue data for DMA transmission
- uart_dma_isr() - DMA completion interrupt handler

**Testing notes:**
- Verified with 115200 baud rate
- Tested buffer sizes up to 512 bytes
```

#### 8.3 Exception: When to Generate Documentation

Generate .md files **ONLY** if:
- User explicitly requests: "Create documentation" or "Generate README"
- Project requires formal specification or API documentation
- Preparing documentation for external teams or official release

Otherwise: **Keep all info in code comments and chat summaries**

### 9. Function Design Guidelines

**Rationale**: Embedded systems have strict resource constraints. Functions must be designed for efficiency, safety, and predictability.

#### 9.1 Function Parameter Constraints

**Rules**:
- Maximum 3-4 parameters per function
- If more parameters needed, use a struct
- Always use pointers for output parameters
- Document parameter constraints and ranges

**Examples**:

❌ **Too many parameters:**
```c
ErrorCode_t uart_config(uint32_t baud, uint8_t data_bits, uint8_t stop_bits,
                       uint8_t parity, uint16_t timeout, uint8_t retries,
                       void (*callback)(void));
```

✅ **Better - use struct:**
```c
/**
 * @brief UART 配置参数结构体
 * @details 所有 UART 初始化参数集中定义
 */
typedef struct {
    uint32_t baud_rate;         // 波特率（9600-115200）
    uint8_t data_bits;          // 数据位（8 或 9）
    uint8_t stop_bits;          // 停止位（1 或 2）
    uint8_t parity;             // 奇偶校验
    uint16_t timeout_ms;        // 超时时间
    uint8_t retries;            // 重试次数
    void (*on_complete)(void);  // 完成回调
} UartConfig_t;

ErrorCode_t uart_init(const UartConfig_t *p_config);
```

#### 9.2 Function Complexity Limits

**Rules**:
- Maximum 100 lines per function (excluding comments)
- Maximum 3 levels of nesting
- Prefer early returns to reduce nesting
- Break complex logic into smaller functions

**Example (中文注释):**
```c
// ❌ 过于复杂 - 深层嵌套
ErrorCode_t process_data(uint8_t *p_data, uint16_t len)
{
    if (len > 0)
    {
        if (validate_crc(p_data))
        {
            if (parse_packet(p_data))
            {
                if (execute_command(p_data))
                {
                    // 嵌套太深，难以理解
                    return ERROR_SUCCESS;
                }
            }
        }
    }
    return ERROR_INVALID;
}

// ✅ 改进 - 提前返回，减少嵌套
ErrorCode_t process_data(uint8_t *p_data, uint16_t len)
{
    // 参数检查 - 提前返回
    if (len == 0)
        return ERROR_INVALID_PARAM;
    
    // 逐步验证 - 减少嵌套
    if (!validate_crc(p_data))
        return ERROR_CRC_FAILED;
    
    if (!parse_packet(p_data))
        return ERROR_PARSE_FAILED;
    
    if (!execute_command(p_data))
        return ERROR_EXECUTION_FAILED;
    
    return ERROR_SUCCESS;
}
```

#### 9.3 Return Values - Always Check

**Rules**:
- Every function must document what it returns
- Caller must check return values
- Use error codes consistently
- Document success and failure conditions

**Good pattern (中文注释):**
```c
/**
 * @brief 读取传感器温度
 * @param p_temp_celsius 输出：温度值指针（-40 到 +125°C）
 * @return ErrorCode_t - 成功返回 ERROR_SUCCESS
 * @note 阻塞最多 50ms
 * 
 * @details
 * 成功时：*p_temp_celsius 被填充
 * 失败时：*p_temp_celsius 值未定义
 */
ErrorCode_t sensor_read_temperature(int16_t *p_temp_celsius);

// 使用方式
int16_t temp;
ErrorCode_t result = sensor_read_temperature(&temp);
if (result != ERROR_SUCCESS)
{
    // 错误处理 - 根据错误类型
    if (result == ERROR_TIMEOUT)
        log_error("传感器读取超时");
    else if (result == ERROR_NOT_INITIALIZED)
        log_error("传感器未初始化");
    
    return result;  // 传播错误，不要忽略
}

// 使用温度值
log_info("Temperature: %d°C", temp);
```

---

### 10. Race Condition and Thread-Safety Guidelines

**Rationale**: Embedded systems often use interrupts and multiple tasks. Race conditions cause unpredictable bugs.

#### 10.1 Critical Section Protection

**Rules**:
- Identify all shared resources (variables, registers)
- Document which operations are critical
- Use `DISABLE_INTERRUPTS()` for ISR conflicts
- Use mutex/semaphore for multi-task conflicts

**Example (中文注释):**
```c
/**
 * @brief 通过 DMA 发送数据
 * @param p_data 数据指针
 * @param len 数据长度
 * @return ErrorCode_t - 成功返回 ERROR_SUCCESS
 * 
 * @details
 * 竞态条件说明：
 * - 修改 s_dma_ptr 和 s_dma_len 时可能被 DMA ISR 中断
 * - ISR 可能同时修改 s_dma_status
 * - 解决方案：禁用中断保护
 */
ErrorCode_t uart_send_dma(const uint8_t *p_data, uint16_t len)
{
    if (!s_inited || p_data == NULL || len == 0)
        return ERROR_INVALID_PARAM;
    
    // 为什么：DMA ISR 可能同时修改这些变量，需要原子操作
    DISABLE_INTERRUPTS();
    {
        s_dma_ptr = (uint8_t *)p_data;
        s_dma_len = len;
        s_dma_status = DMA_STATUS_BUSY;
        
        // 启动 DMA
        DMA->CR1 = DMA_ENABLE | DMA_FROM_MEMORY;
    }
    ENABLE_INTERRUPTS();
    
    return ERROR_SUCCESS;
}
```

#### 10.2 Volatile Variables in ISR

**Rules**:
- Global variables accessed from ISR must be `volatile`
- Document which ISR accesses them
- Mark them with `@isr` tag in comments

**Example (中文注释):**
```c
/**
 * @brief UART 接收缓冲写指针
 * @isr: uart_rx_isr() 写入，主任务读取
 * @thread-safe: ISR 独占写，主任务只读
 */
static volatile uint16_t s_uart_rx_write_idx = 0;

/**
 * @brief 上一个 ADC 采样值
 * @isr: adc_complete_isr() 更新
 * @thread-safe: ISR 原子写，主任务无保护读
 */
static volatile uint16_t s_adc_value = 0;

// ISR 中
void uart_rx_isr(void)
{
    uint8_t byte = UART->DR;
    s_uart_rx_buffer[s_uart_rx_write_idx] = byte;
    s_uart_rx_write_idx = (s_uart_rx_write_idx + 1) % UART_RX_BUFFER_SIZE;
}

// 主任务中
void main_task(void)
{
    uint16_t write_idx = s_uart_rx_write_idx;  // 读取 volatile 变量
    if (write_idx != s_uart_rx_read_idx)
    {
        // 有新数据
        uint8_t byte = s_uart_rx_buffer[s_uart_rx_read_idx];
    }
}
```

---

### 11. Buffer Overflow and Memory Safety

**Rationale**: Buffer overflows are the leading cause of embedded system crashes and security vulnerabilities.

#### 11.1 Buffer Size Validation

**Rules**:
- Always validate buffer sizes at function entry
- Use macros for buffer sizes, never magic numbers
- Document minimum/maximum sizes
- Check before every write operation

**Example (中文注释):**
```c
// ============================================================================
// 缓冲区大小定义
// ============================================================================

#define UART_RX_BUFFER_SIZE     (256)   // 接收缓冲最大容量
#define UART_TX_BUFFER_SIZE     (512)   // 发送缓冲最大容量
#define PACKET_MAX_LEN          (1024)  // 单个数据包最大长度

/**
 * @brief 向发送缓冲写入数据
 * @param p_data 数据指针
 * @param len 数据长度（1-PACKET_MAX_LEN）
 * @return ErrorCode_t - 成功或 ERROR_BUFFER_OVERFLOW
 * 
 * @details
 * 内部验证：
 * - len 必须 > 0 且 <= PACKET_MAX_LEN
 * - 缓冲区剩余空间 >= len
 */
ErrorCode_t uart_tx_write(const uint8_t *p_data, uint16_t len)
{
    // 参数有效性检查
    if (p_data == NULL || len == 0 || len > PACKET_MAX_LEN)
        return ERROR_INVALID_PARAM;
    
    // 为什么：防止缓冲区溢出，必须检查空间
    uint16_t available_space = UART_TX_BUFFER_SIZE - s_tx_write_idx;
    if (available_space < len)
        return ERROR_BUFFER_OVERFLOW;
    
    // 写入数据 - 现在安全
    memcpy(&s_uart_tx_buffer[s_tx_write_idx], p_data, len);
    s_tx_write_idx += len;
    
    return ERROR_SUCCESS;
}
```

#### 11.2 Pointer Validation

**Rules**:
- Never dereference NULL pointers
- Validate pointers before use
- Use `p_` prefix for all pointers
- Document pointer ownership

**Example (中文注释):**
```c
/**
 * @brief 获取缓冲区指针
 * @param p_p_buffer 输出：缓冲区指针的指针
 * @return ErrorCode_t - 成功返回 ERROR_SUCCESS
 * @note p_p_buffer 不能为 NULL
 * 
 * @details
 * 指针所有权：返回的缓冲区归函数所有，调用者不能 free()
 */
ErrorCode_t get_buffer(uint8_t **p_p_buffer)
{
    // 为什么：输出指针必须有效，否则会导致崩溃
    if (p_p_buffer == NULL)
        return ERROR_INVALID_PARAM;
    
    *p_p_buffer = s_internal_buffer;
    return ERROR_SUCCESS;
}

// 使用
uint8_t *p_buf = NULL;
ErrorCode_t result = get_buffer(&p_buf);
if (result == ERROR_SUCCESS && p_buf != NULL)
{
    // 现在 p_buf 安全可用
    p_buf[0] = 0xFF;
}
```

---

### 12. Resource Leak Prevention

**Rationale**: Resource leaks (memory, file handles, semaphores) cause system degradation over time.

#### 12.1 Allocation/Deallocation Pairing

**Rules**:
- Every `malloc()` must have a corresponding `free()`
- Every `acquire()` must have a corresponding `release()`
- Use RAII-like patterns where possible
- Document resource lifetime

**Example (中文注释):**
```c
/**
 * @brief 打开设备
 * @return ErrorCode_t - 成功返回 ERROR_SUCCESS
 * 
 * @details
 * 资源分配：获取互斥锁、打开 UART、启用中断
 * 必须与 device_close() 配对
 */
ErrorCode_t device_open(void)
{
    // 获取资源 1 - 互斥锁
    ErrorCode_t result = mutex_acquire(g_device_mutex, TIMEOUT_MS);
    if (result != ERROR_SUCCESS)
        return result;  // 失败 - 未分配任何资源
    
    // 获取资源 2 - UART
    result = uart_init(&g_uart_config);
    if (result != ERROR_SUCCESS)
    {
        // 为什么：必须释放已获取的资源再返回
        mutex_release(g_device_mutex);
        return result;
    }
    
    // 获取资源 3 - 中断
    result = interrupt_enable(UART_IRQ);
    if (result != ERROR_SUCCESS)
    {
        // 为什么：反向顺序释放，防止死锁
        uart_close();
        mutex_release(g_device_mutex);
        return result;
    }
    
    return ERROR_SUCCESS;
}

/**
 * @brief 关闭设备
 * @note 必须与 device_open() 配对
 */
void device_close(void)
{
    // 为什么：反向顺序释放资源（LIFO），防止依赖问题
    interrupt_disable(UART_IRQ);
    uart_close();
    mutex_release(g_device_mutex);
}
```

---

### 13. Module Organization and API Design

**Rationale**: Clear module boundaries prevent coupling and make testing easier.

#### 13.1 Module Structure

**Rules**:
- Each module has one .h (interface) and one .c (implementation)
- Module name prefixes all public functions: `MODULE_action()`
- Use `static` for private functions
- Keep .h file minimal (public API only)

**File structure (中文注释):**
```c
// uart_driver.h - 公共 API 只有这些
#ifndef UART_DRIVER_H
#define UART_DRIVER_H

/**
 * @file uart_driver.h
 * @brief UART 驱动 - 公共接口
 * 
 * 本模块提供 UART 收发功能
 * 使用方式：
 * 1. 调用 uart_init() 初始化
 * 2. 使用 uart_send_byte() / uart_read_byte() 收发
 * 3. 调用 uart_close() 关闭
 */

typedef enum {
    ERROR_SUCCESS,
    ERROR_TIMEOUT,
    ERROR_NOT_INITIALIZED,
} uart_error_t;

// 公共 API - 只有这些
uart_error_t uart_init(const uart_config_t *p_config);
uart_error_t uart_send_byte(uint8_t byte);
uart_error_t uart_read_byte(uint8_t *p_byte);
void uart_close(void);

#endif

// uart_driver.c - 实现细节
#include "uart_driver.h"

// 私有状态
static uint8_t s_inited = 0;
static uint8_t s_uart_buffer[256];

// 私有函数 - 对外不可见
static void uart_isr(void)
{
    // ISR 处理
}

// 公共函数 - 模块前缀 uart_
uart_error_t uart_init(const uart_config_t *p_config)
{
    // 实现
}
```

#### 13.2 API Design - Principle of Least Surprise

**Rules**:
- Function names clearly indicate what they do
- Similar operations have similar names
- Return types consistent across module
- Document side effects and preconditions

**Good API design (中文注释):**
```c
// ============================================================================
// 传感器模块 - 清晰的 API 设计
// ============================================================================

// 初始化 - 返回错误码
sensor_error_t sensor_init(void);

// 读取 - 输出通过指针，返回错误码
sensor_error_t sensor_read_temperature(int16_t *p_temp_celsius);
sensor_error_t sensor_read_humidity(uint16_t *p_humidity_percent);

// 关闭 - 无返回值（不能失败的操作）
void sensor_close(void);

// 获取状态 - 简单查询，无 I/O
sensor_state_t sensor_get_state(void);

// 这种 API 的优点：
// 1. 名字清晰 - sensor_read_xxx 明显是读操作
// 2. 一致性 - 所有读操作都返回错误码
// 3. 易用 - 不需要记住参数顺序，输出总是指针
// 4. 安全 - 错误码必须检查
```

### 14. 分层架构和解耦设计

**Rationale**: 嵌入式系统随着功能增加会变得复杂。分层架构通过将系统分为独立的层，每层只关注自己的责任，从而提高代码可维护性、可测试性和可扩展性。

#### 14.1 分层模式

**典型的嵌入式系统分层**:

```
┌─────────────────────────────────┐
│   应用层 (APP)                  │  用户应用逻辑
├─────────────────────────────────┤
│   业务层 (Business Logic)        │  状态机、业务规则、事件处理
├─────────────────────────────────┤
│   驱动层 (Driver/HAL)            │  硬件抽象、设备驱动
├─────────────────────────────────┤
│   底层 (BSW/Firmware)            │  硬件初始化、寄存器操作
└─────────────────────────────────┘
```

**每层的职责**:

| 层 | 职责 | 依赖 | 示例 |
|---|------|------|------|
| APP | 应用逻辑 | 业务层 | `app_freq_detector.c` |
| 业务 | 状态机、算法 | 驱动层 | `bsw_freq_analyzer.c` |
| 驱动 | 硬件操作 | 底层 | `uart_driver.c`, `adc_driver.c` |
| 底层 | 寄存器、中断 | 无 | 硬件初始化代码 |

**分层的好处**:
- ✅ 清晰的依赖关系（单向依赖，从上到下）
- ✅ 每层独立开发和测试
- ✅ 硬件更换时只改底层，业务逻辑不变
- ✅ 易于多人协作开发

#### 14.2 函数表模式 - 解决 Switch/If-Else 爆炸

**问题 - 硬编码的 switch 语句**:

```c
// ❌ 问题：switch 语句过大，难以维护
void handle_command(uint8_t cmd_type, uint8_t *p_cmd)
{
    switch (cmd_type)
    {
        case CMD_START:
            start_operation();
            break;
        case CMD_STOP:
            stop_operation();
            break;
        case CMD_RESET:
            reset_operation();
            break;
        // ... 还有 20 多个 case
        default:
            error_handler();
    }
}
```

**问题**:
1. 每添加新命令都要修改这个函数
2. 难以单独测试每个命令处理
3. 命令处理逻辑散开在整个 switch 中
4. 容易遗漏新增命令的初始化

**解决 - 函数表模式 (中文注释)**:

```c
// ============================================================================
// 命令处理函数表 - 解耦和模块化
// ============================================================================

/**
 * @brief 命令处理函数原型
 * @param p_cmd 命令数据指针
 * @param len 命令长度
 * @return ErrorCode_t - 处理结果
 */
typedef ErrorCode_t (*cmd_handler_t)(const uint8_t *p_cmd, uint16_t len);

/**
 * @brief 命令处理表项
 * @details 每个命令类型对应一个处理函数
 */
typedef struct {
    uint8_t cmd_type;           // 命令类型
    cmd_handler_t handler;      // 处理函数
    const char *p_name;         // 命令名称（调试用）
} CmdHandlerEntry_t;

// ============================================================================
// 各个命令的处理函数 - 独立且清晰
// ============================================================================

/**
 * @brief 处理启动命令
 * @param p_cmd 命令数据
 * @param len 数据长度
 * @return ErrorCode_t - 成功返回 ERROR_SUCCESS
 */
static ErrorCode_t cmd_handler_start(const uint8_t *p_cmd, uint16_t len)
{
    // 为什么：命令特定的处理逻辑独立
    // 便于修改、调试、单元测试
    
    if (len < MIN_START_CMD_LEN)
        return ERROR_INVALID_LENGTH;
    
    uint16_t delay_ms = (p_cmd[0] << 8) | p_cmd[1];
    
    return start_operation_delayed(delay_ms);
}

/**
 * @brief 处理停止命令
 */
static ErrorCode_t cmd_handler_stop(const uint8_t *p_cmd, uint16_t len)
{
    return stop_operation();
}

/**
 * @brief 处理复位命令
 */
static ErrorCode_t cmd_handler_reset(const uint8_t *p_cmd, uint16_t len)
{
    return reset_operation();
}

/**
 * @brief 处理查询状态命令
 */
static ErrorCode_t cmd_handler_query_status(const uint8_t *p_cmd, uint16_t len)
{
    return query_system_status();
}

// ============================================================================
// 命令处理表 - 集中管理所有命令
// ============================================================================

/**
 * @brief 命令处理表
 * @note 新增命令时只需在此表中添加一项，无需修改分发函数
 */
static const CmdHandlerEntry_t s_cmd_handler_table[] =
{
    // cmd_type,                  handler,                  name
    { CMD_START,                  cmd_handler_start,        "START" },
    { CMD_STOP,                   cmd_handler_stop,         "STOP" },
    { CMD_RESET,                  cmd_handler_reset,        "RESET" },
    { CMD_QUERY_STATUS,           cmd_handler_query_status, "QUERY_STATUS" },
    // 新增命令时这样添加：
    // { CMD_NEW_COMMAND,          cmd_handler_new_command,  "NEW_COMMAND" },
};

#define CMD_HANDLER_TABLE_SIZE  (sizeof(s_cmd_handler_table) / sizeof(s_cmd_handler_table[0]))

// ============================================================================
// 统一的命令分发函数 - 清晰简洁
// ============================================================================

/**
 * @brief 根据命令类型分发到对应的处理函数
 * @param cmd_type 命令类型
 * @param p_cmd 命令数据
 * @param len 命令长度
 * @return ErrorCode_t - 处理结果
 * 
 * @details
 * 优点：
 * 1. 分发函数清晰，无复杂逻辑
 * 2. 添加新命令时无需修改此函数
 * 3. 每个命令处理独立，易于测试
 * 4. 表项集中管理，便于查看所有命令
 */
ErrorCode_t dispatch_command(uint8_t cmd_type, const uint8_t *p_cmd, uint16_t len)
{
    // 为什么：在表中查找对应的处理函数
    for (uint16_t i = 0; i < CMD_HANDLER_TABLE_SIZE; i++)
    {
        if (s_cmd_handler_table[i].cmd_type == cmd_type)
        {
            // 调用对应的处理函数
            return s_cmd_handler_table[i].handler(p_cmd, len);
        }
    }
    
    // 未找到对应命令
    return ERROR_UNKNOWN_COMMAND;
}
```

#### 14.3 虚接口模式 - 实现多态（不同设备同样操作）

**场景**：不同设备（UART、SPI、I2C）都需要初始化、打开、关闭、收发

**解决 - 设备虚接口 (中文注释)**:

```c
// ============================================================================
// 通用设备接口 - 虚函数表
// ============================================================================

/**
 * @brief 设备操作回调函数类型
 */
typedef ErrorCode_t (*device_init_t)(void);
typedef ErrorCode_t (*device_open_t)(void);
typedef void (*device_close_t)(void);
typedef ErrorCode_t (*device_send_t)(const uint8_t *p_data, uint16_t len);
typedef ErrorCode_t (*device_recv_t)(uint8_t *p_data, uint16_t *p_len);

/**
 * @brief 设备接口结构体 - 类似 C++ 虚函数表
 * @details 不同设备实现不同的函数指针
 */
typedef struct {
    const char *p_device_name;          // 设备名称
    device_init_t init;                 // 初始化
    device_open_t open;                 // 打开
    device_close_t close;               // 关闭
    device_send_t send;                 // 发送
    device_recv_t recv;                 // 接收
} DeviceInterface_t;

// ============================================================================
// UART 设备实现
// ============================================================================

static ErrorCode_t uart_impl_init(void)
{
    return uart_init(&g_uart_config);
}

static ErrorCode_t uart_impl_open(void)
{
    return uart_start();
}

static void uart_impl_close(void)
{
    uart_stop();
}

static ErrorCode_t uart_impl_send(const uint8_t *p_data, uint16_t len)
{
    return uart_send(p_data, len);
}

static ErrorCode_t uart_impl_recv(uint8_t *p_data, uint16_t *p_len)
{
    return uart_receive(p_data, p_len);
}

// ============================================================================
// SPI 设备实现
// ============================================================================

static ErrorCode_t spi_impl_init(void)
{
    return spi_init(&g_spi_config);
}

static ErrorCode_t spi_impl_open(void)
{
    return spi_enable();
}

static void spi_impl_close(void)
{
    spi_disable();
}

static ErrorCode_t spi_impl_send(const uint8_t *p_data, uint16_t len)
{
    return spi_transmit(p_data, len);
}

static ErrorCode_t spi_impl_recv(uint8_t *p_data, uint16_t *p_len)
{
    return spi_receive(p_data, p_len);
}

// ============================================================================
// 注册设备接口
// ============================================================================

/**
 * @brief UART 设备接口
 * @note 为什么用表格：添加新设备只需添加一个接口结构体，无需修改应用代码
 */
static const DeviceInterface_t g_uart_interface =
{
    .p_device_name = "UART",
    .init = uart_impl_init,
    .open = uart_impl_open,
    .close = uart_impl_close,
    .send = uart_impl_send,
    .recv = uart_impl_recv,
};

/**
 * @brief SPI 设备接口
 */
static const DeviceInterface_t g_spi_interface =
{
    .p_device_name = "SPI",
    .init = spi_impl_init,
    .open = spi_impl_open,
    .close = spi_impl_close,
    .send = spi_impl_send,
    .recv = spi_impl_recv,
};

// ============================================================================
// 通用设备管理层
// ============================================================================

/**
 * @brief 统一设备接口 - 应用层只与此接口交互
 */
typedef struct {
    const DeviceInterface_t *p_interface;    // 指向具体设备接口
} Device_t;

/**
 * @brief 初始化设备
 * @param p_device 设备对象
 * @param p_interface 指向设备接口（UART、SPI 等）
 * @return ErrorCode_t
 * 
 * @details
 * 为什么这样设计：
 * 1. 应用代码不知道用的是什么设备
 * 2. 换设备时只改初始化，业务逻辑不变
 * 3. 支持同时使用多个不同设备
 * 4. 易于单元测试（可注入 Mock 设备）
 */
ErrorCode_t device_init(Device_t *p_device, const DeviceInterface_t *p_interface)
{
    if (p_device == NULL || p_interface == NULL || p_interface->init == NULL)
        return ERROR_INVALID_PARAM;
    
    p_device->p_interface = p_interface;
    return p_interface->init();
}

ErrorCode_t device_open(Device_t *p_device)
{
    if (p_device == NULL || p_device->p_interface->open == NULL)
        return ERROR_INVALID_PARAM;
    
    return p_device->p_interface->open();
}

void device_close(Device_t *p_device)
{
    if (p_device != NULL && p_device->p_interface->close != NULL)
        p_device->p_interface->close();
}

ErrorCode_t device_send(Device_t *p_device, const uint8_t *p_data, uint16_t len)
{
    if (p_device == NULL || p_device->p_interface->send == NULL)
        return ERROR_INVALID_PARAM;
    
    return p_device->p_interface->send(p_data, len);
}

ErrorCode_t device_recv(Device_t *p_device, uint8_t *p_data, uint16_t *p_len)
{
    if (p_device == NULL || p_device->p_interface->recv == NULL)
        return ERROR_INVALID_PARAM;
    
    return p_device->p_interface->recv(p_data, p_len);
}

// ============================================================================
// 使用示例 - 应用层代码无需关心设备类型
// ============================================================================

int main(void)
{
    Device_t communication_device;
    
    // 为什么：这是唯一需要指定设备类型的地方
    // 如果要换成 SPI，只改这一行即可
    ErrorCode_t result = device_init(&communication_device, &g_uart_interface);
    
    if (result != ERROR_SUCCESS)
        return 1;
    
    result = device_open(&communication_device);
    if (result != ERROR_SUCCESS)
        return 1;
    
    // 应用代码 - 完全不知道用的是什么设备
    uint8_t tx_data[64] = "Hello";
    device_send(&communication_device, tx_data, 5);
    
    uint8_t rx_data[64];
    uint16_t rx_len = sizeof(rx_data);
    device_recv(&communication_device, rx_data, &rx_len);
    
    device_close(&communication_device);
    return 0;
}

// 如果要改用 SPI，只需改这一行：
// ErrorCode_t result = device_init(&communication_device, &g_spi_interface);
```

#### 14.4 状态机模式 - 管理复杂的状态转换

**场景**：设备有多个状态（IDLE → RUNNING → ERROR → IDLE），每个状态有不同的处理

```c
// ============================================================================
// 状态机实现 - 函数表模式
// ============================================================================

/**
 * @brief 系统状态定义
 */
typedef enum {
    STATE_IDLE,         // 空闲
    STATE_INITIALIZING, // 初始化中
    STATE_RUNNING,      // 运行中
    STATE_ERROR,        // 错误状态
    STATE_STOPPED,      // 已停止
} SystemState_t;

/**
 * @brief 状态事件定义
 */
typedef enum {
    EVENT_START,
    EVENT_STOP,
    EVENT_ERROR,
    EVENT_RECOVER,
    EVENT_TIMEOUT,
} SystemEvent_t;

// ============================================================================
// 每个状态的处理函数
// ============================================================================

/**
 * @brief IDLE 状态的事件处理
 * @return 新状态
 */
static SystemState_t handle_idle_state(SystemEvent_t event)
{
    switch (event)
    {
        case EVENT_START:
            // 为什么：IDLE 态收到 START，转换到 INITIALIZING
            return STATE_INITIALIZING;
        case EVENT_ERROR:
            return STATE_ERROR;
        default:
            return STATE_IDLE;  // 保持当前状态
    }
}

/**
 * @brief INITIALIZING 状态的事件处理
 */
static SystemState_t handle_initializing_state(SystemEvent_t event)
{
    switch (event)
    {
        case EVENT_START:
            // 为什么：初始化完成，转换到 RUNNING
            return STATE_RUNNING;
        case EVENT_STOP:
            return STATE_IDLE;
        case EVENT_ERROR:
            return STATE_ERROR;
        default:
            return STATE_INITIALIZING;
    }
}

/**
 * @brief RUNNING 状态的事件处理
 */
static SystemState_t handle_running_state(SystemEvent_t event)
{
    switch (event)
    {
        case EVENT_STOP:
            return STATE_STOPPED;
        case EVENT_ERROR:
            return STATE_ERROR;
        case EVENT_TIMEOUT:
            return STATE_ERROR;
        default:
            return STATE_RUNNING;
    }
}

/**
 * @brief ERROR 状态的事件处理
 */
static SystemState_t handle_error_state(SystemEvent_t event)
{
    switch (event)
    {
        case EVENT_RECOVER:
            return STATE_IDLE;
        case EVENT_STOP:
            return STATE_STOPPED;
        default:
            return STATE_ERROR;
    }
}

/**
 * @brief STOPPED 状态的事件处理
 */
static SystemState_t handle_stopped_state(SystemEvent_t event)
{
    switch (event)
    {
        case EVENT_START:
            return STATE_INITIALIZING;
        default:
            return STATE_STOPPED;
    }
}

// ============================================================================
// 状态处理表 - 集中管理状态转换
// ============================================================================

/**
 * @brief 状态处理函数原型
 */
typedef SystemState_t (*state_handler_t)(SystemEvent_t event);

/**
 * @brief 状态处理表
 * @note 为什么：所有状态转换规则集中管理，易于查看和修改
 */
static const state_handler_t s_state_handler_table[] =
{
    handle_idle_state,          // STATE_IDLE
    handle_initializing_state,  // STATE_INITIALIZING
    handle_running_state,       // STATE_RUNNING
    handle_error_state,         // STATE_ERROR
    handle_stopped_state,       // STATE_STOPPED
};

#define STATE_HANDLER_TABLE_SIZE  (sizeof(s_state_handler_table) / sizeof(s_state_handler_table[0]))

// ============================================================================
// 状态机核心
// ============================================================================

static SystemState_t s_current_state = STATE_IDLE;

/**
 * @brief 处理事件并转换状态
 * @param event 发生的事件
 * 
 * @details
 * 为什么这样设计：
 * 1. 所有状态转换规则清晰可见
 * 2. 添加新状态只需添加一个处理函数和表项
 * 3. 易于扩展和维护
 * 4. 便于绘制状态转换图
 */
void system_on_event(SystemEvent_t event)
{
    if (s_current_state >= STATE_HANDLER_TABLE_SIZE)
    {
        s_current_state = STATE_IDLE;  // 错误恢复
        return;
    }
    
    // 为什么：通过表查询当前状态的处理函数
    state_handler_t handler = s_state_handler_table[s_current_state];
    
    if (handler != NULL)
    {
        SystemState_t new_state = handler(event);
        
        if (new_state != s_current_state)
        {
            // 状态转换发生
            on_state_changed(s_current_state, new_state, event);
            s_current_state = new_state;
        }
    }
}

SystemState_t system_get_state(void)
{
    return s_current_state;
}
```

---

## Code Review Checklist - Extended

When reviewing embedded C code, verify:

**基础规范 (8 项)**
- [ ] 没有魔法数字：所有常量都用宏定义
- [ ] 大括号格式：所有 `{` `}` 都在新行
- [ ] 变量命名：g_/s_/p_ 前缀一致
- [ ] 注释规范：中文注释，函数有文档
- [ ] 全局变量：都有 @rationale 说明
- [ ] 错误处理：ErrorCode_t 返回，调用者检查
- [ ] 结构体设计：字段分组、大小排序
- [ ] AI 工作流：代码有注释，没有额外 .md

**函数设计 (5 项)**
- [ ] 参数个数：≤ 4 个，否则用结构体
- [ ] 参数验证：入口检查 NULL、范围、大小
- [ ] 函数长度：≤ 100 行代码
- [ ] 嵌套深度：≤ 3 层或提前返回
- [ ] 返回值检查：所有错误返回都被检查

**并发安全 (4 项)**
- [ ] Volatile 标记：ISR 访问的变量都标记
- [ ] 临界段保护：多语句修改用 DISABLE_INTERRUPTS()
- [ ] 文档标记：@isr 说明哪个 ISR 访问
- [ ] 原子操作：组合操作都有保护

**内存安全 (4 项)**
- [ ] 缓冲验证：len 检查（>0 和 ≤ MAX）
- [ ] 指针检查：所有指针 != NULL 后才用
- [ ] 边界检查：所有写操作都检查可用空间
- [ ] 资源配对：init/close、acquire/release 配对

**架构设计【新增】(6 项)**
- [ ] 分层清晰：依赖单向，从上到下
- [ ] 解耦充分：没有大的 switch/if-else，改用函数表
- [ ] 函数表设计：表项集中、处理函数独立
- [ ] 虚接口：相同操作的不同实现用表格管理
- [ ] 状态机：状态转换规则集中、处理函数独立
- [ ] 无循环依赖：模块间依赖是单向的

**模块接口 (3 项)**
- [ ] API 命名：都带模块前缀（uart_*, device_*）
- [ ] API 一致：类似操作有类似签名
- [ ] 私/公分界：私有函数标记 static

---

## Code Generation Guidelines

When writing embedded C code for the user:

1. **Replace all magic numbers with macros** before writing functions, with module prefixes
2. **Name all variables consistently**: Global (`g_`), static (`s_`), pointer (`p_`), functions lowercase_with_underscores
3. **Add comprehensive comments**: File headers, function headers, and explain *why* for hardware operations
4. **Format all braces on new lines** consistently throughout the code
5. **Document globals/statics**: Every global/static variable needs a comment explaining why it's not local
6. **Use consistent error handling**: Functions return ErrorCode_t, callers check return values
7. **Organize structures**: Group related fields, order by size, document each field
8. **Avoid unnecessary globals**: Prefer static variables for file scope, or pass pointers to functions

---

## Feedback Examples

**If code violates conventions:**

"Found magic number `0x1000` on line 42. This should be:
```c
#define MEMORY_BUFFER_SIZE (0x1000)
```
Then use `MEMORY_BUFFER_SIZE` in the code."

"Bracket formatting inconsistent on line 15. Change:
```c
if (ready) {
```
To:
```c
if (ready)
{
```
"

---

## When to Apply This Skill

This skill applies in these scenarios:

1. **Code Review**: Before merging pull requests, check against the full checklist
2. **New Feature Development**: Write code following these conventions from the start
3. **Refactoring**: Update code to comply with naming and style standards
4. **Onboarding**: Review your codebase against these standards to ensure consistency
5. **Debugging**: Poor naming often indicates logic issues; rename as part of bug fixes

---

## Future Enhancements

This skill will be expanded with:
- Memory management patterns (stack allocation, buffer management)
- Interrupt-safe coding patterns (critical sections, atomic operations)
- Function complexity guidelines (max parameters, max lines)
- Test and assert macros for embedded testing
- Performance optimization patterns

