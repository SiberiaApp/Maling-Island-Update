---
name: maling-island
description: 连接并操作码灵属 MCP 服务，处理取码相关工作流。当用户要求列出当前未完成的取件码、取餐码、电影票、票务或随手记，查看最后一次识别结果，将记录重新发布到 Android Live Update 或小米超级岛，通过码灵属截取安卓屏幕，不依赖码灵属内置在线模型而由 Agent 识别一条或多条记录，或将结构化 orders JSON 提交给码灵属保存并上岛时使用。用户提到查询未完成取码、重新上岛、识别当前截图、把截图交给 Agent 识别、提交取餐码/取件码/电影票、码灵属或码灵属 MCP 时触发。
---

# 码灵属

使用已配置的 `maling-island` MCP 服务处理码灵属中的记录和识别任务。用户要求 Agent 识别时，在 Agent 内完成视觉识别，不要调用应用中配置的在线模型。

## 开始操作

1. 确认码灵属 MCP 工具或资源可用。
2. 如果不可用，提示用户在应用中开启 `设置 -> 模型设置 -> AI Agent（MCP）`，再将应用复制的本机配置或局域网配置添加到 MCP 客户端。
3. MCP 连接已经配置时，禁止索取、打印、记录或暴露 Bearer 令牌。
4. 需要确认工具参数、资源 URI、订单字段、连接方法或错误处理细节时，查阅本文后半部分的“MCP 接口参考”。

## 选择工作流

### 处理磁贴或无障碍快捷识别任务

当 Agent 作为码灵属的快捷识别后端运行时，持续执行以下循环：

1. 调用 `wait_for_recognition_task`，建议将 `timeoutSeconds` 设为 `20`。
2. 返回 `task=null` 时立即开始下一次等待，不要退出轮询。
3. 返回任务时，读取同一次返回中的 `recognitionPrompt`、截图和 `taskId`。
4. 使用 Agent 的视觉能力识别截图，严格按照 `recognitionPrompt` 生成一个或多个订单。
5. 调用 `submit_recognition_results`，同时传入完整 `orders` JSON 字符串和原任务的 `taskId`。
6. 图片中确实没有有效记录或识别无法完成时，调用 `fail_recognition_task`，传入 `taskId` 和简短原因。

禁止在该工作流中调用 `capture_current_screen`，因为任务返回的图片就是磁贴或无障碍刚截取的屏幕。禁止调用码灵属内置在线模型。必须带回 `taskId`，否则码灵属无法关闭该任务的“正在识别”状态和清理截图。

### 查询已有记录

调用 `list_unfinished_pickups` 回答当前未完成记录相关问题。客户端更适合读取资源时，使用 `maling://pickups/unfinished`。

只汇总实际返回的数据。禁止推测缺失的取码、品牌、地点、日期或时间。

仅在用户要求查看最近一次识别结果时读取 `maling://recognition/latest`。

### 重新上岛

1. 确认用户指定的取码、品牌或地点。
2. 请求存在歧义时，先调用 `list_unfinished_pickups`，根据返回记录消除歧义。
3. 使用范围尽可能准确的 `query` 调用 `republish_pickup`。
4. 告知用户匹配数量和实际处理的记录。

除非用户明确要求重新发布全部未完成记录，否则禁止传入空查询。重新上岛会产生可见副作用，发布 Live Update 或小米超级岛通知。

### 识别当前安卓屏幕

依次执行：

1. 调用 `get_recognition_prompt`；如果工具不可用但资源可用，读取 `maling://prompt/recognition`。
2. 调用 `capture_current_screen`。
3. 使用 Agent 的视觉能力检查返回的图片。
4. 严格遵循获取到的提示词，每个独立取码、票务或随手记生成一个订单对象。
5. 将完整 JSON 对象序列化为字符串，通过 `orders` 参数调用 `submit_recognition_results`。
6. 告知用户码灵属保存并上岛了多少条记录。

此工作流禁止调用应用内配置的在线模型。`capture_current_screen` 只使用应用当前配置的截图方式；MCP 后台截图要求 Shizuku 或 Root。

### 识别 Agent 已经收到的图片

1. 用户或其他工具已经提供图片时，不要调用 `capture_current_screen`。
2. 调用 `get_recognition_prompt` 获取当前提示词。
3. 检查每张输入图片并遵循提示词规则。
4. 通过 `submit_recognition_results` 提交识别 JSON。

图片中没有有效记录时，不要提交虚构数据，直接说明没有识别到有效订单。

## 构造识别结果

每个独立记录生成一个对象。多个记录放入同一个 `orders` 数组。

使用以下通用格式：

```json
{
  "orders": [
    {
      "code": "001",
      "type": "饮品",
      "brand": "古茗",
      "fullText": "图片中与该订单相关的文字",
      "pickupLocation": "万达店",
      "qrPayload": null
    }
  ]
}
```

电影票需要包含 `day` 和 `time`，将电影名称写入 `code`，将影厅和座位写入 `pickupLocation`。

将 `code` 视为必填字段。无法确认的可选字段使用 `null`，禁止猜测。只有二维码内容已经被实际解码或明确提供时才能设置 `qrPayload`；图片中仅仅出现二维码，不代表已经获得二维码内容。

调用 `submit_recognition_results` 时，将整个 JSON 对象作为字符串传入。禁止在该字符串外添加 Markdown 代码围栏或解释文字。

## 处理错误

- `401 Unauthorized`：已配置的 Bearer 令牌缺失或失效；让用户从码灵属重新复制完整配置。
- 连接被拒绝：检查应用内 MCP 开关；USB 本机模式还需要执行 `adb forward tcp:8765 tcp:8765`。
- 局域网连接超时：确认手机和 Agent 设备位于同一局域网；手机 IP 变化后重新复制局域网配置；检查路由器是否开启客户端隔离。
- 截图失败：提示用户在码灵属中选择并授权 Shizuku 或 Root。后台 MCP 调用无法发起媒体投影授权。
- `orders` JSON 无效：重新构造合法 JSON，确保 `orders` 数组不为空，并且至少包含一个有效 `code`。
- 任务处理失败：调用 `fail_recognition_task`，不要直接丢弃任务；否则任务会保持处理中，直到领取租约超时。

## MCP 接口参考

### 连接

- 传输方式：Streamable HTTP
- 默认本机地址：`http://127.0.0.1:8765/mcp`
- 认证方式：`Authorization: Bearer <token>`
- USB 端口转发：`adb forward tcp:8765 tcp:8765`
- 局域网地址：手机 IPv4 地址可能发生变化，应从应用中复制当前配置。

客户端配置示例：

```json
{
  "mcpServers": {
    "maling-island": {
      "type": "streamable-http",
      "url": "http://127.0.0.1:8765/mcp",
      "headers": {
        "Authorization": "Bearer <token>"
      }
    }
  }
}
```

从 `设置 -> 模型设置 -> AI Agent（MCP）` 获取完整配置。不要让用户把令牌粘贴到对话中。

### 资源

- `maling://prompt/recognition`：返回当前保存的识别提示词和二维码字段规则。每次识别前读取一次。
- `maling://pickups/unfinished`：以 JSON 格式返回全部未完成记录。
- `maling://recognition/latest`：返回码灵属保存的最后一次识别结果 JSON。

### `get_recognition_prompt`

参数：无。返回与 `maling://prompt/recognition` 相同的当前提示词。

### `list_unfinished_pickups`

参数：无。以 JSON 格式返回全部未完成记录。

### `republish_pickup`

参数：

```json
{
  "query": "取码、品牌或地点的部分文本"
}
```

匹配不区分大小写，检查 `code`、`brand` 和地点。空查询会匹配全部未完成记录，只能在用户明确要求时使用。

### `list_pending_recognition_tasks`

参数：无。返回磁贴或无障碍快捷方式创建、尚未完成的 Agent 识别任务。该工具只用于查看状态；常驻 Agent 应使用 `wait_for_recognition_task` 领取任务。

### `wait_for_recognition_task`

参数：

```json
{
  "timeoutSeconds": 20
}
```

最长等待 25 秒并领取最早的可用任务。返回内容包含任务元数据、当前识别提示词和 JPEG 截图。没有任务时返回 `{"task":null}`。任务领取后有 5 分钟租约，租约到期后可以重新领取。

### `fail_recognition_task`

参数：

```json
{
  "taskId": "wait_for_recognition_task 返回的任务 ID",
  "error": "图片中没有有效取码"
}
```

关闭该任务的“正在识别”状态并清理任务截图。无法产生有效 `orders` 时必须调用此工具。

### `capture_current_screen`

参数：无。通过码灵属当前配置的截图方式返回 JPEG 图片。MCP 后台截图支持 Shizuku 和 Root，不会弹出媒体投影授权界面。

### `submit_recognition_results`

参数：

```json
{
  "taskId": "可选的快捷识别任务 ID",
  "orders": "{\"orders\":[{\"code\":\"001\",\"type\":\"饮品\",\"brand\":\"古茗\",\"pickupLocation\":\"万达店\"}]}"
}
```

`orders` 是包含 JSON 对象的字符串，不是直接嵌套的数组。每个有效订单会单独保存，并根据码灵属当前选择的通知方式单独发布通知。

处理 `wait_for_recognition_task` 返回的任务时必须传入对应 `taskId`。码灵属会在提交成功后关闭“正在识别”状态、清理任务截图，并将本地解码出的二维码内容补到首条缺少 `qrPayload` 的订单中。Agent 主动调用 `capture_current_screen` 时不需要 `taskId`。

### 支持的订单字段

- `code`：必填；取码、随手记标题或电影名称
- `type`：用于判断取餐、取件、电影票、票务或随手记类型
- `brand`：商家、快递公司、影院、场馆或品牌
- `fullText`：来源文本上下文
- `pickupLocation`：门店、取件地点、地址、影厅或座位
- `day`：电影日期
- `time`：电影场次时间，推荐格式为 `HH:mm-HH:mm`
- `qrPayload`：解码后的二维码内容

解析器还支持以下别名：

- `code`：`pickupCode`、`pickup_code`、`number`、`ticketName`、`movieName`
- `type`：`category`、`kind`
- `brand`：`shop`、`merchant`、`carrier`
- `pickupLocation`：`location`、`store`、`address`、`seat`
- `day`：`date`、`movieDay`
- `time`：`movieTime`、`showTime`
- `qrPayload`：`qr`、`qrcode`、`qrCode`、`qrUrl`

即使支持别名，也应优先使用标准字段。

### 类型映射

- `type` 包含 `快递`、`取件` 或 `包裹` 时，创建取件记录。
- `type` 包含 `电影票`、`电影` 或 `影院` 时，创建电影票记录。
- `type` 包含 `票务`、`演唱`、`音乐节` 或 `场馆` 时，创建票务记录。
- `type` 包含 `随手记` 或 `笔记` 时，创建随手记记录。
- 其他值创建取餐记录。
