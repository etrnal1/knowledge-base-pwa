# 主题变化追踪系统 - 设计文档

## 1. 功能需求

### 1.1 核心功能
- [x] 创建主题（Topic）和子主题（SubTopic）
- [x] 支持树形结构（主题可嵌套子主题）
- [x] 对主题/子主题进行描述编辑
- [x] 自动追踪所有变化（创建/编辑/删除）
- [x] 展示变化历史和对比
- [x] 显示操作者信息
- [x] 支持撤销/重做功能

## 2. 数据模型设计

### 2.1 Topic（主题）
```typescript
interface Topic {
  id: string                    // UUID
  name: string                  // 主题名称
  description: string           // 主题描述
  children: string[]            // 子主题ID列表
  createdAt: number             // 创建时间戳
  updatedAt: number             // 最后更新时间戳
  createdBy: string             // 创建人
  parent?: string               // 上级主题ID（支持多级嵌套）
}
```

### 2.2 SubTopic（子主题）
```typescript
interface SubTopic {
  id: string                    // UUID
  parentId: string              // 父主题ID
  name: string                  // 子主题名称
  description: string           // 子主题描述
  tags?: string[]               // 标签
  createdAt: number             // 创建时间戳
  updatedAt: number             // 最后更新时间戳
  createdBy: string             // 创建人
}
```

### 2.3 Change（变化记录）
```typescript
interface Change {
  id: string                    // UUID
  targetId: string              // 操作的主题/子主题ID
  targetType: 'topic' | 'subtopic'
  timestamp: number             // 操作时间戳
  operator: string              // 操作者ID/名称
  type: 'create' | 'update' | 'delete'  // 操作类型
  
  // 针对update操作
  updates?: {
    [key: string]: {
      oldValue: any
      newValue: any
    }
  }
  
  // 快照：用于撤销/重做
  snapshot: {
    before?: any                // 修改前的完整对象
    after?: any                 // 修改后的完整对象
  }
}
```

### 2.4 OperationHistory（操作历史 - 用于撤销/重做）
```typescript
interface OperationHistory {
  id: string
  changes: Change[]              // 本次操作涉及的所有变化
  timestamp: number
  operator: string
  description: string            // 操作描述，用于撤销/重做时显示
}
```

## 3. 核心功能设计

### 3.1 变化追踪机制

#### 创建操作
```
用户创建主题 → 生成Change记录 → 保存到DB
  ├─ type: 'create'
  ├─ snapshot.after: 完整的主题对象
  └─ 添加到操作历史
```

#### 更新操作
```
用户编辑主题 → 对比修改前后 → 生成Change记录
  ├─ type: 'update'
  ├─ updates: { fieldName: { oldValue, newValue } }
  ├─ snapshot.before: 修改前对象
  ├─ snapshot.after: 修改后对象
  └─ 添加到操作历史
```

#### 删除操作
```
用户删除主题 → 记录完整信息 → 生成Change记录
  ├─ type: 'delete'
  ├─ snapshot.before: 删除前的完整对象
  └─ 逻辑删除（标记为已删除）
```

### 3.2 撤销/重做实现

#### 撤销栈结构
```typescript
interface UndoRedoManager {
  undoStack: OperationHistory[]    // 可撤销的操作
  redoStack: OperationHistory[]    // 可重做的操作
}
```

#### 撤销流程
```
点击撤销 → 弹出undo栈顶操作 → 使用snapshot.before恢复状态
       → 压入redo栈 → 更新UI
```

#### 重做流程
```
点击重做 → 弹出redo栈顶操作 → 使用snapshot.after恢复状态
       → 压入undo栈 → 更新UI
```

#### 限制
- Undo栈最多保留50条操作历史
- 新操作后redo栈自动清空
- 刷新页面后undo/redo栈清空（只保留变化历史）

### 3.3 变化展示设计

#### 变化列表视图
```
时间线展示：
  ├─ 2024-01-05 10:30 [用户A] 创建了主题"Python基础"
  ├─ 2024-01-05 10:35 [用户B] 修改了子主题"变量"
  │   └─ name: "Variables" → "变量定义"
  │   └─ description: "..." → "..."
  ├─ 2024-01-05 10:40 [用户A] 删除了子主题"函数进阶"
  └─ 2024-01-05 11:00 [用户B] 修改了主题"Python基础"
      └─ description: "..." → "..."
```

#### 对比视图
```
修改前                        修改后
─────────────────────────────────────
name: "Variables"    →        name: "变量定义"
description: "..."   →        description: "..."

高亮显示变化的字段
支持逐字对比（针对长文本）
```

#### 时间线视图
```
    2024-01-05
    ├─ 10:30 ● 创建 (用户A)
    ├─ 10:35 ● 修改 (用户B)
    ├─ 10:40 ● 删除 (用户A)
    └─ 11:00 ● 修改 (用户B)
    
    2024-01-04
    └─ ...
```

## 4. 存储策略

### 4.1 IndexedDB 结构
```
数据库：knowledge-base

ObjectStore:
├─ topics          // 主题表
├─ subtopics       // 子主题表
├─ changes         // 变化记录表
└─ operations      // 操作历史表（用于undo/redo）

索引：
├─ topics.id
├─ topics.parent
├─ subtopics.parentId
├─ changes.targetId
├─ changes.timestamp
└─ operations.timestamp
```

### 4.2 数据查询
- 获取主题树：按parent字段组织
- 获取变化历史：按timestamp降序排列
- 获取操作历史：仅保留当前会话的50条

## 5. UI/UX 设计

### 5.1 左侧面板 - 主题树
```
┌─────────────────────────┐
│ 主题管理                │
├─────────────────────────┤
│ [+] 新建主题            │
│                         │
│ ▼ Python基础            │
│   ├─ ▶ 变量定义         │
│   ├─ ▶ 函数             │
│   └─ ▶ 类和对象         │
│                         │
│ ▼ Web开发               │
│   ├─ ▶ HTML/CSS         │
│   ├─ ▶ JavaScript       │
│   └─ ▶ React            │
│                         │
└─────────────────────────┘
```
操作：
- 右键菜单：编辑、新建子主题、删除
- 拖拽排序：调整位置

### 5.2 中间面板 - 主题详情编辑
```
┌──────────────────────────────┐
│ Python基础                   │
├──────────────────────────────┤
│ 名称: [Python基础________]   │
│ 描述: [____________]         │
│       [___________________]  │
│       [___________________]  │
│                              │
│ [撤销] [重做] [保存] [取消]   │
└──────────────────────────────┘
```

### 5.3 右侧面板 - 变化历史
```
┌─────────────────────────────┐
│ 变化历史                    │
├─────────────────────────────┤
│ 📋 变化列表  📊 时间线       │
│                             │
│ 2024-01-05 10:35            │
│ ✏️ 修改 变量定义            │
│ [用户B]                     │
│ [展开对比▼]                 │
│                             │
│   ┌──────────────────────┐  │
│   │ 名称:                │  │
│   │ - Variables          │  │
│   │ + 变量定义           │  │
│   │                      │  │
│   │ 描述:                │  │
│   │ - ...                │  │
│   │ + ...                │  │
│   └──────────────────────┘  │
│                             │
│ 2024-01-05 10:30            │
│ ➕ 创建 Python基础          │
│ [用户A]                     │
│                             │
└─────────────────────────────┘
```

## 6. 交互流程

### 6.1 编辑流程
```
1. 用户点击选择主题/子主题
2. 中间面板加载详情和编辑器
3. 用户修改内容
4. 用户点击保存
5. 系统：
   - 对比修改前后
   - 生成Change记录
   - 保存到DB
   - 压入undo栈
   - redo栈清空
   - 刷新变化列表
6. 用户可看到新的变化记录
```

### 6.2 撤销流程
```
1. 用户点击撤销
2. 系统：
   - 检查undo栈是否为空
   - 弹出栈顶操作
   - 遍历operation中的所有change
   - 对每个change使用snapshot.before恢复
   - 压入redo栈
   - 更新UI
3. 用户看到恢复后的状态
```

### 6.3 变化展示流程
```
1. 用户点击变化记录中的"展开"
2. 对比视图显示修改前后
3. 对于长文本，支持word-level对比
4. 高亮显示变化部分
```

## 7. 用户信息管理

### 7.1 操作者识别
```typescript
interface User {
  id: string           // 本地唯一ID（可使用UUID）
  name: string         // 用户昵称（可设置）
  createdAt: number    // 用户创建时间
}
```

操作流程：
- 首次打开应用：生成UUID并保存到localStorage
- 用户可在设置中修改昵称
- 所有后续操作自动关联该用户信息

## 8. 导入/导出

### 8.1 导出格式
```json
{
  "version": "1.0",
  "exportAt": 1704336000,
  "topics": [...],
  "subtopics": [...],
  "changes": [...]
}
```

### 8.2 备份策略
- 支持手动导出为JSON文件
- 定期自动备份到localStorage（仅保留最新5份）

## 9. 性能优化

- 虚拟滚动：变化列表较多时支持虚拟滚动
- 分页加载：历史列表支持分页
- 索引优化：frequent queries建立索引
- 定期清理：自动清理90天前的变化记录（可配置）

## 10. 后续扩展

- [ ] 支持协作编辑（多用户同步）
- [ ] 支持分支管理（类似git branch）
- [ ] 支持标签和搜索
- [ ] 支持评论和讨论
- [ ] 支持变化的筛选（按操作者、日期、类型）
