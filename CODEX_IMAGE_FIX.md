# AxonHub Codex 绘图修复补丁

> 适用于 `looplj/axonhub` 的 Codex 渠道图片生成支持。
> 如果你的 AxonHub 允许用户通过 Codex 渠道调用 `/v1/images/generations` 生成图片，需要此补丁。

## 背景

上游 `looplj/axonhub` 在 [68672716](https://github.com/looplj/axonhub/commit/68672716) 提交了 `feat: codex image gen/edit`，提供了 Codex 渠道原生生图支持。但该实现依赖 SSE 聚合器（`responses/aggregator.go`）处理 `response.output_item.done` 事件来提取图片数据。然而 **Codex 的实际 SSE 流中，图片结果没有被聚合器正确捕获**，导致响应解析失败：

```
failed to transform response: codex image response did not contain image_generation_call result
```

## 修改内容

**仅修改 1 个文件：`llm/transformer/openai/codex/outbound.go`**

### 改动 1：`codexExecutor.Do` — 图片请求走 custom 提取

在 SSE chunk 收集完成后、调用 `AggregateStreamChunks` 之前，对 image 请求先用 `findImageResult` 扫描原始 chunk 数据：

```go
// For image requests, scan SSE chunks for image_generation_call result
// before normal aggregation. Codex returns the image data in
// response.output_item.done events which the standard aggregator may miss.
if request.RequestType == llm.RequestTypeImage.String() {
    if result := findImageResult(chunks); result != "" {
        return &httpclient.Response{
            StatusCode: http.StatusOK,
            Headers:    http.Header{"Content-Type": []string{"application/json"}},
            Body:       []byte(result),
            Request:    request,
        }, nil
    }
}
```

### 改动 2：`findImageResult` — 新增函数

直接解析 SSE chunk 的原始 JSON，扫描 `response.output_item.done` 事件中 `item.type == "image_generation_call"` 的结果，提取 `item.result`（base64 图片数据），包装为 `BuildImageResponse` 可识别的 JSON 格式：

```go
func findImageResult(chunks []*httpclient.StreamEvent) string {
    for _, ch := range chunks {
        var ev map[string]any
        if err := json.Unmarshal(ch.Data, &ev); err != nil {
            continue
        }
        if t, _ := ev["type"].(string); t == "response.output_item.done" {
            item, ok := ev["item"].(map[string]any)
            if !ok {
                continue
            }
            if itemType, _ := item["type"].(string); itemType == "image_generation_call" {
                if result, ok := item["result"].(string); ok && result != "" {
                    outFormat := "png"
                    if f, ok := item["output_format"].(string); ok && f != "" {
                        outFormat = f
                    }
                    return fmt.Sprintf(
                        `{"output":[{"type":"image_generation_call","result":"%s","output_format":"%s","status":"completed"}]}`,
                        result, outFormat,
                    )
                }
            }
        }
    }
    return ""
}
```

## 额外修复（可选的增强）

**文件：`llm/transformer/openai/responses/aggregator.go`**

1. **`OutputItemDone` 中创建缺失 item**：Codex 可能跳过 `output_item.added` 直接发送 `output_item.done`，导致 item 不存在。添加 item 创建逻辑：

   ```go
   if item == nil {
       item = newAggregatedItem()
       a.outputItems[ev.OutputIndex] = append(a.outputItems[ev.OutputIndex], item)
   }
   ```

2. **`OutputItemDone` 中传播 type**：确保即使没有 `output_item.added`，item.Type 也能正确设置：

   ```go
   if ev.Item.Type != "" {
       item.Type = ev.Item.Type
   }
   ```

3. **新增 `StreamEventTypeImageGenerationCompleted` 处理器**：处理 Codex 可能使用的独立 image completed 事件。

> ⚠️ 这些修复**不必须**，因为 `findImageResult` 已经绕过了聚合器。保留它们是为了向上游提交 PR 时说明完整的修复方案。

## 如何检查上游是否已修复

下次拉取上游更新时，检查以下几点：

### 1. 检查聚合器是否处理 image 事件

```bash
grep -n "ImageGenerationCompleted\|image_generation_call" llm/transformer/openai/responses/aggregator.go
```

如果 `processEvent` 的 switch 中有 `case StreamEventTypeImageGenerationCompleted:` 处理器，说明上游可能已修复。

### 2. 直接测试生图（最重要的验证）

```bash
curl -s -X POST http://127.0.0.1:8090/v1/images/generations \
  -H "Authorization: Bearer <API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-image-2","prompt":"a cat","n":1,"size":"1024x1024","response_format":"b64_json"}'
```

如果返回 200 且包含 `b64_json` 数据 → **上游已修复，可移除此补丁。**
如果返回 500 `codex image response did not contain image_generation_call result` → **仍需此补丁。**

### 3. 如果上游已修复

只需从 `outbound.go` 中移除以下内容并恢复上游版本：

1. 删除 `codexExecutor.Do` 中 `request.RequestType == llm.RequestTypeImage.String()` 的图片扫描代码块
2. 删除 `findImageResult` 函数
3. 恢复 `TransformResponse` 为上游版本（不需要 fallback）

可以用 `git diff upstream/unstable -- llm/transformer/openai/codex/outbound.go` 对比差异。

## 部署配置要求

代码修复后，还需确保渠道配置正确：

| 配置项 | 要求 |
|--------|------|
| Codex 渠道类型 | Codex（OAuth 认证正常） |
| 支持的模型 | 包含 `gpt-5.5`（或通过模型映射 `gpt-image-2` → `gpt-5.5`） |
| 自定义端点 | **不要添加** `image_generation` 自定义端点 |

## 相关提交

| Commit | 说明 |
|--------|------|
| `68672716` | 上游 `feat: codex image gen/edit` |
| 本补丁 | 修复上游聚合器无法处理 Codex SSE 图片事件的问题 |
