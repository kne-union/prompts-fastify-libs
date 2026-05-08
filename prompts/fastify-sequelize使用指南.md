# Fastify-Sequelize 使用指南

## 快速开始

### 基本注册

```javascript
const fastify = require('fastify')();
const fastifySequelize = require('./index');

// 基本配置
fastify.register(fastifySequelize, {
  db: {
    dialect: 'sqlite',
    storage: './database.sqlite'
  },
  modelsPath: './models',
  prefix: 't_'
});

// 使用模型
fastify.post('/users', async (request, reply) => {
  const { user } = fastify.models;
  const result = await user.create({
    username: request.body.username,
    email: request.body.email
  });
  return result;
});

fastify.listen({ port: 3000 });
```

## 配置详解

### 完整配置选项

```javascript
const config = {
  // 数据库配置
  db: {
    dialect: 'sqlite',        // 数据库类型: sqlite/mysql/postgres/mssql
    database: 'myapp',        // 数据库名
    username: 'root',         // 用户名
    password: 'password',     // 密码
    host: 'localhost',        // 主机
    port: 3306,              // 端口
    storage: './db.sqlite'   // SQLite文件路径
  },
  
  // 雪花ID配置
  snowflake: {
    instance_id: 1,                    // 实例ID (0-1023)
    custom_epoch: new Date('2024-01-01').getTime()  // 基准时间
  },
  
  // 模型配置
  modelsPath: './models',    // 模型目录
  sqlPath: './sql',          // SQL脚本目录
  prefix: 't_',              // 表前缀
  modelPrefix: 'Base',       // 模型前缀
  name: 'models',            // Fastify实例属性名
  
  // Glob配置
  glob: {
    ignore: 'node_modules/**', // 忽略模式
    pattern: '**/*.js'         // 文件匹配模式
  },
  
  // 同步选项
  syncOptions: {
    force: false,    // 是否删除并重新创建表
    alter: true,     // 是否修改现有表结构
    logging: console.log
  }
};
```

## 模型定义

### 基础模型

```javascript
// models/user.js
module.exports = ({ DataTypes, definePrimaryType, fastify, options }) => ({
  name: 'User',
  model: {
    // 主键会自动添加（雪花ID）
    username: {
      type: DataTypes.STRING(50),
      allowNull: false,
      unique: true,
      validate: {
        notEmpty: true
      }
    },
    email: {
      type: DataTypes.STRING(100),
      allowNull: false,
      unique: true,
      validate: {
        isEmail: true
      }
    },
    status: {
      type: DataTypes.TINYINT,
      defaultValue: 1,
      comment: '状态：1-正常，0-禁用'
    }
  },
  options: {
    tableName: 'users',     // 可自定义表名
    paranoid: true,         // 软删除
    timestamps: true,       // 自动时间戳
    underscored: true       // 下划线命名
  },
  associate(db) {
    // 关联关系
    db.user.hasMany(db.post, {
      foreignKey: 'userId',
      as: 'posts'
    });
  }
});
```

### 带关联关系的模型

```javascript
// models/post.js
module.exports = ({ DataTypes }) => ({
  name: 'Post',
  model: {
    title: {
      type: DataTypes.STRING(200),
      allowNull: false
    },
    content: {
      type: DataTypes.TEXT,
      allowNull: false
    },
    userId: {
      type: DataTypes.BIGINT,
      allowNull: false,
      comment: '作者ID'
    }
  },
  associate(db) {
    db.post.belongsTo(db.user, {
      foreignKey: 'userId',
      as: 'author'
    });
    db.post.hasMany(db.comment, {
      foreignKey: 'postId',
      as: 'comments'
    });
  }
});

// models/comment.js
module.exports = ({ DataTypes }) => ({
  name: 'Comment',
  model: {
    content: {
      type: DataTypes.TEXT,
      allowNull: false
    },
    postId: {
      type: DataTypes.BIGINT,
      allowNull: false
    },
    userId: {
      type: DataTypes.BIGINT,
      allowNull: false
    }
  },
  associate(db) {
    db.comment.belongsTo(db.post, {
      foreignKey: 'postId',
      as: 'post'
    });
    db.comment.belongsTo(db.user, {
      foreignKey: 'userId',
      as: 'author'
    });
  }
});
```

## 动态模型管理

### 运行时添加模型

```javascript
// 添加单个模型文件
await fastify.sequelize.addModels('./models/user.js');

// 添加整个目录
await fastify.sequelize.addModels('./models');

// 添加函数式模型
const dynamicModel = ({ DataTypes }) => ({
  name: 'DynamicModel',
  model: {
    name: DataTypes.STRING
  }
});
await fastify.sequelize.addModels(dynamicModel);

// 带选项的添加
await fastify.sequelize.addModels('./models', {
  prefix: 'custom_',
  modelPrefix: 'Custom'
});
```

## ID生成器

### 使用雪花ID

```javascript
// 生成唯一ID
const id = fastify.sequelize.generateId();
console.log(id); // 例如: 1381234567890123456

// 在模型中使用（自动）
const user = await User.create({
  id: fastify.sequelize.generateId(), // 可选，会自动生成
  username: 'john'
});
```

## 数据库同步

### 同步所有模型

```javascript
// 自动同步（包含SQL脚本执行）
await fastify.sequelize.sync();

// 带选项同步
await fastify.sequelize.sync({
  force: false,    // 不删除表
  alter: true      // 修改表结构
});

// 等待同步完成
await fastify.sequelize.syncPromise;
```

### SQL脚本执行

创建 `sql/init.sql`:
```sql
CREATE INDEX IF NOT EXISTS idx_users_email ON t_user(email);
CREATE INDEX IF NOT EXISTS idx_posts_user_id ON t_post(user_id);
```

插件会在同步时自动执行 `sql/` 目录下的所有 `.sql` 文件。

## CRUD操作示例

### 查询操作

```javascript
// 查找所有用户
const users = await fastify.models.user.findAll();

// 条件查询
const activeUsers = await fastify.models.user.findAll({
  where: { status: 1 },
  include: ['posts']  // 包含关联数据
});

// 分页查询
const page = 1;
const pageSize = 10;
const paginatedUsers = await fastify.models.user.findAndCountAll({
  where: { status: 1 },
  limit: pageSize,
  offset: (page - 1) * pageSize,
  order: [['created_at', 'DESC']]
});

// 单条查询
const user = await fastify.models.user.findByPk(userId);
const userByUsername = await fastify.models.user.findOne({
  where: { username: 'john' }
});
```

### 创建操作

```javascript
// 创建单个记录
const user = await fastify.models.user.create({
  username: 'john_doe',
  email: 'john@example.com'
});

// 批量创建
const users = await fastify.models.user.bulkCreate([
  { username: 'user1', email: 'user1@example.com' },
  { username: 'user2', email: 'user2@example.com' }
]);
```

### 更新操作

```javascript
// 更新单条记录
await fastify.models.user.update(
  { status: 0 },
  { where: { id: userId } }
);

// 更新并获取结果
const updatedUser = await fastify.models.user.findByPk(userId);
await updatedUser.update({ status: 0 });
```

### 删除操作

```javascript
// 软删除（需要 paranoid: true）
await fastify.models.user.destroy({
  where: { id: userId }
});

// 硬删除
await fastify.models.user.destroy({
  where: { id: userId },
  force: true
});

// 恢复软删除的记录
await fastify.models.user.restore({
  where: { id: userId }
});
```

## 事务处理

```javascript
const result = await fastify.sequelize.instance.transaction(async (t) => {
  const user = await fastify.models.user.create({
    username: 'john',
    email: 'john@example.com'
  }, { transaction: t });

  await fastify.models.profile.create({
    userId: user.id,
    bio: 'Hello World'
  }, { transaction: t });

  return user;
});
```

## 高级特性

### 模型别名

启用 `modelPrefix` 后，可以自动创建别名：

```javascript
// 配置
const config = {
  modelPrefix: 'Base',
  modelsPath: './models'
};

// 如果有模型 BaseUser，会自动创建别名 User
const user = fastify.models.user; // 等同于 fastify.models.BaseUser
```

### 多数据库支持

```javascript
// 注册多个实例
fastify.register(fastifySequelize, {
  name: 'primaryDb',
  db: { dialect: 'mysql', ... },
  modelsPath: './models/primary'
});

fastify.register(fastifySequelize, {
  name: 'analyticsDb',
  db: { dialect: 'postgres', ... },
  modelsPath: './models/analytics'
});

// 使用
const user = await fastify.primaryDb.user.findByPk(1);
const stats = await fastify.analyticsDb.event.count();
```

## 最佳实践

### 1. 环境配置分离

```javascript
// config/dev.js
module.exports = {
  db: { dialect: 'sqlite', storage: './dev.sqlite' },
  syncOptions: { alter: true, force: false }
};

// config/prod.js  
module.exports = {
  db: {
    dialect: 'mysql',
    host: process.env.DB_HOST,
    database: process.env.DB_NAME,
    username: process.env.DB_USER,
    password: process.env.DB_PASS
  },
  syncOptions: { alter: false, force: false }
};

// app.js
const config = require('./config/' + process.env.NODE_ENV);
fastify.register(fastifySequelize, config);
```

### 2. 错误处理

```javascript
fastify.setErrorHandler(async (error, request, reply) => {
  if (error.name === 'SequelizeValidationError') {
    return reply.status(400).send({
      error: 'Validation Error',
      details: error.errors.map(e => e.message)
    });
  }

  if (error.name === 'SequelizeUniqueConstraintError') {
    return reply.status(409).send({
      error: 'Duplicate Entry',
      message: 'Record already exists'
    });
  }

  reply.send(error);
});
```

### 3. 性能优化

```javascript
// 使用include优化关联查询
const users = await fastify.models.user.findAll({
  include: [
    {
      model: fastify.models.post,
      as: 'posts',
      limit: 5,
      separate: true  // 单独查询避免笛卡尔积
    }
  ]
});

// 批量操作
const users = await fastify.models.user.findAll({
  where: {
    status: 1
  },
  attributes: ['id', 'username'],  // 只选择需要的字段
  raw: true  // 返回纯JSON
});
```

## 注意事项

1. **雪花ID**: 确保 `instance_id` 在分布式环境中唯一
2. **软删除**: 使用 `paranoid: true` 时，destroy操作是软删除
3. **表前缀**: 自动添加前缀，实际表名为 `{prefix}{tableName}`
4. **同步策略**: 生产环境慎用 `force: true`
5. **关联加载**: 使用 `include` 时注意N+1查询问题
6. **事务**: 复杂业务操作使用事务保证数据一致性

## 模型定义规范（关键约定）

### 1. 不显式定义 `id` 字段

框架会自动生成雪花ID主键，**不要**在 `model` 中手动定义 `id`：

```javascript
// ❌ 错误 - 不要显式定义 id
module.exports = ({DataTypes}) => ({
  model: {
    id: {
      type: DataTypes.STRING,
      defaultValue: DataTypes.UUIDV4,  // 错误！
      primaryKey: true
    },
    name: { type: DataTypes.STRING }
  }
});

// ✅ 正确 - 框架自动生成
module.exports = ({DataTypes, options}) => ({
  model: {
    name: { type: DataTypes.STRING }
  }
});
```

### 2. 不显式定义外键字段

外键字段通过 `belongsTo` / `hasMany` 关联自动生成，**不要**在 `model` 中手动定义 `conversationId`、`userId`、`senderId` 等外键：

```javascript
// ❌ 错误 - 不要显式定义外键
module.exports = ({DataTypes}) => ({
  model: {
    conversationId: {  // 错误！会通过 belongsTo 自动生成
      type: DataTypes.STRING,
      allowNull: false
    },
    userId: {  // 错误！
      type: DataTypes.STRING
    },
    content: { type: DataTypes.JSONB }
  },
  associate: ({message, conversation}) => {
    message.belongsTo(conversation, {foreignKey: 'conversationId', as: 'conversation'});
  }
});

// ✅ 正确 - 外键通过关联自动生成
module.exports = ({DataTypes, options}) => ({
  model: {
    content: { type: DataTypes.JSONB }
  },
  associate: ({message, conversation}) => {
    message.belongsTo(conversation, {foreignKey: 'conversationId', as: 'conversation'});
  }
});
```

### 3. 函数签名必须包含 `options` 参数

模型导出函数签名应包含 `options`，用于获取外部依赖和配置：

```javascript
// ❌ 错误 - 缺少 options
module.exports = ({DataTypes}) => ({ ... });

// ✅ 正确
module.exports = ({DataTypes, options}) => ({ ... });
```

### 4. `associate` 使用 camelCase 解构

`associate` 函数的参数是所有模型的 camelCase 名称映射，解构时使用与文件名对应的 camelCase：

```javascript
// 模型文件: conversation.js, conversation-member.js, message.js
// 对应名称: conversation, conversationMember, message

module.exports = ({DataTypes, options}) => ({
  model: { ... },
  associate: ({conversation, conversationMember, message}) => {
    // 使用解构出的 camelCase 名称
    conversation.hasMany(conversationMember, {foreignKey: 'conversationId', as: 'members'});
    conversation.hasMany(message, {foreignKey: 'conversationId', as: 'messages'});
  }
});
```

**命名规则**：文件名 `conversation-member.js` → 模型名 `conversationMember`（小写开头 camelCase）

### 5. 引用外部 User 模型使用 `options.getUserModel()`

当模型需要关联外部 User 表时，不要硬编码引用，使用 `options.getUserModel()` 注入：

```javascript
module.exports = ({DataTypes, options}) => ({
  model: {
    // 不定义 userId 字段，由 belongsTo 自动生成
  },
  associate: ({message}) => {
    // 通过 options.getUserModel() 获取 User 模型
    message.belongsTo(options.getUserModel(), {
      foreignKey: 'senderId',
      as: 'sender'
    });
  }
});
```

宿主应用注册插件时提供 `getUserModel`：

```javascript
fastify.register(require('@kne/fastify-im'), {
  getUserModel: () => fastify.account.models.user
});
```

### 6. 每个模型添加 `options` JSONB 扩展字段

为每个模型预留 `options: DataTypes.JSONB` 字段，方便未来扩展而不改表结构：

```javascript
module.exports = ({DataTypes, options}) => ({
  model: {
    name: { type: DataTypes.STRING },
    // ... 业务字段 ...
    options: {
      type: DataTypes.JSONB,
      comment: '扩展字段'
    }
  }
});
```

### 7. 唯一索引需加软删除过滤

使用 `paranoid: true` 的模型，唯一索引必须加 `where: { deleted_at: null }` 过滤，否则软删除后无法重建同名记录：

```javascript
module.exports = ({DataTypes, options}) => ({
  model: { ... },
  options: {
    paranoid: true,
    indexes: [
      // ❌ 错误 - 不加过滤，软删除后同名记录无法创建
      { fields: ['conversationId', 'userId'], unique: true },
      
      // ✅ 正确 - 加软删除过滤
      { fields: ['conversationId', 'userId'], unique: true, where: { deleted_at: null } }
    ]
  }
});
```

### 8. 完整模型示例

```javascript
// models/conversation-member.js
module.exports = ({DataTypes, options}) => {
  return {
    model: {
      role: {
        type: DataTypes.STRING,
        defaultValue: 'member',
        comment: '群内角色: owner/admin/member'
      },
      lastReadMessageId: {
        type: DataTypes.STRING,
        comment: '最后已读消息ID'
      },
      leftAt: {
        type: DataTypes.DATE,
        comment: '退出时间(null表示仍在群内)'
      },
      options: {
        type: DataTypes.JSONB,
        comment: '扩展字段'
      }
    },
    associate: ({conversation, conversationMember}) => {
      conversationMember.belongsTo(options.getUserModel(), {
        foreignKey: 'userId',
        as: 'user'
      });
      conversationMember.belongsTo(conversation, {
        foreignKey: 'conversationId',
        as: 'conversation'
      });
    },
    options: {
      comment: 'IM会话成员表',
      indexes: [
        {fields: ['conversationId']},
        {fields: ['userId']},
        {fields: ['conversationId', 'userId'], unique: true, where: {deleted_at: null}}
      ]
    }
  };
};
```

### 9. Service 层访问模型的命名约定

Service 层访问模型统一使用 **camelCase（小写开头）**，与 `associate` 解构名一致：

```javascript
// ❌ 错误 - PascalCase
const conv = await models.Conversation.findByPk(id);
const members = await models.ConversationMember.findAll();

// ✅ 正确 - camelCase
const conv = await models.conversation.findByPk(id);
const members = await models.conversationMember.findAll();
```

**映射关系**：

| 文件名 | 模型访问名 |
|--------|-----------|
| `conversation.js` | `models.conversation` |
| `conversation-member.js` | `models.conversationMember` |
| `system-message.js` | `models.systemMessage` |

在 `include` 中也使用 camelCase：

```javascript
const result = await models.conversation.findAll({
  include: [
    {model: models.conversationMember, as: 'members'},
    {model: models.message, as: 'messages'}
  ]
});
```
