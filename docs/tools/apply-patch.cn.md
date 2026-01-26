---
summary: "Apply multi-file patches with the apply_patch tool"
read_when:
  - You need structured file edits across multiple files
  - You want to document or debug patch-based edits
---

# apply_patch 工具

使用结构化补丁格式应用文件更改。这对于单个 `edit` 调用会很脆弱的多文件或多块编辑是理想的。

工具接受单个 `input` 字符串，包装一个或多个文件操作：

```
*** Begin Patch
*** Add File: path/to/file.txt
+line 1
+line 2
*** Update File: src/app.ts
@@
-old line
+new line
*** Delete File: obsolete.txt
*** End Patch
```

## 参数

- `input`（必需）：完整补丁内容，包括 `*** Begin Patch` 和 `*** End Patch`。

## 说明

- 路径相对于工作区根解析。
- 在 `*** Update File:` 块内使用 `*** Move to:` 来重命名文件。
- `*** End of File` 在需要时标记仅 EOF 插入。
- 实验性且默认禁用。使用 `tools.exec.applyPatch.enabled` 启用。
- 仅 OpenAI（包括 OpenAI Codex）。可选地通过 `tools.exec.applyPatch.allowModels` 按模型门控。
- 配置仅在 `tools.exec` 下。

## 示例

```json
{
  "tool": "apply_patch",
  "input": "*** Begin Patch\n*** Update File: src/index.ts\n@@\n-const foo = 1\n+const foo = 2\n*** End Patch"
}
```
