# 云手机自动化流程执行器

## 项目简介

本项目是一个面向云手机平台的 RPA(Robotic Process Automation)自动化流程执行引擎,旨在提供高效、灵活、可扩展的自动化任务编排与执行能力。

## 核心特性

- **可视化流程编排**: 支持拖拽式流程设计,直观的图形化界面
- **丰富的节点类型**: 支持开始/结束节点、操作节点、条件分支、循环等多种节点
- **JSON Schema 驱动**: 通过 JSON Schema 定义节点输入输出,类型安全
- **云手机适配**: 专为云手机环境优化,支持批量设备管理与并发执行
- **灵活编排**: 提供丰富的操作指令集,支持复杂业务场景的流程编排

## 流程模板格式

### 简单示例

以下是一个包含 Start 节点、点击操作节点、If 条件节点和 Loop 循环节点的完整流程示例:

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
            "clickCount": {
              "type": "number",
              "default": 5
            },
            "elements": {
              "type": "array",
              "items": {
                "type": "object",
                "properties": {
                  "selector": {
                    "type": "string"
                  },
                  "enabled": {
                    "type": "boolean"
                  }
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
            "selector": {
              "type": "object",
              "description": "元素定位信息"
            },
            "clickType": {
              "type": "string",
              "enum": ["single", "double", "long"],
              "description": "点击类型"
            },
            "delay": {
              "type": "number",
              "description": "点击后延迟(ms)"
            }
          }
        },
        "outputs": {
          "type": "object",
          "properties": {
            "success": {
              "type": "boolean"
            },
            "timestamp": {
              "type": "number"
            }
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
            "condition": {
              "type": "boolean"
            }
          }
        }
      },
      "blocks": [
        {
          "id": "if_true_0",
          "type": "ifBlock",
          "data": {
            "title": "true"
          },
          "blocks": []
        },
        {
          "id": "if_false_0",
          "type": "ifBlock",
          "data": {
            "title": "false"
          },
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
            },
            "inputs": {
              "type": "object",
              "required": ["selector", "clickType"],
              "properties": {
                "selector": {
                  "type": "object"
                },
                "clickType": {
                  "type": "string"
                }
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
            "schema": {
              "type": "boolean"
            }
          }
        }
      }
    }
  ]
}
```

### 点击操作节点详细数据结构

```typescript
// 点击节点完整示例
const clickNodeExample: FlowNodeJSON = {
  id: "click_like_button",
  type: "click",
  blocks: [],
  data: {
    title: "点赞按钮",
    inputsValues: {
      // 元素选择器 - 支持多种定位方式
      selector: {
        type: "constant",
        content: {
          // 方式1: 坐标定位
          type: "coordinate",
          x: "50%",  // 支持百分比或像素值
          y: "50%"
        }
        // 方式2: 元素ID定位
        // type: "constant",
        // content: {
        //   type: "id",
        //   value: "com.example:id/like_button"
        // }
        // 方式3: XPath定位
        // type: "constant",
        // content: {
        //   type: "xpath",
        //   value: "//android.widget.Button[@text='点赞']"
        // }
        // 方式4: 文本定位
        // type: "constant",
        // content: {
        //   type: "text",
        //   value: "点赞",
        //   exact: false  // 是否精确匹配
        // }
      },
      
      // 点击类型
      clickType: {
        type: "constant",
        content: "double"  // single | double | long
      },
      
      // 点击延迟
      delay: {
        type: "constant",
        content: 300  // 毫秒
      },
      
      // 可选: 点击前等待元素出现
      waitFor: {
        type: "constant",
        content: {
          enabled: true,
          timeout: 5000,  // 超时时间
          checkInterval: 500  // 检查间隔
        }
      },
      
      // 可选: 失败重试
      retry: {
        type: "constant",
        content: {
          enabled: true,
          maxRetries: 3,
          retryInterval: 1000
        }
      }
    },
    
    // 输入定义
    inputs: {
      type: "object",
      required: ["selector", "clickType"],
      properties: {
        selector: {
          type: "object",
          description: "元素定位信息"
        },
        clickType: {
          type: "string",
          enum: ["single", "double", "long"],
          description: "点击类型: single-单击, double-双击, long-长按"
        },
        delay: {
          type: "number",
          description: "点击后延迟时间(ms)",
          default: 0
        },
        waitFor: {
          type: "object",
          description: "等待配置"
        },
        retry: {
          type: "object",
          description: "重试配置"
        }
      }
    },
    
    // 输出定义
    outputs: {
      type: "object",
      properties: {
        success: {
          type: "boolean",
          description: "是否点击成功"
        },
        timestamp: {
          type: "number",
          description: "执行时间戳"
        },
        elementFound: {
          type: "boolean",
          description: "元素是否找到"
        },
        error: {
          type: "string",
          description: "错误信息(如果失败)"
        }
      }
    }
  }
};
```

### 其他常用操作节点示例

#### 滑动节点
```json
{
  "id": "swipe_0",
  "type": "swipe",
  "data": {
    "title": "上滑刷新",
    "inputsValues": {
      "direction": {
        "type": "constant",
        "content": "up"
      },
      "distance": {
        "type": "constant",
        "content": "80%"
      },
      "duration": {
        "type": "constant",
        "content": 300
      }
    },
    "inputs": {
      "type": "object",
      "required": ["direction"],
      "properties": {
        "direction": {
          "type": "string",
          "enum": ["up", "down", "left", "right"]
        },
        "distance": {
          "type": "string",
          "description": "滑动距离(支持百分比)"
        },
        "duration": {
          "type": "number",
          "description": "滑动时长(ms)"
        }
      }
    }
  }
}
```

#### 输入文本节点
```json
{
  "id": "input_0",
  "type": "input",
  "data": {
    "title": "输入搜索关键词",
    "inputsValues": {
      "selector": {
        "type": "constant",
        "content": {
          "type": "id",
          "value": "com.example:id/search_input"
        }
      },
      "text": {
        "type": "ref",
        "content": ["start_0", "keyword"]
      },
      "clearBefore": {
        "type": "constant",
        "content": true
      }
    },
    "inputs": {
      "type": "object",
      "required": ["selector", "text"],
      "properties": {
        "selector": {
          "type": "object"
        },
        "text": {
          "type": "string"
        },
        "clearBefore": {
          "type": "boolean",
          "description": "输入前是否清空"
        }
      }
    }
  }
}
```

#### 截图节点
```json
{
  "id": "screenshot_0",
  "type": "screenshot",
  "data": {
    "title": "截图保存",
    "inputsValues": {
      "filename": {
        "type": "template",
        "content": "screenshot_{{timestamp}}.png"
      },
      "quality": {
        "type": "constant",
        "content": 90
      }
    },
    "outputs": {
      "type": "object",
      "properties": {
        "path": {
          "type": "string"
        },
        "size": {
          "type": "number"
        }
      }
    }
  }
}
```
