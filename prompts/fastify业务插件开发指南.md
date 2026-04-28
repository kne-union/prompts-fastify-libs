# Fastify 业务插件开发指南

## 概述

本指南旨在帮助开发者构建功能完整、可扩展、易维护的 Fastify 业务插件包。通过合理的架构设计和模块划分,实现高内聚低耦合的插件系统。

## 核心架构设计

### 1. 插件化架构

使用 `fastify-plugin` 包装插件,确保作用域正确:

```javascript
const fp = require('fastify-plugin');

module.exports = fp(async (fastify, options) => {
  // 插件逻辑
}, {
  name: 'your-plugin-name',
  dependencies: ['dependency-plugin'] // 声明依赖
});
```

**设计原则**:
- 每个插件独立封装,单一职责
- 明确声明依赖关系,确保加载顺序
- 通过 options 参数实现可配置性

### 2. 分层架构设计

采用经典的三层架构,清晰分离关注点:

```
controllers/  - 路由控制层
services/     - 业务逻辑层
models/       - 数据模型层
schema/       - 数据验证
utils/        - 工具函数
```

**各层职责**:
- **Controllers**: 处理 HTTP 请求/响应,参数校验,调用 Service
- **Services**: 核心业务逻辑,事务处理,数据计算
- **Models**: 数据结构定义,数据库映射
- **Schema**: 请求数据验证规则
- **Utils**: 通用工具函数,可复用逻辑

### 3. 命名空间组织

使用命名空间模式组织模块,避免全局污染:

```javascript
fastify.register(require('@kne/fastify-namespace'), {
  name: 'yourModule',
  modules: [
    ['controllers', './controllers'],  // 目录自动加载
    ['models', modelInstances],        // 模型实例
    ['services', './services'],        // 服务目录
    ['utils', { helper: () => {} }],   // 工具对象
    ['authenticate', {                 // 认证中间件
      custom: async (request) => {}
    }]
  ]
});
```

## 配置管理

### 1. 默认配置模式

提供合理的默认值,同时允许覆盖:

```javascript
module.exports = fp(async (fastify, options) => {
  options = Object.assign({}, {
    // 数据库配置
    dbTableNamePrefix: 't_',

    // 插件命名
    name: 'defaultName',
    prefix: '/api/default',

    // 业务配置
    maxItems: 100,
    timeout: 30000,

    // 扩展点
    getUserModel: () => {
      if (!fastify.account) {
        throw new Error('请安装依赖插件或提供自定义实现');
      }
      return fastify.account.models.user;
    },

    // 外部配置文件
    configPath: path.resolve(process.cwd(), './config.js')
  }, options);
});
```

**设计要点**:
- 所有配置项有明确默认值
- 提供扩展点函数,允许注入外部实现
- 配置文件支持多种格式 (.js, .json, .yml)
- 配置校验和错误提示

### 2. 配置合并策略

支持外部配置与内部配置的深度合并:

```javascript
const merge = require('lodash/merge');

const loadExternalConfig = async (filePath) => {
  const ext = path.extname(filePath);

  if (ext === '.js') {
    return require(filePath);
  }
  if (ext === '.json') {
    return JSON.parse(await fs.readFile(filePath, 'utf8'));
  }
  if (ext === '.yml') {
    return yml.load(await fs.readFile(filePath, 'utf8'));
  }

  return {};
};

const finalConfig = merge({}, defaultConfig, await loadExternalConfig(configPath));
```

## 认证与授权

### 1. 多层认证中间件

组合多个认证步骤,构建完整的认证流程:

```javascript
fastify.register(namespace, {
  name: 'module',
  modules: [
    ['authenticate', {
      // 自定义用户认证
      customUser: async (request) => {
        const { services } = fastify[options.name];

        // 获取用户详细信息
        request.userDetail = await services.user.getDetail(
          request.userInfo.id
        );

        // 验证用户状态
        if (request.userDetail.status !== 'active') {
          throw new Forbidden('用户账号已被禁用');
        }
      },

      // 管理员认证
      admin: async (request) => {
        if (!request.userInfo.isAdmin) {
          throw new Forbidden('需要管理员权限');
        }
      }
    }]
  ]
});
```

### 2. 路由认证配置

在路由中组合多个认证中间件:

```javascript
fastify.post('/api/resource', {
  onRequest: [
    userAuthenticate,           // 用户认证
    authenticate.customUser,    // 自定义用户认证
    authenticate.admin          // 管理员认证
  ],
  schema: {
    summary: '操作描述',
    body: { /* 请求体验证 */ }
  }
}, async (request) => {
  // 已通过所有认证,可直接访问
  return await services.resource.create(request.body);
});
```

### 3. 用户上下文传递

在认证中间件中构建完整的用户上下文:

```javascript
const authenticate = async (request) => {
  const { services } = fastify[options.name];

  // 基础用户信息 (来自上游插件)
  const { userInfo } = request;

  // 扩展上下文
  request.userDetail = await services.user.getDetail(userInfo.id);
};
```

## Service 层设计

### 1. 服务模块化

每个 Service 模块独立封装,通过 `Object.assign` 挂载:

```javascript
const fp = require('fastify-plugin');
const omit = require('lodash/omit');

module.exports = fp(async (fastify, options) => {
  const { models } = fastify[options.name];
  const { Op } = fastify.sequelize.Sequelize;

  // 创建 - 第一个参数为 authenticatePayload,自动注入 tenantId
  const create = async (authenticatePayload, data) => {
    const { tenantId } = authenticatePayload;
    return await models.resource.create(Object.assign({}, data, { tenantId }));
  };

  // 详情 - 校验 tenantId 确保数据隔离
  const detail = async (authenticatePayload, { id }) => {
    const { tenantId } = authenticatePayload;
    const record = await models.resource.findByPk(id);
    if (!record || record.tenantId !== tenantId) {
      throw fastify.httpErrors.notFound(fastify.intl('resourceNotFound'));
    }
    return record;
  };

  // 修改 - 使用 save 命名,用 omit 过滤不可修改字段
  const save = async (authenticatePayload, { id, ...data }) => {
    const record = await detail(authenticatePayload, { id });
    await record.update(omit(data, ['tenantId', 'createdAt', 'updatedAt']));
    return record;
  };

  // 删除 - 无返回值
  const remove = async (authenticatePayload, { id }) => {
    const record = await detail(authenticatePayload, { id });
    await record.destroy();
  };

  // 列表 - filter 对象包裹筛选条件,使用 perPage 分页
  const list = async (authenticatePayload, { filter = {}, perPage = 20, currentPage = 1 }) => {
    const { tenantId } = authenticatePayload;
    const where = { tenantId };

    // 从 filter 中提取筛选条件
    if (filter.status) {
      where.status = filter.status;
    }
    if (filter.keyword) {
      where[Op.or] = [
        { name: { [Op.iLike]: `%${filter.keyword}%` } }
      ];
    }

    const { rows, count } = await models.resource.findAndCountAll({
      where,
      offset: perPage * (currentPage - 1),
      limit: perPage,
      order: [['createdAt', 'DESC']]
    });

    return {
      pageData: rows,
      totalCount: count
    };
  };

  // 挂载到 fastify 命名空间
  Object.assign(fastify[options.name].services, {
    resource: {
      create,
      detail,
      save,
      remove,
      list
    }
  });
});
```

**Service 方法签名规范**:
- 第一个参数始终为 `authenticatePayload`(包含 `tenantId` 等认证信息)
- 第二个参数为业务数据(来自 `request.query` 或 `request.body`)
- 修改方法使用 `save` 命名(而非 `update`),body 中 `id` 与其他字段同级,service 内用 `{ id, ...data }` 解构
- 使用 `lodash/omit` 过滤 `tenantId`、`createdAt`、`updatedAt` 等不可修改字段
- `detail` 方法内校验 `record.tenantId === tenantId`,其他方法(如 `save`、`remove`)复用 `detail` 做鉴权

### 2. 业务逻辑封装

Service 层负责核心业务逻辑:

```javascript
const create = async ({ name, email, category }) => {
  // 1. 数据验证
  if (!email && !phone) {
    throw new Error('邮箱或手机号必填');
  }
  
  // 2. 业务规则检查
  const existingCount = await models.user.count({ 
    where: { email } 
  });
  
  if (existingCount > 0) {
    throw new Error('邮箱已存在');
  }
  
  // 3. 关联数据验证
  const currentCount = await models.user.count();
  
  if (currentCount >= MAX_USERS) {
    throw new Error('用户数量已达上限');
  }
  
  // 4. 数据转换
  const validatedCategory = await services.category.validate(category);
  
  // 5. 创建记录
  return await models.user.create({
    name,
    email,
    categoryId: validatedCategory.id
  });
};
```

### 3. 服务间协作

Service 可以相互调用,避免代码重复:

```javascript
const detail = async ({ id }) => {
  // 复用其他 Service 的方法
  const user = await models.user.findByPk(id);
  
  if (!user) {
    throw new Error('用户不存在');
  }
  
  // 组装返回数据
  user.setDataValue('categoryDetail', 
    await services.category.getDetail(user.categoryId)
  );
  
  return user;
};
```

## Controller 层设计

### 1. 路由规则规范

#### HTTP 方法约定

| 操作 | HTTP 方法 | 参数来源 | 说明 |
|------|-----------|----------|------|
| 列表查询 | `GET` | `request.query` | 读操作,参数通过 query string 传递 |
| 详情查询 | `GET` | `request.query` | 读操作,参数通过 query string 传递 |
| 创建资源 | `POST` | `request.body` | 写操作,参数通过 request body 传递 |
| 修改资源 | `POST` | `request.body` | 写操作,使用 `save` 而非 `update` |
| 删除资源 | `POST` | `request.body` | 写操作,参数通过 request body 传递 |

**核心原则**: 读操作用 `GET`,写操作用 `POST`。

#### URL 命名约定

所有路由统一使用以下 5 个路径,以 `${options.prefix}/资源名/操作` 格式:

```
GET  ${options.prefix}/resource/list    - 列表查询
GET  ${options.prefix}/resource/detail  - 详情查询
POST ${options.prefix}/resource/create  - 创建资源
POST ${options.prefix}/resource/save    - 修改资源(注意:使用 save 而非 update)
POST ${options.prefix}/resource/remove  - 删除资源
```

#### 列表查询参数约定

列表接口统一使用 `GET` + `query`,参数结构如下:

```javascript
query: {
  type: 'object',
  properties: {
    filter: {
      type: 'object',
      default: {}
    },
    perPage: {
      type: 'number',
      default: 20
    },
    currentPage: {
      type: 'number',
      default: 1
    }
  }
}
```

- **filter**: 筛选条件对象,所有业务筛选字段放在 filter 内(如 `filter.type`、`filter.status`、`filter.keyword`)
- **perPage**: 每页条数,默认 20(注意:使用 `perPage` 而非 `pageSize`)
- **currentPage**: 当前页码,默认 1

#### 详情查询参数约定

```javascript
query: {
  type: 'object',
  required: ['id'],
  properties: {
    id: { type: 'string' }
  }
}
```

#### Service 调用约定

所有 Service 方法**第一个参数为 `authenticatePayload`**(即 `request.tenantUserInfo`),第二个参数为业务数据:

```javascript
// GET 请求
async (request) => {
  return services.resource.list(request.tenantUserInfo, request.query);
}

// POST 请求 - 创建
async (request) => {
  return services.resource.create(request.tenantUserInfo, request.body);
}

// POST 请求 - 修改(save)
async (request) => {
  await services.resource.save(request.tenantUserInfo, request.body);
  return {};
}

// POST 请求 - 删除
async (request) => {
  await services.resource.remove(request.tenantUserInfo, request.body);
  return {};
}
```

**返回值约定**:
- `create`: 返回创建的记录
- `save` / `remove`: 返回 `{}`(空对象)
- `list` / `detail`: 返回查询结果

#### 认证中间件约定

租户业务路由统一使用 `authenticate.user` + `tenantAuthenticate.tenantUser`:

```javascript
const { authenticate } = fastify.account;
const { authenticate: tenantAuthenticate } = fastify.tenant;

// 路由认证
{
  onRequest: [authenticate.user, tenantAuthenticate.tenantUser]
}
```

#### 完整路由示例

```javascript
const fp = require('fastify-plugin');

module.exports = fp(async (fastify, options) => {
  const { services } = fastify[options.name];
  const { authenticate } = fastify.account;
  const { authenticate: tenantAuthenticate } = fastify.tenant;

  // 列表查询
  fastify.get(
    `${options.prefix}/resource/list`,
    {
      onRequest: [authenticate.user, tenantAuthenticate.tenantUser],
      schema: {
        summary: '资源列表',
        query: {
          type: 'object',
          properties: {
            filter: { type: 'object', default: {} },
            perPage: { type: 'number', default: 20 },
            currentPage: { type: 'number', default: 1 }
          }
        }
      }
    },
    async (request) => {
      return services.resource.list(request.tenantUserInfo, request.query);
    }
  );

  // 详情查询
  fastify.get(
    `${options.prefix}/resource/detail`,
    {
      onRequest: [authenticate.user, tenantAuthenticate.tenantUser],
      schema: {
        summary: '资源详情',
        query: {
          type: 'object',
          required: ['id'],
          properties: {
            id: { type: 'string' }
          }
        }
      }
    },
    async (request) => {
      return services.resource.detail(request.tenantUserInfo, request.query);
    }
  );

  // 创建资源
  fastify.post(
    `${options.prefix}/resource/create`,
    {
      onRequest: [authenticate.user, tenantAuthenticate.tenantUser],
      schema: {
        summary: '创建资源',
        body: {
          type: 'object',
          properties: {
            name: { type: 'string' },
            description: { type: 'string' }
          },
          required: ['name']
        }
      }
    },
    async (request) => {
      return services.resource.create(request.tenantUserInfo, request.body);
    }
  );

  // 修改资源
  fastify.post(
    `${options.prefix}/resource/save`,
    {
      onRequest: [authenticate.user, tenantAuthenticate.tenantUser],
      schema: {
        summary: '修改资源',
        body: {
          type: 'object',
          properties: {
            id: { type: 'string' },
            name: { type: 'string' },
            description: { type: 'string' }
          },
          required: ['id']
        }
      }
    },
    async (request) => {
      await services.resource.save(request.tenantUserInfo, request.body);
      return {};
    }
  );

  // 删除资源
  fastify.post(
    `${options.prefix}/resource/remove`,
    {
      onRequest: [authenticate.user, tenantAuthenticate.tenantUser],
      schema: {
        summary: '删除资源',
        body: {
          type: 'object',
          required: ['id'],
          properties: {
            id: { type: 'string' }
          }
        }
      }
    },
    async (request) => {
      await services.resource.remove(request.tenantUserInfo, request.body);
      return {};
    }
  );
});
```

### 2. 参数注入模式

Service 方法第一个参数统一为 `authenticatePayload`(租户用户信息),由 Controller 层注入:

```javascript
// Controller 层注入 request.tenantUserInfo
async (request) => {
  return services.resource.list(request.tenantUserInfo, request.query);
}
```

Service 层从 `authenticatePayload` 获取租户 ID,确保数据隔离:

```javascript
const list = async (authenticatePayload, { filter = {}, perPage = 20, currentPage = 1 }) => {
  const { tenantId } = authenticatePayload;
  // 使用 tenantId 过滤,确保租户数据隔离
  const where = { tenantId, ...filter };
  // ...
};
```

### 3. Schema 复用

提取公共 Schema,避免重复定义:

```javascript
// schema/resource.js
module.exports = {
  type: 'object',
  properties: {
    name: { type: 'string' },
    description: { type: 'string' },
    status: { type: 'string', enum: ['open', 'closed'] }
  },
  required: ['name']
};

// controllers/resource.js
const schema = require('../schema/resource');

fastify.post('/create', {
  schema: {
    summary: '创建资源',
    body: merge({}, schema)  // 复用并扩展
  }
}, handler);
```

## 数据模型设计

### 1. 关联关系定义

明确模型间的关联关系:

```javascript
module.exports = ({ DataTypes, options }) => ({
  model: {
    name: { type: DataTypes.STRING },
    description: { type: DataTypes.TEXT }
  },
  
  associate: ({ resource, user, category }) => {
    // 属于某个用户
    resource.belongsTo(user, {
      foreignKey: 'createdBy'
    });
    
    // 属于某个分类
    resource.belongsTo(category, {
      foreignKey: 'categoryId'
    });
  },
  
  options: {
    paranoid: true,  // 软删除
    timestamps: true,
    indexes: [
      {
        fields: ['name'],
        unique: true,
        where: { deleted_at: null }
      }
    ]
  }
});
```

### 2. 扩展字段设计

使用 JSONB 存储扩展数据,保持灵活性:

```javascript
model: {
  // 标准字段
  name: { type: DataTypes.STRING },
  status: { type: DataTypes.ENUM('open', 'closed') },
  
  // 扩展字段
  options: {
    type: DataTypes.JSONB,
    comment: '扩展配置',
    defaultValue: {}
  },
  
  // 元数据
  metadata: {
    type: DataTypes.JSONB,
    comment: '元数据'
  }
}
```

## 工具函数设计

### 1. 通用工具提取

将可复用的逻辑提取为工具函数:

```javascript
// utils/format.js
const formatPageData = (data, formatter) => {
  return data.map(item => ({
    ...item.toJSON(),
    ...formatter(item)
  }));
};

// utils/validate.js
const validateEmail = (email) => {
  const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return regex.test(email);
};

// utils/transform.js
const groupByField = (data, field) => {
  return data.reduce((acc, item) => {
    const key = item[field];
    if (!acc[key]) acc[key] = [];
    acc[key].push(item);
    return acc;
  }, {});
};
```

### 2. 工具函数挂载

通过命名空间提供全局访问:

```javascript
fastify.register(namespace, {
  modules: [
    ['utils', {
      formatPageData: require('./utils/format'),
      validateEmail: require('./utils/validate'),
      groupByField: require('./utils/group')
    }]
  ]
});

// 使用
const { utils } = fastify[options.name];
const formatted = utils.formatPageData(data, item => ({
  displayName: `${item.firstName} ${item.lastName}`
}));
```

## 错误处理

### 1. 业务错误定义

使用语义化的错误类型:

```javascript
const { BadRequest, Forbidden, NotFound } = require('http-errors');

const create = async (data) => {
  const user = await models.user.findByPk(data.userId);
  
  if (!user) {
    throw new NotFound('用户不存在');
  }
  
  if (user.status === 'inactive') {
    throw new Forbidden('用户已被禁用');
  }
  
  if (!data.name) {
    throw new BadRequest('名称不能为空');
  }
  
  // ... 创建逻辑
};
```

### 2. 错误处理中间件

统一错误响应格式:

```javascript
fastify.setErrorHandler(async (error, request, reply) => {
  // 验证错误
  if (error.validation) {
    return reply.status(400).send({
      error: 'ValidationError',
      message: error.message,
      details: error.validation
    });
  }
  
  // 业务错误
  if (error instanceof BadRequest) {
    return reply.status(400).send({
      error: 'BusinessError',
      message: error.message
    });
  }
  
  // 数据库错误
  if (error.name === 'SequelizeUniqueConstraintError') {
    return reply.status(409).send({
      error: 'DuplicateError',
      message: '数据已存在'
    });
  }
  
  // 未知错误
  console.error(error);
  return reply.status(500).send({
    error: 'InternalError',
    message: '服务器内部错误'
  });
});
```

## 最佳实践

### 1. 依赖注入

通过配置函数注入外部依赖,而非硬编码:

```javascript
// 默认实现
options = {
  getUserModel: () => fastify.account.models.user,
  getMessageService: () => fastify.message.services
};

// 在代码中使用
const User = options.getUserModel();
const { sendMessage } = options.getMessageService();
```

### 2. 事务处理

涉及多表操作时使用事务:

```javascript
const createWithRelations = async (data) => {
  return await fastify.sequelize.instance.transaction(async (t) => {
    const resource = await models.resource.create(data, { transaction: t });
    await models.relation.create({
      resourceId: resource.id,
      ...relationData
    }, { transaction: t });
    
    return resource;
  });
};
```

### 3. 数据转换

在 Service 层进行数据转换,Controller 层保持简单:

```javascript
const detail = async (authenticatePayload, { id }) => {
  const resource = await models.resource.findByPk(id);
  
  // 数据转换
  resource.setDataValue('displayName', 
    `${resource.firstName} ${resource.lastName}`
  );
  
  // 关联数据
  resource.setDataValue('relations',
    await services.relation.list(authenticatePayload, { resourceId: id })
  );
  
  return resource;
};
```

### 4. 分页查询标准化

```javascript
const list = async (authenticatePayload, { filter = {}, perPage = 20, currentPage = 1 }) => {
  const { tenantId } = authenticatePayload;
  const where = { tenantId };

  // 从 filter 中构建查询条件
  // ...

  const { count, rows } = await models.resource.findAndCountAll({
    where,
    limit: perPage,
    offset: perPage * (currentPage - 1),
    order: [['createdAt', 'DESC']]
  });

  return {
    pageData: rows,
    totalCount: count
  };
};
```

### 6. 软删除

使用 `paranoid: true` 实现软删除:

```javascript
options: {
  paranoid: true,  // 启用软删除
    timestamps: true
}

// 删除操作
await resource.destroy();  // 软删除,设置 deletedAt

// 恢复操作
await resource.restore();  // 恢复删除

// 硬删除
await resource.destroy({ force: true });  // 真实删除
```

### 7. 扩展性设计

预留扩展点,方便定制:

```javascript
options = {
  // 钩子函数
  beforeCreate: async (data) => data,
  afterCreate: async (resource) => {},

  // 自定义验证
  validateCreate: async (data) => {
    if (!data.custom) {
      throw new Error('自定义验证失败');
    }
  },

  // 自定义处理器
  customProcessor: async (resource) => {
    // 自定义处理逻辑
  }
};
```

## 目录结构规范

```
your-plugin/
├── index.js                 # 插件入口
├── libs/                    # 核心代码
│   ├── controllers/         # 路由控制器
│   │   ├── resource.js
│   │   └── admin.js
│   ├── services/            # 业务服务
│   │   ├── resource.js
│   │   └── category.js
│   ├── models/              # 数据模型
│   │   ├── resource.js
│   │   └── relation.js
│   ├── schema/              # 数据验证
│   │   ├── resource.js
│   │   └── query.js
│   ├── utils/               # 工具函数
│   │   ├── format.js
│   │   └── validate.js
├── package.json
└── README.md
```

## 发布与使用

### 1. package.json 配置

```json
{
  "name": "@scope/fastify-plugin-name",
  "version": "1.0.0",
  "main": "index.js",
  "files": [
    "index.js",
    "libs"
  ],
  "peerDependencies": {
    "fastify": ">=4",
    "fastify-plugin": ">=4",
    "@kne/fastify-namespace": "*",
    "@kne/fastify-sequelize": "*"
  }
}
```

### 2. 使用示例

```javascript
const fastify = require('fastify')();

// 注册依赖插件
await fastify.register(require('@kne/fastify-sequelize'), sequelizeConfig);
await fastify.register(require('@kne/fastify-account'), accountConfig);

// 注册业务插件
await fastify.register(require('your-plugin'), {
  name: 'customModule',
  prefix: '/api/custom',
  maxItems: 50,
  customHook: async (data) => {
    // 自定义处理
  }
});

// 使用插件功能
fastify.get('/test', async (request, reply) => {
  const { services } = fastify.customModule;
  return await services.resource.list({ filter: {} });
});
```

## 总结

构建高质量的 Fastify 业务插件需要:

1. **清晰的架构分层** - Controllers/Services/Models 职责分明
2. **合理的配置管理** - 默认值 + 可覆盖 + 扩展点
3. **健壮的认证机制** - 多层认证 + 上下文传递
4. **优雅的错误处理** - 语义化错误 + 统一响应
5. **良好的扩展性** - 依赖注入 + 钩子函数 + 自定义实现

遵循这些原则,可以构建出功能完整、易于维护、高度可扩展的业务插件系统。
