---
title: 分层架构与分发结构选择指南
---

# 分层架构与分发结构选择指南

本指南补充 `SKILL.md` 第 14 节。重点是按职责拆分模块，并根据实际需求选择 `switch`、查表或函数表，不把某一种分发结构作为普遍答案。

## 1. 分层架构

分层用于明确职责和依赖方向。层级名称应按项目实际架构定义；典型 MCU 项目可采用：

```text
应用层（APP）：产品流程、业务策略
业务/服务层（BSW）：协议、算法、状态控制
驱动/硬件抽象层：外设驱动和硬件访问接口
平台层：启动、芯片支持包和寄存器定义
```

上层通过下层接口完成工作。避免下层反向依赖应用逻辑；跨层共享数据应通过明确接口传递。不是每个项目都需要相同数量的层，避免为形式而增加空壳模块。

## 2. 选择分发结构

### 使用 `switch` 的典型情况

- 状态或事件集合有限、稳定。
- 需要显式看到所有分支，或希望编译器帮助发现遗漏。
- 每个分支短小、行为局部且容易理解。

```c
switch (event)
{
    case EVENT_START:
        start_operation();
        break;

    case EVENT_STOP:
        stop_operation();
        break;

    default:
        handle_unknown_event();
        break;
}
```

### 使用函数表的典型情况

- 处理项需要独立实现、测试或按配置替换。
- 项目有运行时注册/扩展需求。
- 表驱动能明显减少重复的映射代码。

```c
typedef ErrorCode_t (*command_handler_t)(const uint8_t *p_data,
                                         size_t data_len);

typedef struct
{
    uint8_t command_id;
    command_handler_t handler;
} CommandEntry_t;

static const CommandEntry_t s_command_table[] =
{
    { CMD_START, command_start },
    { CMD_STOP, command_stop }
};

static ErrorCode_t dispatch_command(uint8_t command_id,
                                    const uint8_t *p_data,
                                    size_t data_len)
{
    size_t index;

    for (index = 0U;
         index < (sizeof(s_command_table) / sizeof(s_command_table[0]));
         index++)
    {
        if (s_command_table[index].command_id == command_id)
        {
            if (s_command_table[index].handler == NULL)
            {
                return ERROR_INVALID_STATE;
            }

            return s_command_table[index].handler(p_data, data_len);
        }
    }

    return ERROR_UNKNOWN_COMMAND;
}
```

表驱动实现需要验证键值/索引、函数指针和未知项处理；直接数组索引时需先检查索引小于表长。对于少量、稳定且需要穷举的命令，`switch` 往往更简单清晰。不要仅以减少 `switch` 行数为目标引入间接调用。

## 3. 设备接口抽象

当多个设备确实需要通过同一组操作替换或注入测试实现时，可定义函数指针接口。接口对象应保持只读配置，并在调用前确认必需函数指针有效。若设备在构建时已固定且没有测试替换需求，直接调用具体驱动接口通常更简单。

```c
typedef struct
{
    ErrorCode_t (*init)(void);
    ErrorCode_t (*send)(const uint8_t *p_data, size_t data_len);
} CommInterface_t;

static ErrorCode_t comm_send(const CommInterface_t *p_interface,
                             const uint8_t *p_data,
                             size_t data_len)
{
    if ((p_interface == NULL) || (p_interface->send == NULL) ||
        (p_data == NULL) || (data_len == 0U))
    {
        return ERROR_INVALID_PARAM;
    }

    return p_interface->send(p_data, data_len);
}
```

## 4. 状态机

有限状态机可用 `switch` 表达状态与事件转换，也可按状态使用处理函数表。选择便于审查转换完整性、边界行为和副作用的实现方式。无效状态、未知事件和不允许的转换必须有明确策略；使用函数表时，在索引前校验状态范围并检查处理函数非空。

```c
static SystemState_t transition(SystemState_t state, SystemEvent_t event)
{
    switch (state)
    {
        case STATE_IDLE:
            if (event == EVENT_START)
            {
                return STATE_RUNNING;
            }
            return STATE_IDLE;

        case STATE_RUNNING:
            if (event == EVENT_STOP)
            {
                return STATE_IDLE;
            }
            return STATE_RUNNING;

        default:
            return STATE_ERROR;
    }
}
```

## 5. 评审要点

- [ ] 模块边界对应真实职责，依赖方向清晰。
- [ ] 没有为了“分层”或“去 switch”而增加无收益的抽象。
- [ ] `switch` 的未知分支和状态穷举策略明确。
- [ ] 函数表的索引、键值、函数指针和未知项经过验证。
- [ ] 接口抽象对应实际替换、扩展或隔离测试需求。
- [ ] 状态转换、副作用和错误处理容易单独验证。
