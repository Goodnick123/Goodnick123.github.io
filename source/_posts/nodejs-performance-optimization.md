---
title: Node.js 性能优化实战指南
date: 2026-02-19 09:00:00
tags:
  - Node.js
  - 性能优化
  - 后端开发
  - JavaScript
categories:
  - 技术分享
keywords: Node.js, 性能优化, 内存管理, 异步编程
description: 深入 Node.js 性能优化技巧，从内存管理到异步编程，全方位提升应用性能
---

## 引言

Node.js 以其非阻塞 I/O 和事件驱动特性成为高性能服务端开发的热门选择。但随着业务复杂度增加，性能问题也随之而来。本文将分享实战中总结的 Node.js 性能优化技巧。

## 1. 内存优化

### 1.1 理解 V8 内存模型

```javascript
// V8 内存限制（64位系统默认约1.4GB）
// 可以通过启动参数调整
node --max-old-space-size=4096 app.js
```

### 1.2 避免内存泄漏

```javascript
// ❌ 错误：全局缓存无限制增长
const cache = {};
function getData(key) {
  if (!cache[key]) {
    cache[key] = fetchData(key); // 永远不清除
  }
  return cache[key];
}

// ✅ 正确：使用 LRU 缓存
const LRU = require('lru-cache');
const cache = new LRU({
  max: 500,              // 最多缓存500条
  maxAge: 1000 * 60 * 60 // 1小时过期
});

function getData(key) {
  if (!cache.has(key)) {
    cache.set(key, fetchData(key));
  }
  return cache.get(key);
}
```

### 1.3 流式处理大文件

```javascript
// ❌ 错误：一次性读取大文件到内存
const content = fs.readFileSync('large-file.txt'); // 内存爆炸

// ✅ 正确：使用流式处理
const fs = require('fs');
const readStream = fs.createReadStream('large-file.txt', {
  encoding: 'utf8',
  highWaterMark: 64 * 1024 // 64KB 分块读取
});

readStream.on('data', chunk => {
  console.log('Received chunk:', chunk.length);
});

readStream.on('end', () => {
  console.log('File reading completed');
});
```

## 2. 异步优化

### 2.1 Promise vs Async/Await

```javascript
// ❌ 错误：串行执行（慢）
async function fetchUsers() {
  const user1 = await fetchUser(1);  // 等待 100ms
  const user2 = await fetchUser(2);  // 等待 100ms
  const user3 = await fetchUser(3);  // 等待 100ms
  return [user1, user2, user3];      // 总计 300ms
}

// ✅ 正确：并行执行（快）
async function fetchUsers() {
  const [user1, user2, user3] = await Promise.all([
    fetchUser(1),
    fetchUser(2),
    fetchUser(3)
  ]);
  return [user1, user2, user3];      // 总计 100ms
}

// ✅ 更优：控制并发数
const pLimit = require('p-limit');
const limit = pLimit(5); // 最多5个并发

async function fetchAllUsers(userIds) {
  return Promise.all(
    userIds.map(id => limit(() => fetchUser(id)))
  );
}
```

### 2.2 使用 Worker Threads

```javascript
const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');

if (isMainThread) {
  // 主线程
  module.exports = function parseData(data) {
    return new Promise((resolve, reject) => {
      const worker = new Worker(__filename, {
        workerData: data
      });
      
      worker.on('message', resolve);
      worker.on('error', reject);
      worker.on('exit', code => {
        if (code !== 0) reject(new Error(`Worker stopped with code ${code}`));
      });
    });
  };
} else {
  // Worker 线程
  const result = heavyComputation(workerData);
  parentPort.postMessage(result);
}

// CPU 密集型任务不阻塞主线程
const parseData = require('./worker');
const result = await parseData(largeDataset);
```

### 2.3 集群模式

```javascript
const cluster = require('cluster');
const os = require('os');

if (cluster.isMaster) {
  const numCPUs = os.cpus().length;
  
  console.log(`Master ${process.pid} is running`);
  
  // Fork workers
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }
  
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died`);
    cluster.fork(); // 自动重启
  });
} else {
  require('./app.js');
  console.log(`Worker ${process.pid} started`);
}
```

## 3. 数据库优化

### 3.1 连接池

```javascript
// ❌ 错误：每次请求创建新连接
app.get('/users', async (req, res) => {
  const client = new Client(config); // 慢！
  await client.connect();
  const result = await client.query('SELECT * FROM users');
  await client.end();
  res.json(result.rows);
});

// ✅ 正确：使用连接池
const pool = new Pool({
  host: 'localhost',
  database: 'mydb',
  user: 'user',
  password: 'password',
  max: 20,        // 最大连接数
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

app.get('/users', async (req, res) => {
  const client = await pool.connect(); // 从池中获取
  try {
    const result = await client.query('SELECT * FROM users');
    res.json(result.rows);
  } finally {
    client.release(); // 归还到池中
  }
});
```

### 3.2 查询优化

```javascript
// ❌ 错误：N+1 查询
const users = await db.query('SELECT * FROM users');
for (const user of users) {
  user.orders = await db.query('SELECT * FROM orders WHERE user_id = ?', [user.id]);
}

// ✅ 正确：JOIN 查询
const users = await db.query(`
  SELECT u.*, o.id as order_id, o.total
  FROM users u
  LEFT JOIN orders o ON u.id = o.user_id
`);

// 或使用数据加载器自动优化
const DataLoader = require('dataloader');
const orderLoader = new DataLoader(async (userIds) => {
  const orders = await db.query(
    'SELECT * FROM orders WHERE user_id IN (?)',
    [userIds]
  );
  return userIds.map(id => orders.filter(o => o.user_id === id));
});
```

## 4. HTTP 优化

### 4.1 压缩

```javascript
const compression = require('compression');
const express = require('express');
const app = express();

// 启用 gzip 压缩
app.use(compression({
  level: 6,        // 压缩级别（1-9）
  filter: (req, res) => {
    if (req.headers['x-no-compression']) return false;
    return compression.filter(req, res);
  }
}));

// 静态文件缓存
app.use(express.static('public', {
  maxAge: '1d',
  etag: true,
  lastModified: true
}));
```

### 4.2 使用 HTTP/2

```javascript
const spdy = require('spdy');
const fs = require('fs');

const options = {
  key: fs.readFileSync('server.key'),
  cert: fs.readFileSync('server.crt'),
  spdy: {
    protocols: ['h2', 'http/1.1']
  }
};

spdy.createServer(options, app).listen(443, () => {
  console.log('HTTP/2 server running on port 443');
});
```

## 5. 监控与诊断

### 5.1 性能分析

```javascript
// 启用性能钩子
const { performance, PerformanceObserver } = require('perf_hooks');

const obs = new PerformanceObserver((items) => {
  items.getEntries().forEach((item) => {
    console.log(`${item.name}: ${item.duration}ms`);
  });
});

obs.observe({ entryTypes: ['measure'] });

// 测量代码性能
performance.mark('A');
await heavyOperation();
performance.mark('B');
performance.measure('heavyOperation', 'A', 'B');
```

### 5.2 内存分析

```bash
# 生成堆快照
node --heapsnapshot-near-heap-limit=3 app.js

# 使用 Chrome DevTools 分析
# 1. 打开 chrome://inspect
# 2. 点击 "Open dedicated DevTools for Node"
```

### 5.3 日志监控

```javascript
const pino = require('pino');
const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport: {
    target: 'pino-pretty'
  }
});

// 记录性能指标
logger.info({
  responseTime: Date.now() - startTime,
  memoryUsage: process.memoryUsage(),
  cpuUsage: process.cpuUsage()
}, 'Request completed');
```

## 6. 部署优化

### 6.1 使用 PM2

```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'my-app',
    script: './app.js',
    instances: 'max',      // 使用所有 CPU
    exec_mode: 'cluster',
    max_memory_restart: '1G',
    env: {
      NODE_ENV: 'production'
    },
    log_date_format: 'YYYY-MM-DD HH:mm:ss',
    error_file: './logs/err.log',
    out_file: './logs/out.log',
    merge_logs: true
  }]
};
```

```bash
# 启动
pm2 start ecosystem.config.js

# 监控
pm2 monit

# 日志
pm2 logs
```

### 6.2 Docker 优化

```dockerfile
# 多阶段构建
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
USER node
CMD ["node", "app.js"]
```

## 总结

Node.js 性能优化是一个系统工程，需要从多个维度考虑：

1. **内存管理** - 避免泄漏，合理使用缓存
2. **异步编程** - 充分利用事件循环
3. **数据库** - 连接池，批量查询
4. **HTTP 层** - 压缩，HTTP/2，缓存
5. **监控** - 性能分析，日志记录
6. **部署** - 集群，容器化

性能优化的黄金法则：**先测量，再优化**。使用 profiling 工具找到真正的瓶颈，而不是盲目优化。

---

**推荐工具**
- [clinic.js](https://clinicjs.org/) - 性能诊断工具
- [0x](https://github.com/davidmarkclements/0x) - 火焰图生成
- [autocannon](https://github.com/mcollina/autocannon) - HTTP 压测工具
- [heapdump](https://github.com/bnoordhuis/node-heapdump) - 堆快照
