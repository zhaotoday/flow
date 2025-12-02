# 云手机自动化工作流执行器

> 基于 FlowGram 规范的云手机自动化工作流执行引擎

[![TypeScript](https://img.shields.io/badge/TypeScript-5.5.4-blue.svg)](https://www.typescriptlang.org/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](./LICENSE)

## 📚 文档导航

- 🚀 [快速开始](./QUICKSTART.md) - 5 分钟上手指南
- 📖 [使用指南](./USAGE.md) - 详细的使用文档和示例
- 🏗️ [架构设计](./ARCHITECTURE.md) - 技术架构和扩展开发
- 📝 [更新日志](./CHANGELOG.md) - 版本历史和计划
- 📊 [项目总结](./SUMMARY.md) - 项目概览和特性总结
- 💡 [示例代码](./examples/) - 完整的示例工作流

## 项目简介

本项目专注于云手机自动化工作流的数据格式定义和工作流执行器的实现,基于 JSON Schema 驱动的节点化工作流编排。

## 核心功能

- **工作流数据格式定义**: 标准化的 JSON 工作流格式,支持节点嵌套和数据流转
- **工作流执行引擎**: 解析和执行工作流定义,支持条件分支、循环等控制流
- **类型安全**: 完整的 TypeScript 类型定义,确保工作流数据的正确性

## 工作流数据格式

### 工作流结构(WorkflowSchema)

工作流由节点(nodes)组成:

```typescript
interface WorkflowSchema {
  nodes: WorkflowNodeSchema[];  // 节点列表
}
```

### 节点结构(WorkflowNodeSchema)

每个节点包含以下字段:

- `id`: string - 节点唯一标识
- `type`: string - 节点类型(start/end/click/condition/loop等)
- `data`: 节点数据,包含以下字段:
  - `title`: string - 节点标题
  - `inputs`: JSONSchema - 输入数据结构定义
  - `inputsValues`: Record<string, ValueSchema> - 输入值配置
  - `outputs`: JSONSchema - 输出数据结构定义
- `blocks`: WorkflowNodeSchema[] - 子节点数组(用于嵌套结构,如循环节点、条件节点)

### JSONSchema

用于定义节点的输入和输出数据结构,兼容 JSON Schema 标准:

```typescript
interface JSONSchema {
  type: 'string' | 'number' | 'boolean' | 'object' | 'array' | 'file';
  title?: string;           // 字段名称
  description?: string;     // 字段描述
  default?: any;            // 默认值
  enum?: (string | number)[]; // 枚举值
  properties?: Record<string, JSONSchema>; // 对象属性
  items?: JSONSchema;       // 数组项定义
  required?: string[];      // 必填字段
}
```

### 数据值类型(ValueSchema)

节点的 `inputsValues` 支持四种值类型:

- **常量值(constant)**: 直接指定固定值
  ```json
  { "type": "constant", "content": "登录" }
  ```

- **引用值(ref)**: 引用其他节点的输出
  ```json
  { "type": "ref", "content": ["nodeId", "fieldName"] }
  ```

## 节点数据结构

### 内置节点

#### 开始节点(start)

##### 功能

Start 节点是工作流的起始节点,用于接收工作流的输入数据并开始工作流的执行。每个工作流必须有且只有一个 Start 节点。

`outputs` 中定义的字段值由任务初始化时传入,为整个工作流提供初始数据。

##### 配置选项

| 选项 | 类型 | 必填 | 描述 |
|------|------|------|------|
| outputs | JSONSchema | 是 | 定义工作流的输入数据结构 |

##### 示例

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
          "type": "file"
        },
        "视频描述": {
          "type": "string"
        },
        "发布按钮文字": {
          "type": "string"
        }
      }
    }
  }
}
```

#### 结束节点(end)

##### 功能

End 节点是工作流的结束节点,用于收集工作流的输出数据并结束工作流的执行。每个工作流必须有至少一个 End 节点。

##### 配置选项

| 选项 | 类型 | 必填 | 描述 |
|------|------|------|------|
| inputs | JSONSchema | 是 | 定义工作流的输出数据结构 |
| inputsValues | Record<string, ValueSchema> | 是 | 定义工作流的输出数据值,可以是引用或常量 |

##### 示例

```json
{
  "id": "end_0",
  "type": "结束",
  "blocks": [],
  "data": {
    "title": "End",
    "inputsValues": {
      "success": {
        "type": "constant",
        "content": true,
        "schema": {
          "type": "boolean"
        }
      }
    }
  }
}
```

#### 条件节点(condition)

##### 功能

Condition 节点用于根据条件选择不同的执行分支,实现工作流的条件逻辑。

##### 配置选项

| 选项 | 类型 | 必填 | 描述 |
|------|------|------|------|
| conditions | Array | 是 | 条件数组,每个条件包含 key 和 value |

**条件 value 的结构:**

| 选项 | 类型 | 必填 | 描述 |
|------|------|------|------|
| left | ValueSchema | 是 | 左值,可以是引用或常量 |
| operator | string | 是 | 操作符,如 "eq"、"gt" 等 |
| right | ValueSchema | 是 | 右值,可以是引用或常量 |

##### 示例

```json
{
  "id": "condition_0",
  "type": "condition",
  "data": {
    "title": "If 条件",
    "conditions": [
      {
        "key": "if_true",
        "value": {
          "left": {
            "type": "ref",
            "content": ["start_0", "value"]
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
            "content": ["start_0", "value"]
          },
          "operator": "lte",
          "right": {
            "type": "constant",
            "content": 10
          }
        }
      }
    ]
  }
}
```

#### 循环节点(loop)

##### 功能

Loop 节点用于对数组中的每个元素执行相同的操作,实现工作流的循环逻辑。

##### 配置选项

| 选项 | 类型 | 必填 | 描述 |
|------|------|------|------|
| loopFor | ValueSchema | 是 | 要迭代的数组,通常是一个引用 |
| loopOutputs | Record<string, ValueSchema> | 是 | 循环输出,引用子节点的变量 |
| blocks | Array<NodeSchema> | 是 | 循环体内的节点数组 |

##### 示例

```json
{
  "id": "loop_0",
  "type": "loop",
  "data": {
    "title": "For 循环",
    "loopFor": {
      "type": "ref",
      "content": ["start_0", "items"]
    },
    "loopOutputs": {
      "results": {
        "type": "ref",
        "content": ["llm_1", "result"]
      }
    }
  },
  "blocks": [
    {
      "id": "llm_1",
      "type": "llm",
      "data": {
        "inputsValues": {
          "prompt": {
            "type": "ref",
            "content": ["loop_0_locals", "item"]
          }
        }
      }
    }
  ]
}
```

### 动作节点

#### 打开应用(openApp)

##### 功能

打开应用节点用于在 Android 设备上启动指定的应用程序,支持通过包名启动应用或通过 URI 启动应用的特定页面。

##### 配置选项

| 选项 | 类型 | 必填 | 描述 |
|------|------|------|------|
| packageName | string | 是 | 应用包名,如 `com.zhiliaoapp.musically` |
| uri | string | 否 | URI 地址,用于打开应用的特定页面 |

##### 示例

```json
{
  "id": "openApp_0",
  "type": "openApp",
  "data": {
    "title": "打开 TikTok",
    "inputs": {
      "type": "object",
      "required": [
        "packageName"
      ],
      "properties": {
        "packageName": {
          "type": "string",
          "label": "包名",
          "default": "com.zhiliaoapp.musically"
        },
        "uri": {
          "type": "string",
          "label": "URI",
          "default": ""
        }
      }
    },
    "inputsValues": {
      "packageName": {
        "type": "constant",
        "content": "com.zhiliaoapp.musically"
      },
      "uri": {
        "type": "constant",
        "content": ""
      }
    },
    "outputs": {
      "type": "object",
      "properties": {
        "result": {
          "type": "string"
        }
      }
    }
  },
  "blocks": []
}
```

#### 点击元素(click)

##### 功能

点击元素节点用于模拟用户在 Android 设备上的点击操作,支持通过多个属性组合精确定位 UI 元素。

##### 配置选项

| 选项 | 类型 | 必填 | 描述 |
|------|------|------|------|
| selector | Object | 是 | 元素选择器,支持多个属性组合定位 |
| clickType | string | 是 | 点击类型: single(单击)、double(双击)、long(长按) |
| delay | number | 否 | 点击后延迟时间(毫秒) |

**selector 支持的属性:**
- `className`: Android 组件类名,如 `android.widget.Button`
- `resourceId`: 资源 ID,如 `com.example.app:id/button_login`
- `text`: 元素文本内容
- `contentDescription`: 内容描述(无障碍描述)

至少需要提供一个选择器属性,支持多个属性组合以提高定位精确度。

##### 示例

```json
{
  "id": "click_0",
  "type": "click",
  "data": {
    "title": "点击发布按钮",
    "inputs": {
      "type": "object",
      "required": [
        "clickType"
      ],
      "properties": {
        "selector": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["attribute", "condition"],
            "properties": {
              "attribute": {
                "type": "string",
                "enum": ["text", "contentDescription", "resourceId", "className"]
              },
              "condition": {
                "type": "string",
                "enum": ["equals", "includes"]
              },
              "value": {
                "type": "string|ref"
              }
            }
          }
        },
        "selectorRef": {
          "type": "object",
          "label": "元素引用"
        },
        "clickType": {
          "type": "string",
          "enum": [
            "single",
            "double",
            "long"
          ]
        }
      }
    },
    "inputsValues": {
      "selector": {
        "type": "object",
        "content": [
          {
            "attribute": "text",
            "condition": "includes",
            "value": {
              "type": "ref",
              "content": [
                "start_0",
                "发布按钮文字"
              ]
            }
          },
          {
            "attribute": "contentDescription",
            "condition": "equals",
            "value": {
              "type": "constant",
              "content": "发布"
            }
          }
        ]
      },
      "selectorRef":  {
        "type": "ref",
        "content": [
          "findElement_0",
          "发布按钮"
        ]
      },
      "clickType": {
        "type": "constant",
        "content": "single"
      }
    },
    "outputs": {
      "type": "object",
      "properties": {
        "result": {
          "type": "string"
        }
      }
    }
  }
}
```

## 执行器使用

### 快速开始

```typescript
import { Executor } from '@hubstudio/rpa-executor'

// 执行整个工作流
const params = {
  id: '123456',
  locale: 'zh-CN',
  workflowTemplateUrl: 'https://example.com/workflow-template.json',
  initialParams: {
    videoUrl: 'https://example.com/video.mp4'
  }
}

const executor = new Executor(params)
executor.execute()
```

### 执行器参数

#### ExecutorParams

| 参数 | 类型 | 必填 | 描述 |
|------|------|------|------|
| id | string | 是 | 任务 ID |
| locale | string | 是 | 语言环境,如 `zh-CN`、`en-US` |
| workflowTemplateUrl | string | 是 | 工作流模板 JSON 文件 URL |
| initialParams | Record<string, unknown> | 是 | 任务初始化参数,将传递给开始节点 |
| runNode | { id: string } | 否 | 指定执行的节点 ID(可选) |

### 执行模式

#### 1. 执行整个工作流

执行从开始节点到结束节点的完整工作流:

```typescript
import { Executor } from '@hubstudio/rpa-executor'
import type { ExecutorParams } from '@hubstudio/rpa-executor'

const params: ExecutorParams = {
  id: '123456',
  locale: 'zh-CN',
  workflowTemplateUrl: 'https://example.com/workflow-template.json',
  initialParams: {
    videoUrl: 'https://example.com/video.mp4',
    items: ['item1', 'item2', 'item3'],
    threshold: 75
  }
}

const executor = new Executor(params)
executor.execute()
```

#### 2. 执行指定节点

仅执行工作流中的某个特定节点:

```typescript
const params: ExecutorParams = {
  id: '789012',
  locale: 'en-US',
  workflowTemplateUrl: 'https://example.com/workflow-template.json',
  initialParams: {
    videoUrl: 'https://example.com/video.mp4'
  },
  runNode: {
    id: 'click_0'
  }
}

const executor = new Executor(params)
executor.executeSpecificNode()
```

### 执行报告

执行完成后,执行器会生成并输出执行报告:

```typescript
interface ExecutionReport {
  /** 任务 ID */
  taskId: string
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

每个节点的执行记录包含:

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

### 动作节点扩展

执行器采用插件化架构,可以轻松扩展新的动作节点:

```typescript
// src/actions/custom-action.ts
import type { FlowNodeJSON } from '../types/flow'
import type { ExecutionContext } from '../types/executor'

export function customAction(
  node: FlowNodeJSON,
  inputs: Record<string, unknown>,
  context: ExecutionContext
): Record<string, unknown> {
  // 实现自定义动作逻辑
  console.log('执行自定义动作:', inputs)
  
  return {
    success: true,
    result: 'custom result'
  }
}
```

```typescript
// src/actions/index.ts
import { customAction } from './custom-action'

export const ACTION_REGISTRY: Record<string, NodeActionFunction> = {
  click,
  openApp,
  customAction  // 注册新动作
}
```

### 工作流验证

执行器会在执行前验证工作流:

- 必须包含唯一的开始节点(id=`start_0`)
- 必须包含唯一的结束节点(id=`end_0`)
- 所有引用的节点必须存在
- 节点配置必须符合 Schema 定义

### 示例工作流

完整的测试示例见 `examples/workflow-template.json`,包含:

- **开始节点**: 接收初始参数
- **条件节点**: If 条件判断
- **循环节点**: For 循环遍历
- **动作节点**: 点击元素
- **结束节点**: 输出执行结果

### 注意事项

1. **同步执行**: 执行器不使用 Promise/async/await,采用同步方式执行,适配云手机脚本引擎
2. **错误处理**: 任何节点执行失败都会中断工作流并抛出异常
3. **数据流转**: 节点通过 `ref` 引用其他节点的输出,形成数据流
4. **国际化**: 执行器会打印 `locale` 参数,但暂未实现完整的国际化功能
