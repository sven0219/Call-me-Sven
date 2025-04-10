---
title: "Step - by - Step Guide: Configuring Index Lifecycle Policy in Kibana"
date: 2025-04-10 13:00:42+08:00
draft: false
tags: ["es","lifecycle"]
Categories: ["devops"]
summary: "Add lifecycle to index"
showtoc: true
---
To configure the index lifecycle policy in Kibana, please follow these steps:

### Step 1: Confirm if the current index already has an alias

Use the following API to check the alias of the index:

```bash
GET /rollover-demo-k8s-docker/_alias
```

If there is no alias in the returned result, you need to add one for it.

### Step 2: Add an alias to the index

Add an alias to the index via the Elasticsearch API and set it as the write index:

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

Replace `rollover-alias` with the alias name you choose.

### Step 3: Ensure that the index name conforms to the Rollover naming rules

Rollover requires that the index name ends with a number (e.g., `myindex-000001`) to automatically generate subsequent indexes. If the current index does not conform to this format:

1. **Recreate the index**: Use the correct naming format (e.g., `rollover-alias-000001`).

2. **Create an index template** (recommended):

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

3. **Initialize the index**:

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

### Step 4: Apply the lifecycle policy

In Kibana, bind the policy `rollover-7days` to the alias `rollover-alias`, instead of directly binding it to the index name.

### Step 5: Verify the configuration

• Ensure that the alias points to the index correctly:

```bash
GET /*/_alias/rollover-alias
```

• Check the lifecycle policy status of the index:

```bash
GET /rollover-alias-*/_ilm/explain
```





### Notes

• **Migration of existing data**: If the current index name does not conform to the naming rules and cannot be modified, you need to create a new index and migrate the data.
• **Automatic Rollover**: Ensure that the Rollover conditions (such as `max_size`, `max_docs`, `max_age`) in the policy are set reasonably.

Through the above steps, your index will be correctly associated with the alias and meet the requirements of the Rollover policy.
