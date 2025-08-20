# 企业圈系统设计文档

## 1. 项目概述

### 1.1 项目背景
仿造微信朋友圈，设计并实现企业内部社交圈功能，支持企业员工内部信息分享、互动交流。

### 1.2 业务需求
- 企业用户可以发布或删除企业圈内容
- 同企业成员互相可见
- 支持点赞企业圈内容，24小时内发布且点赞数超过10的内容中，点赞数前三的自动置顶
- 支持企业圈内容评论

### 1.3 性能指标
- 企业数量：1000
- 每日新增内容：500000条
- 日活用户数：5000000
- 预计峰值QPS：10000+
- 响应时间要求：< 200ms

## 2. 系统架构设计

### 2.1 整体架构

```
┌─────────────────────────────────────────────┐
│                客户端层                      │
│           Postman / API Client              │
└─────────────────────────────────────────────┘
                       │
┌─────────────────────────────────────────────┐                   │
│            Gin HTTP Server                  │
│        (路由、中间件、参数验证)               │
└─────────────────────────────────────────────┘
                       │
┌─────────────────────────────────────────────┐
│                业务服务层                    │
│     UserService | PostService | LikeService │
│              (业务逻辑处理)                  │
└─────────────────────────────────────────────┘
                       │
┌─────────────────────────────────────────────┐
│                数据访问层                    │
│   UserRepository | PostRepository | ...     │
│              (数据库操作抽象)                │
└─────────────────────────────────────────────┘
          │                           │
┌─────────────────┐         ┌─────────────────┐
│    MySQL集群    │         │   Redis缓存      │
│  Master | Slave │         │    (热点数据)    │
└─────────────────┘         └─────────────────┘
          │
┌─────────────────────────────────────────────┐
│              消息队列层                      │
│         Apache Kafka + Zookeeper            │
│           (异步任务、数据同步)                │
└─────────────────────────────────────────────┘
```

### 2.2 主要功能

#### 2.2.1 功能拆分
- **用户服务 (User Service)**: 用户信息管理、认证授权
- **企业服务 (Company Service)**: 企业信息管理、成员关系
- **内容服务 (Content Service)**: 企业圈内容发布、删除、查询
- **互动服务 (Interaction Service)**: 点赞、评论功能
- **排序服务 (Ranking Service)**: 内容排序、置顶逻辑
- **推送服务 (Notification Service)**: 消息推送

#### 2.2.2 技术栈选择
- **后端框架**: Gin (高性能HTTP框架)
- **配置管理**: Viper (配置文件管理)
- **日志系统**: Logrus (结构化日志)
- **数据库**: MySQL 8.0 (主存储)
- **缓存**: Redis 6.2.14 (缓存层)
- **消息队列**: Apache Kafka 3.8.0 (异步处理)
- **容器化**: Docker & Docker Compose

## 3. 数据库设计

### 3.1 MySQL 数据库设计

#### 3.1.1 用户表 (users)
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    nickname VARCHAR(50),
    avatar_url VARCHAR(255),
    company_id BIGINT NOT NULL,
    status TINYINT DEFAULT 1, -- 1:active, 0:inactive
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_company_id (company_id),
    INDEX idx_email (email)
);
```

#### 3.1.2 企业表 (companies)
```sql
CREATE TABLE companies (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    code VARCHAR(20) UNIQUE NOT NULL,
    description TEXT,
    logo_url VARCHAR(255),
    status TINYINT DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_code (code)
);
```

#### 3.1.3 企业圈内容表 (posts)
```sql
CREATE TABLE posts (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    company_id BIGINT NOT NULL,
    content TEXT NOT NULL,
    images JSON, -- 存储图片URL数组
    like_count INT DEFAULT 0,
    comment_count INT DEFAULT 0,
    is_pinned TINYINT DEFAULT 0, -- 是否置顶
    pinned_at TIMESTAMP NULL, -- 置顶时间
    status TINYINT DEFAULT 1, -- 1:published, 0:deleted
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_company_created (company_id, created_at DESC),
    INDEX idx_user_created (user_id, created_at DESC),
    INDEX idx_like_count (like_count DESC),
    INDEX idx_pinned (is_pinned, pinned_at DESC)
);
```

#### 3.1.4 点赞表 (likes)
```sql
CREATE TABLE likes (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    post_id BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE KEY uk_user_post (user_id, post_id),
    INDEX idx_post_id (post_id),
    INDEX idx_created_at (created_at)
);
```

#### 3.1.5 评论表 (comments)
```sql
CREATE TABLE comments (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    post_id BIGINT NOT NULL,
    parent_id BIGINT DEFAULT NULL, -- 父评论ID，支持回复
    content TEXT NOT NULL,
    status TINYINT DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_post_created (post_id, created_at DESC),
    INDEX idx_parent_id (parent_id)
);
```

### 3.2 数据库分片策略

#### 3.2.1 水平分片
- **按企业ID分片**: 将不同企业的数据分布到不同的数据库
- **分片数量**: 根据1000个企业，用4-8个分片
- **分片键**: company_id

#### 3.2.2 读写分离
- **主库**: 处理写操作
- **从库**: 处理读操作

### 3.3 Redis 缓存设计

#### 3.3.1 缓存策略
```
# 用户信息缓存
user:info:{user_id} -> User对象 (TTL: 30min)

# 企业圈内容列表缓存
company:posts:{company_id}:page:{page} -> 帖子列表 (TTL: 5min)

# 热门内容缓存 (置顶逻辑)
company:hot_posts:{company_id} -> 热门帖子列表 (TTL: 10min)

# 点赞计数缓存
post:like_count:{post_id} -> 点赞数 (TTL: 1hour)

# 评论计数缓存
post:comment_count:{post_id} -> 评论数 (TTL: 1hour)

# 用户点赞状态缓存
user:likes:{user_id} -> 已点赞帖子ID集合 (TTL: 1hour)
```

#### 3.3.2 缓存更新策略
- **Cache Aside模式**: 先更新数据库，再删除缓存
- **异步更新**: 使用Kafka异步更新缓存

## 4. 核心功能设计

### 4.1 内容发布功能

#### 4.1.1 API设计
```go
// POST /api/v1/posts
type CreatePostRequest struct {
    Content string   `json:"content" binding:"required"`
    Images  []string `json:"images"`
}

type CreatePostResponse struct {
    PostID int64 `json:"post_id"`
}
```

#### 4.1.2 处理流程
1. 参数验证 (内容长度、图片数量限制)
2. 用户权限验证
3. 内容审核 (敏感词过滤)
4. 数据库插入
5. 缓存更新
6. 异步任务 (推送通知、内容索引)

### 4.2 内容查询功能

#### 4.2.1 API设计
```go
// GET /api/v1/companies/{company_id}/posts
type GetPostsRequest struct {
    Page     int `form:"page" binding:"min=1"`
    PageSize int `form:"page_size" binding:"min=1,max=20"`
}

type GetPostsResponse struct {
    Posts      []PostInfo `json:"posts"`
    TotalCount int64      `json:"total_count"`
    HasMore    bool       `json:"has_more"`
}
```

#### 4.2.2 排序逻辑
1. **置顶内容**: 24小时内发布且点赞数>10的前3名
2. **普通内容**: 按发布时间倒序
3. **缓存策略**: 热门内容缓存10分钟，普通内容缓存5分钟

### 4.3 点赞功能

#### 4.3.1 API设计
```go
// POST /api/v1/posts/{post_id}/like
// DELETE /api/v1/posts/{post_id}/like
```

#### 4.3.2 处理流程
1. 用户权限验证
2. 防重复点赞检查
3. 数据库操作 (事务)
4. 缓存更新
5. 异步任务 (排行榜更新、推送通知)

### 4.4 评论功能

#### 4.4.1 API设计
```go
// POST /api/v1/posts/{post_id}/comments
type CreateCommentRequest struct {
    Content  string `json:"content" binding:"required"`
    ParentID *int64 `json:"parent_id"`
}

// GET /api/v1/posts/{post_id}/comments
```

### 4.5 置顶逻辑实现

#### 4.5.1 定时任务设计
```go
// 每5分钟执行一次置顶计算
func CalculateTopPosts() {
    // 1. 查询24小时内发布的内容
    // 2. 过滤点赞数>10的内容
    // 3. 按点赞数排序取前3名
    // 4. 更新置顶状态
    // 5. 清除相关缓存
}
```

## 6. 安全设计

### 6.1 身份认证
- JWT Token认证
- Token刷新机制

### 6.2 权限控制
- RBAC权限模型
- 企业级数据隔离
- API接口权限校验


## 7. 日志设计

```go
// 结构化日志格式
type LogEntry struct {
    Level     string    `json:"level"`
    Timestamp time.Time `json:"timestamp"`
    Service   string    `json:"service"`
    TraceID   string    `json:"trace_id"`
    UserID    int64     `json:"user_id"`
    Action    string    `json:"action"`
    Message   string    `json:"message"`
    Duration  int64     `json:"duration_ms"`
    Error     string    `json:"error,omitempty"`
}
```

## 8. 部署方案

Docker容器化部署 mysql、redis、kafka 服务，编写环境、启动、测试、停止脚本。
