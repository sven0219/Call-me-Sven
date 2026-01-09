---
title: "分步指南：在 Kibana 中配置索引生命周期策略"
date: 2025-04-10 13:00:42+08:00
draft: false
tags: ["es","lifecycle"]
Categories: ["devops"]
summary: "为索引添加生命周期策略"
showtoc: true
---

以下是在 Kibana 中配置索引生命周期策略的步骤：

### 步骤 1：确认当前索引是否已有别名

使用以下 API 检查索引别名：

```bash
GET /rollover-demo-k8s-docker/_alias
```

如果返回结果中没有别名，需要先为索引添加别名。

### 步骤 2：为索引添加别名

通过 Elasticsearch API 添加别名，并设置为写入索引：

```bash
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "rollover-demo-k8s-docker",
        "alias": "rollover-alias",
        "is_write_index": true
      }
    }
  ]
}
```

将 `rollover-alias` 替换为你选择的别名名称。

### 步骤 3：确保索引名称符合 Rollover 命名规则

Rollover 要求索引名称以数字结尾（例如 `myindex-000001`），以便自动生成后续索引。如果当前索引不符合该格式：

1. **重新创建索引**：使用正确的命名格式（例如 `rollover-alias-000001`）。

2. **创建索引模板**（推荐）：

   ```bash
   PUT /_index_template/rollover-template
   {
     "index_patterns": ["rollover-alias-*"],
     "template": {
       "settings": {
         "index.lifecycle.name": "rollover-7days",
         "index.lifecycle.rollover_alias": "rollover-alias"
       },
       "aliases": {
         "rollover-alias": {}
       }
     },
     "priority": 200
   }
   ```

3. **初始化索引**：

   ```bash
   PUT /rollover-alias-000001
   {
     "aliases": {
       "rollover-alias": {
         "is_write_index": true
       }
     }
   }
   ```

### 步骤 4：应用生命周期策略

在 Kibana 中将策略 `rollover-7days` 绑定到别名 `rollover-alias`，而不是直接绑定到索引名。

### 步骤 5：验证配置

• 确保别名正确指向索引：

```bash
GET /*/_alias/rollover-alias
```

• 检查索引的生命周期策略状态：

```bash
GET /rollover-alias-*/_ilm/explain
```

### 注意事项

• **迁移已有数据**：如果当前索引名称不符合规则且无法修改，需要新建索引并迁移数据。
• **自动 Rollover**：确保策略中的条件（如 `max_size`、`max_docs`、`max_age`）设置合理。

通过以上步骤，你的索引会正确关联别名，并满足 Rollover 策略要求。
