# 云手机自动化工作流执行器

> 基于 JSON Schema 驱动的云手机自动化工作流执行引擎

[![TypeScript](https://img.shields.io/badge/TypeScript-5.5.4-blue.svg)](https://www.typescriptlang.org/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](./LICENSE)

## 📋 目录

- [项目简介](#项目简介)
  - [系统架构](#系统架构)
  - [执行流程](#执行流程)
  - [数据流转机制](#数据流转机制)
  - [节点分类](#节点分类)
- [构建产物](#构建产物)
- [快速开始](#快速开始)
- [执行器使用](#执行器使用)
- [工作流数据格式](#工作流数据格式)
  - [整体结构关系](#整体结构关系)
  - [工作流结构](#工作流结构workflowschema)
  - [节点结构](#节点结构flownodejson)
  - [数据值类型](#数据值类型flowvalue)
- [内置节点](#内置节点)
  - [节点执行逻辑](#节点执行逻辑)
- [动作节点](#动作节点)
- [动作函数开发](#动作函数开发)
  - [开发流程](#开发流程)
  - [动作函数执行机制](#动作函数执行机制)
  - [开发规范](#开发规范)
  - [开发步骤](#开发步骤)
  - [最佳实践](#最佳实践)
- [类型定义](#类型定义)

## 项目简介

本项目是一个基于 JSON Schema 驱动的云手机自动化工作流执行引擎，专注于：

- **工作流数据格式定义**: 标准化的 JSON 工作流格式，支持节点嵌套和数据流转
- **工作流执行引擎**: 解析和执行工作流定义，支持条件分支、循环等控制流
- **插件化动作系统**: 灵活的动作节点注册机制，易于扩展
- **类型安全**: 完整的 TypeScript 类型定义，确保工作流数据的正确性

### 系统架构

```mermaid
graph TB
    subgraph Application["应用层 (Application)"]
        Main["main.ts<br/>(生产环境)"]
        Demo["demo.ts<br/>(演示)"]
        Other["其他入口"]
    end
    
    subgraph Engine["执行引擎层 (Engine)"]
        Executor["Executor (executor.ts)<br/>• 工作流加载与验证<br/>• 节点顺序执行<br/>• 数据流转管理<br/>• 执行报告生成"]
    end
    
    subgraph NodeLayer["节点处理层 (Node Layer)"]
        BuiltIn["内置节点<br/>• start<br/>• end"]
        Control["控制流节点<br/>• condition<br/>• loop"]
        Action["动作节点<br/>• click<br/>• openApp<br/>• waitElem<br/>• ...更多"]
    end
    
    subgraph Registry["动作注册层 (Action Registry)"]
        ActionReg["ACTION_REGISTRY<br/>(actions/index.ts)<br/>• 插件化注册机制<br/>• 动态函数映射<br/>• 扩展点"]
    end
    
    subgraph Platform["平台适配层 (Platform Adapter)"]
        Runtime["云手机运行时环境<br/>• runtime.args<br/>• runtime.report<br/>• runtime.end<br/>• globalThis.fetchSync<br/>• 元素查找与操作 API"]
    end
    
    Application --> Engine
    Engine --> NodeLayer
    NodeLayer --> Registry
    Registry --> Platform
    
    style Application fill:#e1f5ff
    style Engine fill:#fff3e0
    style NodeLayer fill:#f3e5f5
    style Registry fill:#e8f5e9
    style Platform fill:#fce4ec
```

### 执行流程

```mermaid
flowchart TD
    Start([开始执行]) --> Init[1. 初始化执行器<br/>• 创建 Executor 实例<br/>• 规范化 initialParams<br/>• 初始化 ExecutionContext]
    Init --> Load[2. 加载工作流<br/>• 从 URL 加载 或<br/>• 直接使用传入的 JSON]
    Load --> Validate[3. 验证工作流<br/>• 检查 start 节点唯一性<br/>• 检查 end 节点唯一性]
    Validate --> Execute[4. 顺序执行节点<br/>遍历 nodes 数组]
    
    Execute --> Process[处理单个节点]
    Process --> Parse[解析 inputsValues]
    Parse --> Run[执行节点逻辑]
    Run --> Save[保存 outputs]
    Save --> Record[记录执行报告]
    
    Record --> TypeCheck{节点类型判断}
    TypeCheck -->|start| StartNode[设置初始参数]
    TypeCheck -->|control| ControlNode[条件/循环分支]
    TypeCheck -->|action| ActionNode[调用动作函数]
    TypeCheck -->|end| EndNode[收集输出结果]
    
    StartNode --> Report
    ControlNode --> Report
    ActionNode --> Report
    EndNode --> Report
    
    Report[5. 生成执行报告<br/>• status: success/failed<br/>• nodeRecords[]<br/>• duration] --> Upload[6. 上报执行结果<br/>• runtime.end()<br/>• 输出日志]
    Upload --> End([执行完成])
    
    style Start fill:#4caf50,color:#fff
    style End fill:#4caf50,color:#fff
    style TypeCheck fill:#ff9800,color:#fff
    style Report fill:#2196f3,color:#fff
```

### 数据流转机制

```mermaid
flowchart TD
    InitParams[initialParams<br/>初始参数] --> StartNode
    
    StartNode["start 节点<br/>outputs: { 视频文件, 视频描述 }"]
    StartNode -.->|ref 引用| OpenApp
    
    OpenApp["openApp 节点<br/>inputs: { url: ref[start, 视频文件] }<br/>outputs: { packageName }"]
    OpenApp -.->|ref 引用| WaitElem
    
    WaitElem["waitElem 节点<br/>inputs: { selector: constant[...] }<br/>outputs: { element }"]
    WaitElem -.->|ref 引用| Click
    
    Click["click 节点<br/>inputs: { selectorRef: ref[waitElem, element] }<br/>outputs: { element }"]
    Click -.->|ref 引用| EndNode
    
    EndNode["end 节点<br/>inputs: { success: constant[true] }"]
    EndNode --> Report[ExecutionReport]
    
    subgraph Storage["数据存储: ExecutionContext.outputs (Map)"]
        direction TB
        Store["nodeId → outputs<br/>━━━━━━━━━━━━━━━━<br/>start_0 → { 视频文件: ... }<br/>openApp_0 → { packageName: ... }<br/>waitElem_0 → { element: {...} }<br/>click_0 → { element: {...} }<br/>loop_0_locals → { item: 0, ... }"]
    end
    
    style InitParams fill:#4caf50,color:#fff
    style Report fill:#2196f3,color:#fff
    style Storage fill:#fff3e0
```

### 节点分类

```mermaid
graph TD
    Root[FlowNodeJSON]
    
    Root --> BuiltIn[内置节点]
    Root --> Control[控制流节点]
    Root --> Action[动作节点]
    
    BuiltIn --> Start[start<br/>必须且唯一]
    BuiltIn --> End[end<br/>必须且唯一]
    
    Control --> Condition[condition<br/>条件判断]
    Control --> Loop[loop<br/>循环]
    
    Action --> UI[UI操作]
    Action --> App[应用操作]
    
    UI --> Click[click<br/>点击]
    UI --> WaitElem[waitElem<br/>等待元素]
    UI --> Swipe[swipe<br/>滑动]
    UI --> Keyboard[keyboard<br/>键盘]
    UI --> BackPage[backPage<br/>后退]
    UI --> WaitTime[waitTime<br/>等待]
    
    App --> OpenApp[openApp<br/>打开应用]
    App --> CloseApp[closeApp<br/>关闭应用]
    
    style Root fill:#e3f2fd
    style BuiltIn fill:#c8e6c9
    style Control fill:#fff9c4
    style Action fill:#f8bbd0
    style Start fill:#4caf50,color:#fff
    style End fill:#f44336,color:#fff
    style Condition fill:#ff9800,color:#fff
    style Loop fill:#ff9800,color:#fff
```

**节点特性:**
- **start/end**: 必须且唯一，定义工作流边界
- **condition**: 支持多分支条件判断 (eq/ne/gt/gte/lt/lte)
- **loop**: 支持数组循环(loopFor)和次数循环(count)
- **动作节点**: 可扩展，通过 ACTION_REGISTRY 注册

## 📦 构建产物

项目构建后会生成两个 JavaScript 文件：

### 1. `dist/main.js` - 生产环境脚本

**用途**: 实际运行的执行器脚本，从 `runtime.args` 动态获取任务参数

**特点**:
- 从云手机运行时环境的 `runtime.args` 获取任务参数
- 支持完整工作流执行
- 包含完整的错误处理和执行报告
- 适用于云手机自动化平台的生产环境

**使用场景**: 在云手机平台上作为 RPA 任务执行器

**示例 runtime.args**:
```json
{
  "locale": "zh-CN",
  "workflowUrl": "https://example.com/workflow.json",
  "initialParams": {
    "视频文件": "https://example.com/video.mp4",
    "视频描述": "这是一个测试视频"
  }
}
```

### 2. `dist/demo.js` - 演示脚本

**用途**: 本地测试和演示用的脚本，包含硬编码的工作流参数

**特点**:
- 直接在代码中定义工作流 URL 和参数
- 包含硬编码的初始化参数
- 可以直接运行，无需外部参数
- 适用于本地开发、测试和演示

**使用场景**: 开发阶段的本地测试和功能演示

**源文件**: `src/demo.ts`

## 快速开始

### 基本使用示例

```typescript
import { Executor } from './executor'
import type { ExecutorParams } from './types/executor'

// 配置执行器参数
const params: ExecutorParams = {
  locale: 'zh-CN',
  workflowUrl: 'https://example.com/workflow.json',
  initialParams: {
    // 方式1: 直接传入简单值（会自动转换为 FlowValue）
    '视频文件': 'https://example.com/video.mp4',
    '视频描述': '这是一个测试视频',
    
    // 方式2: 使用 FlowValue 格式
    '发布按钮文字': {
      type: 'constant',
      content: '发布'
    }
  }
}

// 创建执行器并执行工作流
try {
  const executor = new Executor(params)
  executor.execute()
} catch (error) {
  console.error('执行工作流时出错:', error)
}
```

### 执行流程说明

执行器的执行流程如下：

1. **初始化**: 创建执行器实例，规范化初始参数
2. **加载工作流**: 从 URL 或直接使用传入的工作流 JSON
3. **验证工作流**: 检查开始节点和结束节点是否唯一
4. **按序执行节点**: 遍历节点数组，依次执行每个节点
5. **生成报告**: 收集执行记录，生成执行报告

### 完整示例工作流

以下是一个 TikTok 自动发布视频的完整工作流示例：

```mermaid
flowchart TD
    Start["① start 节点<br/>━━━━━━━━━━<br/>提供初始参数:<br/>• 视频文件: https://example.com/video.mp4<br/>• 视频描述: 这是一个测试视频"]
    
    OpenApp["② openApp 节点<br/>━━━━━━━━━━<br/>打开 TikTok 应用<br/>packageName: com.zhiliaoapp.musically"]
    
    Wait1["③ waitTime 节点<br/>━━━━━━━━━━<br/>等待应用启动完成<br/>waitTime: 3000ms"]
    
    WaitElem["④ waitElement 节点<br/>━━━━━━━━━━<br/>等待'发布'按钮出现<br/>selector: [{ mode: textContains, text: 发布 }]<br/>输出: element (元素对象)"]
    
    Click["⑤ click 节点<br/>━━━━━━━━━━<br/>点击发布按钮<br/>selectorRef: ref[waitElement_0, element]<br/>actionEvent: click"]
    
    Wait2["⑥ waitTime 节点<br/>━━━━━━━━━━<br/>等待上传页面加载<br/>waitTime: 2000ms"]
    
    End["⑦ end 节点<br/>━━━━━━━━━━<br/>结束工作流<br/>success: true"]
    
    Start -->|输出参数| OpenApp
    OpenApp --> Wait1
    Wait1 --> WaitElem
    WaitElem -.->|输出元素| Click
    Click -.->|引用元素| Wait2
    Wait2 --> End
    
    style Start fill:#4caf50,color:#fff
    style End fill:#f44336,color:#fff
    style WaitElem fill:#2196f3,color:#fff
    style Click fill:#ff9800,color:#fff
```

## 工作流数据格式

### 整体结构关系

```mermaid
graph TD
    Workflow[WorkflowSchema<br/>工作流]
    Workflow --> Nodes[nodes: FlowNodeJSON[]]
    
    Nodes --> Node[FlowNodeJSON<br/>单个节点]
    
    Node --> NodeProps["• id: string<br/>• type: string<br/>• blocks?: FlowNodeJSON[]<br/>• data: { ... }"]
    
    NodeProps --> Data["data 属性"]
    Data --> DataFields["• title?: string<br/>• inputsValues?: Record<string, FlowValue><br/>• inputs?: JsonSchema<br/>• outputs?: JsonSchema<br/>• loopFor?: FlowValue"]
    
    DataFields --> FlowValue
    DataFields --> JsonSchema
    
    FlowValue[FlowValue<br/>数据值类型] --> Constant[constant<br/>常量类型]
    FlowValue --> Ref[ref<br/>引用类型]
    
    JsonSchema[JsonSchema<br/>定义数据结构] --> InputSchema[定义输入结构]
    JsonSchema --> OutputSchema[定义输出结构]
    
    style Workflow fill:#e1f5ff
    style Node fill:#fff3e0
    style FlowValue fill:#f3e5f5
    style JsonSchema fill:#e8f5e9
    style Constant fill:#c8e6c9
    style Ref fill:#ffccbc
```

### 工作流结构(WorkflowSchema)

工作流由节点(nodes)组成:

```typescript
interface WorkflowSchema {
  nodes: FlowNodeJSON[]  // 节点列表
}
```

### 节点结构(FlowNodeJSON)

每个节点包含以下字段:

```typescript
interface FlowNodeJSON {
  /** 节点唯一标识 */
  id: string
  /** 节点类型 (start/end/click/condition/loop等) */
  type: string
  /** 子节点数组 (用于嵌套结构,如循环节点、条件节点) */
  blocks?: FlowNodeJSON[]
  /** 节点数据 */
  data: {
    /** 节点标题 */
    title?: string
    /** 输入值配置 */
    inputsValues?: Record<string, FlowValue>
    /** 输入定义 (JSON Schema) */
    inputs?: JsonSchema
    /** 输出定义 (JSON Schema) */
    outputs?: JsonSchema
    /** 循环配置 (仅 loop 节点) */
    loopFor?: FlowValue
    /** 其他自定义属性 */
    [key: string]: unknown
  }
}
```

### JsonSchema

用于定义节点的输入和输出数据结构，兼容 JSON Schema 标准:

```typescript
interface JsonSchema {
  type?: 'string' | 'number' | 'integer' | 'boolean' | 'object' | 'array' | 'null'
  properties?: Record<string, JsonSchema>  // 对象属性
  items?: JsonSchema                       // 数组项定义
  required?: string[]                      // 必填字段
  enum?: unknown[]                         // 枚举值
  default?: unknown                        // 默认值
  description?: string                     // 字段描述
  extra?: Record<string, unknown>          // 扩展属性
  [key: string]: unknown
}
```

### 数据值类型(FlowValue)

节点的 `inputsValues` 支持两种值类型:

#### 1. 常量值(constant)

直接指定固定值:

```typescript
{
  type: 'constant',
  content: '登录',
  schema?: JsonSchema  // 可选的 schema 定义
}
```

示例:
```json
{
  "type": "constant",
  "content": "发布"
}
```

#### 2. 引用值(ref)

引用其他节点的输出:

```typescript
{
  type: 'ref',
  content: [nodeId, fieldName]  // [节点ID, 字段名]
}
```

示例:
```json
{
  "type": "ref",
  "content": ["start_0", "视频文件"]
}
```

**特殊引用**: 使用 `"init"` 作为节点 ID 可以引用开始节点（会自动映射到实际的开始节点 ID）:
```json
{
  "type": "ref",
  "content": ["init", "视频文件"]
}
```

### FlowValue 解析流程

```mermaid
flowchart TD
    Input["FlowValue 输入<br/>{ type: constant, content: 值 }<br/>或<br/>{ type: ref, content: [id, field] }"]
    
    Input --> Resolve[Executor.resolveValue]
    
    Resolve --> Check{type 类型?}
    
    Check -->|constant| DirectReturn[直接返回 content]
    Check -->|ref| RefProcess[引用解析流程]
    
    RefProcess --> Step1[1. 解析节点 ID]
    Step1 --> Step2[2. 查找 outputs nodeId]
    Step2 --> Step3[3. 返回 outputs fieldName]
    
    DirectReturn --> Result[实际值<br/>any type]
    Step3 --> Result
    
    style Input fill:#e3f2fd
    style Check fill:#fff9c4
    style DirectReturn fill:#c8e6c9
    style RefProcess fill:#ffccbc
    style Result fill:#4caf50,color:#fff
```

## 内置节点

### 节点执行逻辑

#### 条件节点 (condition)

```mermaid
flowchart TD
    Start[开始] --> Read[读取 conditions 数组<br/>{key:if_true, value:{...}}]
    Read --> Evaluate[依次评估每个条件]
    
    Evaluate --> Parse[解析 left FlowValue<br/>解析 right FlowValue<br/>应用 operator]
    
    Parse --> CheckResult{条件结果?}
    
    CheckResult -->|true| ExecuteBlock[执行对应 blocks i<br/>子节点]
    CheckResult -->|false| NextCondition[继续评估下一个条件]
    
    ExecuteBlock --> End[返回结果]
    NextCondition --> Evaluate
    
    style Start fill:#4caf50,color:#fff
    style CheckResult fill:#ff9800,color:#fff
    style ExecuteBlock fill:#2196f3,color:#fff
    style End fill:#4caf50,color:#fff
```

#### 循环节点 (loop)

```mermaid
flowchart TD
    Start[开始] --> CheckType{判断循环方式}
    
    CheckType -->|loopFor| ArrayLoop[数组循环<br/>items = 解析数组]
    CheckType -->|count| CountLoop[次数循环<br/>items = 0,1,2,...,n-1]
    
    ArrayLoop --> ForLoop
    CountLoop --> ForLoop
    
    ForLoop[for i in 0..items.length-1] --> SetVar[设置循环变量<br/>outputs nodeId_locals = {<br/>  item: items i,<br/>  index: i,<br/>  length: items.length<br/>}]
    
    SetVar --> ExecuteBlocks[执行 blocks 中的所有子节点<br/>for block in blocks:<br/>  executeNode block]
    
    ExecuteBlocks --> CheckNext{还有下一项?}
    CheckNext -->|是| ForLoop
    CheckNext -->|否| End[返回执行结果]
    
    style Start fill:#4caf50,color:#fff
    style CheckType fill:#ff9800,color:#fff
    style ForLoop fill:#2196f3,color:#fff
    style End fill:#4caf50,color:#fff
```

### 开始节点(start)

#### 功能

Start 节点是工作流的起始节点，用于接收工作流的输入数据并开始工作流的执行。每个工作流必须有且只有一个 Start 节点。

`outputs` 中定义的字段值由任务初始化时传入，为整个工作流提供初始数据。

#### 配置选项

| 选项 | 类型 | 必填 | 描述 |
|------|------|------|------|
| outputs | JsonSchema | 是 | 定义工作流的输入数据结构 |

#### 示例

```json
{
  "id": "start_0",
  "type": "start",
  "data": {
    "title": "设置初始化参数",
    "outputs": {
      "type": "object",
      "properties": {
        "视频文件": {
          "type": "string"
        },
        "视频描述": {
          "type": "string"
        }
      }
    }
  }
}
```

### 结束节点(end)

#### 功能

End 节点是工作流的结束节点，用于收集工作流的输出数据并结束工作流的执行。每个工作流必须有且只有一个 End 节点。

#### 配置选项

| 选项 | 类型 | 必填 | 描述 |
|------|------|------|------|
| inputs | JsonSchema | 是 | 定义工作流的输出数据结构 |
| inputsValues | Record<string, FlowValue> | 是 | 定义工作流的输出数据值，可以是引用或常量 |

#### 示例

```json
{
  "id": "end_0",
  "type": "end",
  "data": {
    "title": "结束",
    "inputsValues": {
      "success": {
        "type": "constant",
        "content": true
      }
    }
  }
}
```

### 条件节点(condition)

#### 功能

Condition 节点用于根据条件选择不同的执行分支，实现工作流的条件逻辑。

#### 配置选项

| 选项 | 类型 | 必填 | 描述 |
|------|------|------|------|
| conditions | Array | 是 | 条件数组，每个条件包含 key 和 value |

**条件 value 的结构:**

| 选项 | 类型 | 必填 | 描述 |
|------|------|------|------|
| left | FlowValue | 是 | 左值，可以是引用或常量 |
| operator | string | 是 | 操作符: `eq`(等于)、`ne`(不等于)、`gt`(大于)、`gte`(大于等于)、`lt`(小于)、`lte`(小于等于) |
| right | FlowValue | 是 | 右值，可以是引用或常量 |

#### 示例

```json
{
  "id": "condition_0",
  "type": "condition",
  "data": {
    "title": "条件判断",
    "conditions": [
      {
        "key": "if_true",
        "value": {
          "left": {
            "type": "ref",
            "content": ["start_0", "count"]
          },
          "operator": "gt",
          "right": {
            "type": "constant",
            "content": 10
          }
        }
      },
      {
        "key": "if_false",
        "value": {
          "left": {
            "type": "ref",
            "content": ["start_0", "count"]
          },
          "operator": "lte",
          "right": {
            "type": "constant",
            "content": 10
          }
        }
      }
    ]
  },
  "blocks": [
    {
      "id": "branch_1",
      "type": "waitTime",
      "data": {
        "title": "分支1"
      }
    },
    {
      "id": "branch_2",
      "type": "waitTime",
      "data": {
        "title": "分支2"
      }
    }
  ]
}
```

### 循环节点(loop)

#### 功能

Loop 节点用于对数组中的每个元素执行相同的操作，或执行指定次数的循环，实现工作流的循环逻辑。

#### 配置选项

| 选项 | 类型 | 必填 | 描述 |
|------|------|------|------|
| loopFor | FlowValue | 否 | 要迭代的数组（数组循环方式） |
| count | FlowValue | 否 | 循环次数（次数循环方式） |
| blocks | Array<FlowNodeJSON> | 是 | 循环体内的节点数组 |

**注意**: `loopFor` 和 `count` 二选一使用。

#### 循环变量

在循环体内，可以通过 `{循环节点ID}_locals` 引用以下变量：

| 变量 | 类型 | 描述 |
|------|------|------|
| item | any | 当前迭代项（数组循环）或当前索引（次数循环） |
| index | number | 当前迭代索引 |
| length | number | 数组长度或循环次数 |

#### 示例

**方式一: 数组循环**

```json
{
  "id": "loop_0",
  "type": "loop",
  "data": {
    "title": "遍历数组",
    "loopFor": {
      "type": "ref",
      "content": ["start_0", "items"]
    }
  },
  "blocks": [
    {
      "id": "click_1",
      "type": "click",
      "data": {
        "title": "点击项目",
        "inputsValues": {
          "selector": {
            "type": "ref",
            "content": ["loop_0_locals", "item"]
          }
        }
      }
    }
  ]
}
```

**方式二: 次数循环**

```json
{
  "id": "loop_0",
  "type": "loop",
  "data": {
    "title": "循环3次",
    "inputsValues": {
      "count": {
        "type": "constant",
        "content": 3
      }
    }
  },
  "blocks": [
    {
      "id": "waitTime_1",
      "type": "waitTime",
      "data": {
        "title": "等待"
      }
    }
  ]
}
```

## 动作节点

### 打开应用(openApp)

#### 功能

打开应用节点用于在 Android 设备上启动指定的应用程序，支持通过包名启动应用或通过 URI 启动应用的特定页面。

#### 配置选项

| 选项 | 类型 | 必填 | 描述 |
|------|------|------|------|
| packageName | string | 是 | 应用包名，如 `com.zhiliaoapp.musically` |
| uri | string | 否 | URI 地址，用于打开应用的特定页面 |

#### 输出

| 字段 | 类型 | 描述 |
|------|------|------|
| packageName | string | 已打开的应用包名 |

#### 示例

```json
{
  "id": "openApp_0",
  "type": "openApp",
  "data": {
    "title": "打开 TikTok",
    "inputsValues": {
      "packageName": {
        "type": "constant",
        "content": "com.zhiliaoapp.musically"
      },
      "uri": {
        "type": "constant",
        "content": ""
      }
    }
  }
}
```

### 等待元素出现(waitElement)

#### 功能

等待元素出现节点用于在 Android 设备上查找 UI 元素，支持通过多个属性组合精确定位元素，并返回元素对象供后续节点使用。

#### 配置选项

| 选项 | 类型 | 必填 | 描述 |
|------|------|------|------|
| selector | Array | 是 | 元素选择器数组，支持多个条件组合定位 |
| wait | number | 否 | 等待超时时间(毫秒)，默认 5000ms |
| includeInvisible | boolean | 否 | 是否包含不可见元素，默认 true |

**selector 条件结构 (新格式):**

```typescript
interface SelectorCondition {
  mode: 'text' | 'textContains' | 'class' | 'classContains' | 
        'fullId' | 'fullIdContains' | 'desc' | 'descContains'
  text: FlowValue  // 支持常量值或引用值
}
```

| mode 选项 | 说明 |
|----------|------|
| text | 文本完全匹配 |
| textContains | 文本包含匹配 |
| class | 类名完全匹配 |
| classContains | 类名包含匹配 |
| fullId | resourceId 完全匹配 |
| fullIdContains | resourceId 包含匹配 |
| desc | contentDescription 完全匹配 |
| descContains | contentDescription 包含匹配 |

#### 输出

| 字段 | 类型 | 描述 |
|------|------|------|
| element | Element | 找到的元素对象，可供后续节点引用 |

#### 示例

```json
{
  "id": "waitElement_0",
  "type": "waitElement",
  "data": {
    "title": "等待发布按钮出现",
    "inputsValues": {
      "selector": {
        "type": "constant",
        "content": [
          {
            "mode": "textContains",
            "text": {
              "type": "constant",
              "content": "发布"
            }
          },
          {
            "mode": "class",
            "text": {
              "type": "constant",
              "content": "android.widget.Button"
            }
          }
        ]
      },
      "wait": {
        "type": "constant",
        "content": 5000
      }
    }
  }
}
```

#### 使用场景

等待元素出现节点通常与点击元素节点配合使用:

1. 使用 `waitElement` 等待并返回元素对象
2. 使用 `click` 节点的 `selectorRef` 引用找到的元素进行点击

```json
{
  "nodes": [
    {
      "id": "waitElement_0",
      "type": "waitElement",
      "data": {
        "title": "等待发布按钮出现",
        "inputsValues": {
          "selector": {
            "type": "constant",
            "content": [
              {
                "mode": "textContains",
                "text": {
                  "type": "constant",
                  "content": "发布"
                }
              }
            ]
          }
        }
      }
    },
    {
      "id": "click_0",
      "type": "click",
      "data": {
        "title": "点击发布按钮",
        "inputsValues": {
          "selectorRef": {
            "type": "ref",
            "content": ["waitElement_0", "element"]
          },
          "actionEvent": {
            "type": "constant",
            "content": "click"
          }
        }
      }
    }
  ]
}
```

### 点击元素(click)

#### 功能

点击元素节点用于模拟用户在 Android 设备上的点击操作，支持通过多个属性组合精确定位 UI 元素。

#### 配置选项

| 选项 | 类型 | 必填 | 描述 |
|------|------|------|------|
| selector | Array | 否 | 元素选择器数组，支持多个条件组合定位 |
| selectorRef | Element | 否 | 引用其他节点返回的元素对象 |
| actionEvent | string | 是 | 点击类型: `click`(单击)、`dblclick`(双击)、`longclick`(长按) |
| pressTime | number | 否 | 长按持续时间(毫秒)，仅 actionEvent 为 longclick 时有效，默认 1000ms |
| wait | number | 否 | 等待超时时间(毫秒)，默认 5000ms |
| delayTime | number | 否 | 点击后延迟时间(毫秒) |

**注意**: `selector` 和 `selectorRef` 二选一，优先使用 `selectorRef`。

#### 输出

| 字段 | 类型 | 描述 |
|------|------|------|
| element | Element | 点击的元素对象 |

#### 示例

**方式一: 使用 selector 直接查找并点击**

```json
{
  "id": "click_0",
  "type": "click",
  "data": {
    "title": "点击发布按钮",
    "inputsValues": {
      "selector": {
        "type": "constant",
        "content": [
          {
            "mode": "textContains",
            "text": {
              "type": "constant",
              "content": "发布"
            }
          }
        ]
      },
      "actionEvent": {
        "type": "constant",
        "content": "click"
      }
    }
  }
}
```

**方式二: 使用 selectorRef 引用已查找的元素**

```json
{
  "id": "click_1",
  "type": "click",
  "data": {
    "title": "点击引用的元素",
    "inputsValues": {
      "selectorRef": {
        "type": "ref",
        "content": ["waitElement_0", "element"]
      },
      "actionEvent": {
        "type": "constant",
        "content": "click"
      }
    }
  }
}
```

**方式三: 长按操作**

```json
{
  "id": "click_2",
  "type": "click",
  "data": {
    "title": "长按元素",
    "inputsValues": {
      "selector": {
        "type": "constant",
        "content": [
          {
            "mode": "text",
            "text": {
              "type": "constant",
              "content": "长按我"
            }
          }
        ]
      },
      "actionEvent": {
        "type": "constant",
        "content": "longclick"
      },
      "pressTime": {
        "type": "constant",
        "content": 2000
      }
    }
  }
}
```

### 等待时间(waitTime)

#### 功能

等待时间节点用于在工作流执行过程中暂停指定的时间，常用于等待页面加载、动画完成或其他异步操作。

#### 配置选项

| 选项 | 类型 | 必填 | 描述 |
|------|------|------|------|
| waitTime | number | 是 | 等待时间(毫秒)，默认 3000ms |

#### 示例

```json
{
  "id": "waitTime_0",
  "type": "waitTime",
  "data": {
    "title": "等待3秒",
    "inputsValues": {
      "waitTime": {
        "type": "constant",
        "content": 3000
      }
    }
  }
}
```

### 滑动(swipe)

#### 功能

滑动节点用于模拟用户在屏幕上的滑动操作，支持自定义起点、终点和滑动时长。

#### 配置选项

| 选项 | 类型 | 必填 | 描述 |
|------|------|------|------|
| startX | number | 是 | 起点 X 坐标 |
| startY | number | 是 | 起点 Y 坐标 |
| endX | number | 是 | 终点 X 坐标 |
| endY | number | 是 | 终点 Y 坐标 |
| duration | number | 否 | 滑动时长(毫秒)，默认 300ms |
| delayTime | number | 否 | 滑动后延迟时间(毫秒) |

#### 示例

```json
{
  "id": "swipe_0",
  "type": "swipe",
  "data": {
    "title": "向上滑动",
    "inputsValues": {
      "startX": {
        "type": "constant",
        "content": 500
      },
      "startY": {
        "type": "constant",
        "content": 1000
      },
      "endX": {
        "type": "constant",
        "content": 500
      },
      "endY": {
        "type": "constant",
        "content": 300
      },
      "duration": {
        "type": "constant",
        "content": 500
      }
    }
  }
}
```

### 键盘操作(keyboard)

#### 功能

键盘操作节点用于模拟键盘按键操作，如回车、删除等。

#### 配置选项

| 选项 | 类型 | 必填 | 描述 |
|------|------|------|------|
| actionType | string | 是 | 按键类型: `enter`(回车)、`delete`(删除) |
| delayTime | number | 否 | 操作后延迟时间(毫秒) |

#### 示例

```json
{
  "id": "keyboard_0",
  "type": "keyboard",
  "data": {
    "title": "按回车键",
    "inputsValues": {
      "actionType": {
        "type": "constant",
        "content": "enter"
      },
      "delayTime": {
        "type": "constant",
        "content": 1000
      }
    }
  }
}
```

### 页面后退(backPage)

#### 功能

页面后退节点用于模拟 Android 设备的返回键操作。

#### 配置选项

| 选项 | 类型 | 必填 | 描述 |
|------|------|------|------|
| delayTime | number | 否 | 后退后延迟时间(毫秒) |

#### 示例

```json
{
  "id": "backPage_0",
  "type": "backPage",
  "data": {
    "title": "返回上一页",
    "inputsValues": {
      "delayTime": {
        "type": "constant",
        "content": 500
      }
    }
  }
}
```

### 关闭应用(closeApp)

#### 功能

关闭应用节点用于强制关闭指定的应用程序。

#### 配置选项

| 选项 | 类型 | 必填 | 描述 |
|------|------|------|------|
| packageName | string | 是 | 应用包名 |

#### 示例

```json
{
  "id": "closeApp_0",
  "type": "closeApp",
  "data": {
    "title": "关闭 TikTok",
    "inputsValues": {
      "packageName": {
        "type": "constant",
        "content": "com.zhiliaoapp.musically"
      }
    }
  }
}
```

## 执行器使用

### ExecutorParams 参数说明

| 参数 | 类型 | 必填 | 描述 |
|------|------|------|------|
| locale | string | 是 | 语言环境，如 `zh-CN`、`en-US` |
| workflowUrl | string | 否 | 工作流 JSON 文件 URL（与 workflow 二选一） |
| workflow | WorkflowSchema | 否 | 工作流 JSON 数据（与 workflowUrl 二选一） |
| initialParams | Record<string, unknown> | 是 | 任务初始化参数，将传递给开始节点 |

### 使用方式

#### 方式一: 从 URL 加载工作流

```typescript
const params: ExecutorParams = {
  locale: 'zh-CN',
  workflowUrl: 'https://example.com/workflow.json',
  initialParams: {
    '视频文件': 'https://example.com/video.mp4',
    '视频描述': '测试视频'
  }
}

const executor = new Executor(params)
executor.execute()
```

#### 方式二: 直接传入工作流数据

```typescript
const params: ExecutorParams = {
  locale: 'zh-CN',
  workflow: {
    nodes: [
      // 工作流节点定义
    ]
  },
  initialParams: {
    '视频文件': 'https://example.com/video.mp4'
  }
}

const executor = new Executor(params)
executor.execute()
```

### 执行报告

执行完成后，执行器会生成并输出执行报告：

```typescript
interface ExecutionReport {
  /** 执行状态 */
  status: 'success' | 'failed'
  /** 开始时间 */
  startTime: number
  /** 结束时间 */
  endTime: number
  /** 总耗时(毫秒) */
  duration: number
  /** 节点执行记录 */
  nodeRecords: NodeExecutionRecord[]
  /** 错误信息 */
  error?: string
}
```

每个节点的执行记录包含：

```typescript
interface NodeExecutionRecord {
  /** 节点 ID */
  nodeId: string
  /** 节点类型 */
  nodeType: string
  /** 节点标题 */
  nodeTitle?: string
  /** 执行状态 */
  status: 'success' | 'failed' | 'skipped'
  /** 开始时间(时间戳) */
  startTime: number
  /** 结束时间(时间戳) */
  endTime: number
  /** 执行耗时(毫秒) */
  duration: number
  /** 输入数据 */
  inputs?: Record<string, unknown>
  /** 输出数据 */
  outputs?: Record<string, unknown>
  /** 错误信息 */
  error?: string
}
```

## 动作函数开发

### 开发流程

```mermaid
flowchart TD
    Step1["步骤 1: 定义错误码<br/>━━━━━━━━━━━━━━━<br/>文件: src/shared/error-definitions.ts<br/><br/>export const ErrorCode = {<br/>  'MY_ACTION/PARAM_REQUIRED': '...',<br/>  'MY_ACTION/EXECUTION_FAILED': '...',<br/>}<br/><br/>export const ErrorMessage = {<br/>  'MY_ACTION/PARAM_REQUIRED': '缺少必要参数',<br/>  'MY_ACTION/EXECUTION_FAILED': '执行失败',<br/>}"]
    
    Step2["步骤 2: 创建动作函数<br/>━━━━━━━━━━━━━━━<br/>文件: src/actions/my-action.ts<br/><br/>export function myAction(<br/>  node: FlowNodeJSON,<br/>  context: ExecutionContext<br/>): ActionResult {<br/>  // 1. 获取输入参数<br/>  const inputs = node.data.inputs<br/>  // 2. 参数验证<br/>  if (!inputs.param1) {<br/>    throwActionError('MY_ACTION/PARAM_REQUIRED')<br/>  }<br/>  // 3. 执行动作逻辑<br/>  const result = doSomething(inputs.param1)<br/>  // 4. 返回结果<br/>  return { result }<br/>}"]
    
    Step3["步骤 3: 注册动作函数<br/>━━━━━━━━━━━━━━━<br/>文件: src/actions/index.ts<br/><br/>import { myAction } from './my-action'<br/><br/>export const ACTION_REGISTRY = {<br/>  click,<br/>  openApp,<br/>  myAction,  // 添加新动作<br/>}"]
    
    Step4["步骤 4: 在工作流中使用<br/>━━━━━━━━━━━━━━━<br/>{<br/>  id: myAction_0,<br/>  type: myAction,<br/>  data: {<br/>    title: 执行自定义动作,<br/>    inputsValues: {<br/>      param1: {<br/>        type: constant,<br/>        content: 测试参数<br/>      }<br/>    }<br/>  }<br/>}"]
    
    Step5["步骤 5: 测试与验证<br/>━━━━━━━━━━━━━━━<br/>• 构建项目: npm run build<br/>• 运行测试: 使用 demo.ts 或在云手机平台测试<br/>• 验证输出: 检查 ExecutionReport 中的节点记录"]
    
    Step1 --> Step2
    Step2 --> Step3
    Step3 --> Step4
    Step4 --> Step5
    
    style Step1 fill:#e3f2fd
    style Step2 fill:#f3e5f5
    style Step3 fill:#fff9c4
    style Step4 fill:#e8f5e9
    style Step5 fill:#4caf50,color:#fff
```

### 动作函数执行机制

```mermaid
flowchart TD
    Start[Executor.executeNode] --> Parse["1. 解析 inputsValues<br/>• resolveValue FlowValue<br/>• 处理 constant / ref<br/>• 生成 inputs 对象"]
    
    Parse --> Get["2. 获取动作函数<br/>• getActionFunction node.type<br/>• 从 ACTION_REGISTRY 查找"]
    
    Get --> Inject["3. 注入数据到 node.data.inputs<br/>• node.data.inputs = inputs"]
    
    Inject --> Call["4. 调用动作函数<br/>• actionFn node, context"]
    
    Call --> Internal["动作函数内部执行:<br/>• 获取 inputs<br/>• 参数验证<br/>• 执行平台 API<br/>• 返回 ActionResult"]
    
    Internal --> Handle["5. 处理返回值<br/>• 提取 result.result<br/>• 保存到 context.outputs"]
    
    Handle --> Record["6. 记录执行报告<br/>• NodeExecutionRecord<br/>• status / duration / outputs"]
    
    Record --> End([完成])
    
    style Start fill:#4caf50,color:#fff
    style Call fill:#ff9800,color:#fff
    style Internal fill:#2196f3,color:#fff
    style End fill:#4caf50,color:#fff
```

### 开发规范

执行器采用插件化架构，可以轻松扩展新的动作节点。每个动作节点由一个动作函数实现。

#### 动作函数签名

```typescript
type NodeActionFunction = (
  node: FlowNodeJSON,
  context: ExecutionContext
) => ActionResult | void
```

**参数说明:**

- `node`: 节点定义，包含节点配置和输入数据
- `context`: 执行上下文，包含语言环境、节点输出存储等

**返回值:**

- `ActionResult`: 包含 `result` 字段的对象，`result` 为节点的输出数据
- `void`: 无返回值，表示该节点不输出数据

#### 输入数据获取

动作函数通过 `node.data.inputs` 获取已解析的输入数据：

```typescript
export function myAction(
  node: FlowNodeJSON,
  context: ExecutionContext
): ActionResult {
  // 获取已解析的输入数据
  const inputs = node.data.inputs as Record<string, unknown>
  
  // 访问具体参数
  const param1 = inputs.param1 as string
  const param2 = inputs.param2 as number
  
  // 执行动作逻辑
  // ...
  
  // 返回结果
  return {
    result: {
      success: true,
      data: 'some data'
    }
  }
}
```

**注意**: 执行器会在调用动作函数前解析 `inputsValues` 中的所有 `FlowValue`，并将解析后的实际值存入 `node.data.inputs`。

#### 错误处理

使用 `ActionError` 类抛出结构化错误：

```typescript
import { throwActionError } from '../shared/action-error'

export function myAction(
  node: FlowNodeJSON,
  context: ExecutionContext
): ActionResult {
  const inputs = node.data.inputs as Record<string, unknown>
  
  if (!inputs.requiredParam) {
    // 使用预定义的错误码
    throwActionError('MY_ACTION/PARAM_REQUIRED', {
      param: 'requiredParam'
    })
  }
  
  // 继续执行...
}
```

**错误码格式**: `动作名/错误描述`，如 `OPEN_APP/START_FAILED`

#### 错误定义

在 `src/shared/error-definitions.ts` 中定义错误码和错误信息：

```typescript
export const ErrorCode = {
  // 添加新的错误码
  'MY_ACTION/PARAM_REQUIRED': 'MY_ACTION/PARAM_REQUIRED',
  'MY_ACTION/EXECUTION_FAILED': 'MY_ACTION/EXECUTION_FAILED',
} as const

export const ErrorMessage: Record<ErrorCodeKey, string> = {
  // 添加错误信息
  'MY_ACTION/PARAM_REQUIRED': '缺少必要参数',
  'MY_ACTION/EXECUTION_FAILED': '执行失败',
}
```

### 开发步骤

#### 1. 创建动作函数文件

在 `src/actions/` 目录下创建新的动作函数文件：

```typescript
// src/actions/my-action.ts
import type { FlowNodeJSON } from '../types/flow'
import type { ActionResult, ExecutionContext } from '../types/executor'
import { throwActionError } from '../shared/action-error'

/**
 * 我的自定义动作
 * @param node - 节点定义
 * @param context - 执行上下文
 * @returns 动作执行结果
 */
export function myAction(
  node: FlowNodeJSON,
  context: ExecutionContext
): ActionResult {
  const inputs = node.data.inputs as Record<string, unknown>
  
  // 参数验证
  const param1 = inputs.param1 as string
  if (!param1) {
    throwActionError('MY_ACTION/PARAM_REQUIRED', { param: 'param1' })
  }
  
  // 执行动作逻辑
  console.log('执行自定义动作:', param1)
  const result = doSomething(param1)
  
  // 返回结果
  return {
    result: {
      success: true,
      data: result
    }
  }
}

function doSomething(param: string): string {
  // 实现具体逻辑
  return `处理结果: ${param}`
}
```

#### 2. 注册动作函数

在 `src/actions/index.ts` 中注册新的动作函数：

```typescript
import { myAction } from './my-action'

export const ACTION_REGISTRY: Record<string, NodeActionFunction> = {
  click,
  openApp,
  waitElement,
  waitTime,
  backPage,
  closeApp,
  keyboard,
  swipe,
  myAction,  // 注册新动作
}
```

#### 3. 使用动作节点

在工作流 JSON 中使用新的动作节点：

```json
{
  "id": "myAction_0",
  "type": "myAction",
  "data": {
    "title": "执行自定义动作",
    "inputsValues": {
      "param1": {
        "type": "constant",
        "content": "测试参数"
      }
    }
  }
}
```

### 最佳实践

1. **参数验证**: 在函数开始时验证所有必要参数
2. **错误处理**: 使用 `ActionError` 抛出结构化错误，便于调试
3. **日志输出**: 使用 `console.log` 输出关键信息，方便问题排查
4. **类型安全**: 为输入参数添加类型断言，确保类型安全
5. **返回值规范**: 如果动作有输出，使用 `ActionResult` 包装返回值
6. **无副作用**: 避免修改传入的 `node` 和 `context` 对象（除非必要）

### 示例：查找元素动作

这是一个完整的动作函数示例，展示了最佳实践：

```typescript
import type { FlowNodeJSON } from '../types/flow'
import type { ActionResult, ExecutionContext } from '../types/executor'
import { findElementBySelector } from './utils/shared'
import { convertSelector, type NewSelectorCondition } from './utils/selector-converter'

export function waitElement(
  node: FlowNodeJSON,
  context: ExecutionContext,
): ActionResult {
  const inputs = node.data.inputs as Record<string, unknown>

  // 转换新格式选择器
  const rawSelector = inputs.selector as NewSelectorCondition[]
  const selector = convertSelector(rawSelector, context)

  const wait = (inputs.wait as number) || 5000
  const includeInvisible = inputs.includeInvisible !== false

  // 使用共享函数查找元素
  const element = findElementBySelector(selector, { wait, includeInvisible })

  // 返回找到的元素
  return {
    result: element,
  }
}
```
## 类型定义

### 核心类型

#### ExecutorParams

执行器初始化参数:

```typescript
interface ExecutorParams {
  /** 语言环境 */
  locale: string
  /** 工作流 URL（与 workflow 二选一） */
  workflowUrl?: string
  /** 工作流 JSON 数据（与 workflowUrl 二选一） */
  workflow?: WorkflowSchema
  /** 任务初始化参数 */
  initialParams: Record<string, unknown>
}
```

#### WorkflowSchema

工作流定义:

```typescript
interface WorkflowSchema {
  /** 节点列表 */
  nodes: FlowNodeJSON[]
}
```

#### FlowNodeJSON

节点定义:

```typescript
interface FlowNodeJSON {
  /** 节点唯一标识 */
  id: string
  /** 节点类型 */
  type: string
  /** 子节点数组（用于嵌套结构） */
  blocks?: FlowNodeJSON[]
  /** 节点数据 */
  data: {
    /** 节点标题 */
    title?: string
    /** 输入值配置 */
    inputsValues?: Record<string, FlowValue>
    /** 输入定义 (JSON Schema) */
    inputs?: JsonSchema
    /** 输出定义 (JSON Schema) */
    outputs?: JsonSchema
    /** 循环配置 (仅 loop 节点) */
    loopFor?: FlowValue
    /** 其他自定义属性 */
    [key: string]: unknown
  }
}
```

#### FlowValue

流程值类型，支持常量和引用:

```typescript
type FlowValue =
  | { 
      type: 'constant'
      content: unknown
      schema?: JsonSchema
    }
  | { 
      type: 'ref'
      content: string[]  // [节点ID, 字段名]
    }
```

#### JsonSchema

JSON Schema 定义:

```typescript
interface JsonSchema {
  type?: 'string' | 'number' | 'integer' | 'boolean' | 'object' | 'array' | 'null'
  properties?: Record<string, JsonSchema>
  items?: JsonSchema
  required?: string[]
  enum?: unknown[]
  default?: unknown
  description?: string
  extra?: Record<string, unknown>
  [key: string]: unknown
}
```

### 执行相关类型

#### ExecutionContext

执行上下文:

```typescript
interface ExecutionContext {
  /** 语言环境 */
  locale: string
  /** 节点输出存储 */
  outputs: Map<string, Record<string, unknown>>
  /** 初始化参数 */
  initialParams: Record<string, unknown>
  /** 开始节点 ID */
  startNodeId?: string
}
```

#### ExecutionReport

执行报告:

```typescript
interface ExecutionReport {
  /** 执行状态 */
  status: 'success' | 'failed'
  /** 开始时间 */
  startTime: number
  /** 结束时间 */
  endTime: number
  /** 总耗时 */
  duration: number
  /** 节点执行记录 */
  nodeRecords: NodeExecutionRecord[]
  /** 错误信息 */
  error?: string
}
```

#### NodeExecutionRecord

节点执行记录:

```typescript
interface NodeExecutionRecord {
  /** 节点 ID */
  nodeId: string
  /** 节点类型 */
  nodeType: string
  /** 节点标题 */
  nodeTitle?: string
  /** 执行状态 */
  status: 'success' | 'failed' | 'skipped'
  /** 开始时间(时间戳) */
  startTime: number
  /** 结束时间(时间戳) */
  endTime: number
  /** 执行耗时(毫秒) */
  duration: number
  /** 输入数据 */
  inputs?: Record<string, unknown>
  /** 输出数据 */
  outputs?: Record<string, unknown>
  /** 错误信息 */
  error?: string
}
```

### 动作相关类型

#### NodeActionFunction

动作函数类型:

```typescript
type NodeActionFunction = (
  node: FlowNodeJSON,
  context: ExecutionContext
) => ActionResult | void
```

#### ActionResult

动作执行结果:

```typescript
interface ActionResult {
  /** 执行结果，如果动作没有返回值则为 null */
  result: unknown
}
```

#### ActionError

动作错误类:

```typescript
class ActionError extends Error {
  /** 错误码，格式: 动作名/错误描述 */
  code: string
  /** 额外的错误信息 */
  info?: any
  
  constructor(message: string, code: string, info?: any)
}
```

### Executor 类方法

#### 构造函数

```typescript
constructor(params: ExecutorParams)
```

创建执行器实例。

**参数:**
- `params`: 执行器参数

**抛出:**
- `Error`: 当参数不合法时

#### execute()

```typescript
execute(): void
```

执行整个工作流。

**流程:**
1. 加载工作流（从 URL 或直接使用传入的工作流数据）
2. 验证工作流（检查开始节点和结束节点）
3. 按顺序执行所有节点
4. 生成并上报执行报告

**抛出:**
- `Error`: 当工作流执行失败时

## 工作流验证规则

执行器会在执行前验证工作流:

1. **开始节点**: 必须有且只有一个类型为 `start` 的节点
2. **结束节点**: 必须有且只有一个类型为 `end` 的节点
3. **节点引用**: 所有 `ref` 类型的引用必须指向存在的节点

## 注意事项

1. **同步执行**: 执行器采用同步方式执行，适配云手机脚本引擎（不使用 Promise/async/await）
2. **错误处理**: 任何节点执行失败都会中断工作流并抛出异常
3. **数据流转**: 节点通过 `ref` 引用其他节点的输出，形成数据流
4. **节点顺序**: 节点按照 `nodes` 数组的顺序依次执行
5. **循环变量**: 循环节点会创建 `{loopNodeId}_locals` 作为循环变量存储
6. **初始参数**: 支持传入简单值（会自动转换为 FlowValue）或直接传入 FlowValue 格式

## 相关链接

- [TypeScript 官网](https://www.typescriptlang.org/)
- [JSON Schema 规范](https://json-schema.org/)

## License

MIT License
