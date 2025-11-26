# 云手机自动化流程执行器

## 项目简介

本项目专注于云手机自动化流程的数据格式定义和流程执行器的实现,基于 JSON Schema 驱动的节点化流程编排。

## 核心功能

- **流程数据格式定义**: 标准化的 JSON 流程格式,支持节点嵌套和数据流转
- **流程执行引擎**: 解析和执行流程定义,支持条件分支、循环等控制流
- **类型安全**: 完整的 TypeScript 类型定义,确保流程数据的正确性

## 流程数据格式

### 基础结构

流程由节点(nodes)组成,每个节点包含:
- `id`: 节点唯一标识
- `type`: 节点类型(start/end/click/swipe/input/if/loop等)
- `blocks`: 子节点数组(用于嵌套结构)
- `data`: 节点数据,包含 title、inputsValues、inputs、outputs

### 数据值类型

节点的 `inputsValues` 支持三种值类型:
- `constant`: 常量值 `{ type: "constant", content: value }`
- `ref`: 引用其他节点输出 `{ type: "ref", content: ["nodeId", "field"] }`
- `template`: 模板字符串 `{ type: "template", content: "text_{{variable}}" }`

### 完整示例

```json
{
  "nodes": [
    {
      "id": "start_0",
      "type": "start",
      "blocks": [],
      "data": {
        "title": "Start",
        "outputs": {
          "type": "object",
          "properties": {
            "targetApp": {
              "type": "string",
              "default": "com.ss.android.ugc.aweme"
            },
            "elements": {
              "type": "array",
              "items": {
                "type": "object",
                "properties": {
                  "selector": { "type": "string" },
                  "enabled": { "type": "boolean" }
                }
              }
            }
          }
        }
      }
    },
    {
      "id": "click_0",
      "type": "click",
      "blocks": [],
      "data": {
        "title": "点击屏幕中心",
        "inputsValues": {
          "selector": {
            "type": "constant",
            "content": {
              "type": "coordinate",
              "x": "50%",
              "y": "50%"
            }
          },
          "clickType": {
            "type": "constant",
            "content": "double"
          }
        },
        "inputs": {
          "type": "object",
          "required": ["selector", "clickType"],
          "properties": {
            "selector": { "type": "object" },
            "clickType": { "type": "string", "enum": ["single", "double", "long"] }
          }
        },
        "outputs": {
          "type": "object",
          "properties": {
            "success": { "type": "boolean" },
            "timestamp": { "type": "number" }
          }
        }
      }
    },
    {
      "id": "if_0",
      "type": "if",
      "data": {
        "title": "检查点击是否成功",
        "inputsValues": {
          "condition": {
            "type": "ref",
            "content": ["click_0", "success"]
          }
        },
        "inputs": {
          "type": "object",
          "required": ["condition"],
          "properties": {
            "condition": { "type": "boolean" }
          }
        }
      },
      "blocks": [
        {
          "id": "if_true_0",
          "type": "ifBlock",
          "data": { "title": "true" },
          "blocks": []
        },
        {
          "id": "if_false_0",
          "type": "ifBlock",
          "data": { "title": "false" },
          "blocks": []
        }
      ]
    },
    {
      "id": "loop_0",
      "type": "loop",
      "data": {
        "title": "循环点击元素列表",
        "loopFor": {
          "type": "ref",
          "content": ["start_0", "elements"]
        }
      },
      "blocks": [
        {
          "id": "click_element_0",
          "type": "click",
          "data": {
            "title": "点击当前元素",
            "inputsValues": {
              "selector": {
                "type": "ref",
                "content": ["$item", "selector"]
              },
              "clickType": {
                "type": "constant",
                "content": "single"
              }
            }
          }
        }
      ]
    },
    {
      "id": "end_0",
      "type": "end",
      "blocks": [],
      "data": {
        "title": "End",
        "inputsValues": {
          "success": {
            "type": "constant",
            "content": true,
            "schema": { "type": "boolean" }
          }
        }
      }
    }
  ]
}
```

## 操作节点数据结构

### 点击节点 (click)

```typescript
{
  "id": "click_0",
  "type": "click",
  "data": {
    "title": "点击操作",
    "inputsValues": {
      "selector": {
        "type": "constant",
        "content": {
          "type": "coordinate",  // 或 "id" | "xpath" | "text" | "image"
          "x": "50%",
          "y": "50%"
        }
      },
      "clickType": {
        "type": "constant",
        "content": "single"  // "single" | "double" | "long"
      },
      "delay": {
        "type": "constant",
        "content": 300
      }
    },
    "inputs": {
      "type": "object",
      "required": ["selector", "clickType"],
      "properties": {
        "selector": { "type": "object" },
        "clickType": { "type": "string", "enum": ["single", "double", "long"] },
        "delay": { "type": "number" }
      }
    },
    "outputs": {
      "type": "object",
      "properties": {
        "success": { "type": "boolean" },
        "timestamp": { "type": "number" }
      }
    }
  }
}
```

### 滑动节点 (swipe)

```json
{
  "id": "swipe_0",
  "type": "swipe",
  "data": {
    "title": "滑动操作",
    "inputsValues": {
      "direction": { "type": "constant", "content": "up" },
      "distance": { "type": "constant", "content": "80%" },
      "duration": { "type": "constant", "content": 300 }
    },
    "inputs": {
      "type": "object",
      "required": ["direction"],
      "properties": {
        "direction": { "type": "string", "enum": ["up", "down", "left", "right"] },
        "distance": { "type": "string" },
        "duration": { "type": "number" }
      }
    }
  }
}
```

### 输入节点 (input)

```json
{
  "id": "input_0",
  "type": "input",
  "data": {
    "title": "输入文本",
    "inputsValues": {
      "selector": {
        "type": "constant",
        "content": { "type": "id", "value": "com.example:id/search" }
      },
      "text": { "type": "ref", "content": ["start_0", "keyword"] },
      "clearBefore": { "type": "constant", "content": true }
    },
    "inputs": {
      "type": "object",
      "required": ["selector", "text"],
      "properties": {
        "selector": { "type": "object" },
        "text": { "type": "string" },
        "clearBefore": { "type": "boolean" }
      }
    }
  }
}
```

## TypeScript 类型定义

完整类型定义见 `types/flow.ts`,核心类型包括:

```typescript
// 流程文档
interface FlowDocumentJSON {
  nodes: FlowNodeJSON[];
}

// 流程节点
interface FlowNodeJSON {
  id: string;
  type: string;
  blocks?: FlowNodeJSON[];
  data: {
    title?: string;
    inputsValues?: Record<string, FlowValue>;
    inputs?: JsonSchema;
    outputs?: JsonSchema;
    [key: string]: any;
  };
}

// 流程值类型
type FlowValue = 
  | { type: 'constant'; content: any; schema?: JsonSchema }
  | { type: 'ref'; content: string[] }
  | { type: 'template'; content: string };

// 流程执行上下文
interface FlowContext {
  instanceId: string;
  flow: FlowDocumentJSON;
  variables: Map<string, any>;
  currentNodeId?: string;
  status: FlowStatus;
  nodeExecutions: Map<string, NodeExecution>;
}

// 节点执行器接口
interface NodeExecutor {
  type: string;
  execute(node: FlowNodeJSON, context: FlowContext): Promise<any>;
  validate?(node: FlowNodeJSON): boolean;
}

// 流程引擎接口
interface FlowEngine {
  loadFlow(flow: FlowDocumentJSON): void;
  startFlow(variables?: Record<string, any>): Promise<FlowContext>;
  pauseFlow(instanceId: string): Promise<void>;
  resumeFlow(instanceId: string): Promise<void>;
  stopFlow(instanceId: string): Promise<void>;
  registerExecutor(executor: NodeExecutor): void;
}
```

## 流程执行器实现

### 执行器接口

每个节点类型需要实现 `NodeExecutor` 接口:

```typescript
class ClickNodeExecutor implements NodeExecutor {
  type = 'click';
  
  async execute(node: FlowNodeJSON, context: FlowContext): Promise<any> {
    const { selector, clickType, delay } = this.resolveInputs(node, context);
    
    // 执行点击操作
    const result = await this.performClick(selector, clickType);
    
    // 延迟
    if (delay) {
      await sleep(delay);
    }
    
    return {
      success: result.success,
      timestamp: Date.now()
    };
  }
  
  validate(node: FlowNodeJSON): boolean {
    // 验证节点配置
    return node.data?.inputsValues?.selector && 
           node.data?.inputsValues?.clickType;
  }
  
  private resolveInputs(node: FlowNodeJSON, context: FlowContext) {
    // 解析节点输入值(处理 constant/ref/template)
    // ...
  }
  
  private async performClick(selector: any, clickType: string) {
    // 执行实际点击操作
    // ...
  }
}
```

### 引擎实现

```typescript
class FlowEngineImpl implements FlowEngine {
  private executors = new Map<string, NodeExecutor>();
  
  registerExecutor(executor: NodeExecutor): void {
    this.executors.set(executor.type, executor);
  }
  
  async startFlow(variables?: Record<string, any>): Promise<FlowContext> {
    const context = this.createContext(variables);
    await this.executeNodes(this.flow.nodes, context);
    return context;
  }
  
  private async executeNodes(nodes: FlowNodeJSON[], context: FlowContext) {
    for (const node of nodes) {
      const executor = this.executors.get(node.type);
      if (!executor) {
        throw new Error(`No executor for node type: ${node.type}`);
      }
      
      context.currentNodeId = node.id;
      const result = await executor.execute(node, context);
      context.variables.set(node.id, result);
      
      // 处理子节点
      if (node.blocks?.length) {
        await this.executeNodes(node.blocks, context);
      }
    }
  }
}
```

## 项目结构

```
rpa-executor/
├── types/
│   └── flow.ts              # TypeScript 类型定义
├── examples/
│   └── simple-flow.json     # 流程示例
├── src/
│   ├── engine/
│   │   ├── flow-engine.ts   # 流程引擎实现
│   │   └── context.ts       # 执行上下文
│   ├── executors/
│   │   ├── click.ts         # 点击节点执行器
│   │   ├── swipe.ts         # 滑动节点执行器
│   │   ├── input.ts         # 输入节点执行器
│   │   ├── if.ts            # 条件节点执行器
│   │   └── loop.ts          # 循环节点执行器
│   └── utils/
│       ├── value-resolver.ts # 值解析器(处理 constant/ref/template)
│       └── selector.ts       # 元素选择器
└── README.md
```

