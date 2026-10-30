# 樱花别院 — 运维部署手册

> 灰·韵 HUI·YUN 旗下产品

---

## 一、项目简介

**樱花别院** 是**灰·韵 HUI·YUN** 旗下的个人聚合页（Link-in-Bio）产品。每位用户通过专属链接（如 `153900.xyz/用户名`）拥有独立主页，可自由聚合社交链接、展示音乐品味、定制主题外观。

"灰·韵"：灰，是包容一切的底色；韵，是跳脱而出的个性。在低调中蕴藏每个人独特的节奏。

### 核心功能

| 功能模块 | 说明 |
|---------|------|
| 用户注册/登录 | 邮箱验证码 + 密码双模式登录，支持验证码免密登录 |
| 个人主页定制 | 主题切换（6 套预设）、自定义强调色、背景图、头像 |
| 社交链接聚合 | 支持 15+ 主流平台，支持自定义链接与纯文本类型 |
| 音乐播放器 | 网易云音乐外链与自定义音频播放，支持自动播放 |
| 发现页 | 浏览所有公开用户，按最新/最多赞/最多访问排序 |
| 点赞互动 | IP 匿名化去重点赞/取消点赞 |
| 访问统计 | 5 分钟去重统计，IP 匿名化存储 |
| 二维码分享 | 自动生成用户主页二维码 |
| 管理员后台 | 用户管理（搜索/排序/查看详情）、数据统计、角色管理 |
| 账户安全 | 修改密码、注销账户（级联删除）、数据导出（GDPR） |
| 性别设置 | 用户注册时选择性别，影响默认主题分配 |

### 技术栈

| 组件 | 技术选型 |
|------|---------|
| 后端语言 | 原生 PHP（无框架） |
| 数据库 | MySQL 5.7+（PDO 驱动，InnoDB 引擎） |
| 前端 | 原生 HTML5 / CSS3（CSS Variables 主题系统） / JavaScript |
| 邮件发送 | SMTP 直连 → Resend API → PHP mail() 三级降级 |
| 认证方案 | 自实现 JWT（HMAC-SHA256 签名） |
| 密码加密 | PHP `password_hash()` + `PASSWORD_BCRYPT` |
| Web 服务器 | Nginx + PHP CGI/FastCGI |
| 部署环境 | 虚拟主机（小兔互联） / 任意支持 PHP 的主机 |

### 项目特色

- **零依赖**：不使用任何 PHP 框架或前端库，纯原生实现
- **轻量级**：全部后端代码仅 8 个 PHP 文件，部署极其简单
- **隐私优先**：IP 匿名化存储（SHA-256），不记录原始 IP 地址
- **安全多层防护**：CSP、HSTS、XSS 过滤、速率限制、输入净化、SQL 预处理
- **自动初始化**：首次访问自动建表、迁移、创建管理员，无需手动导入 SQL

---

## 二、环境要求

### 必需环境

| 要求 | 最低版本 | 说明 |
|------|---------|------|
| PHP | >= 7.4 | 需要 PDO、openssl、mbstring、json、curl 扩展 |
| MySQL | >= 5.7 | 或 MariaDB >= 10.2，需支持 InnoDB 和 utf8mb4 字符集 |
| Nginx | 任意版本 | 需支持 `try_files` 和 `rewrite` 伪静态指令 |
| 虚拟主机 | 支持 PHP CGI/FastCGI | 需支持自定义 Nginx 伪静态规则或 .htaccess |

### PHP 扩展需求

| 扩展 | 用途 | 必需 |
|------|------|------|
| `PDO` + `pdo_mysql` | MySQL 数据库连接 | 是 |
| `openssl` | JWT HMAC 签名、SMTP SSL/TLS 连接 | 是 |
| `mbstring` | 中文/Unicode 字符处理（截取、正则） | 是 |
| `json` | API JSON 响应编码/解码 | 是 |
| `curl` | Resend API 备用邮件发送 | 是 |
| `fileinfo` | 上传文件 MIME 类型检测 | 建议开启 |
| `gd` 或 `imagick` | 图片处理（可选，当前未使用） | 否 |

检查已安装扩展：

```bash
php -m | grep -iE 'pdo|openssl|mbstring|json|curl|fileinfo'
```

---

## 三、项目结构

```
huiyun/
├── index.php              # 前端主路由 — 通过 path 参数分发 HTML 页面和用户主页
├── p.php                  # 页面路由器 — 通过 page 参数分发 HTML 页面（备用入口）
├── u.php                  # 用户主页兼容入口 — /u.php?u=username 转发到 p.php
├── api.php                # API 统一入口 — 所有 REST 接口通过 path 参数路由
├── config.php             # 集中配置 — 数据库、JWT、SMTP、上传、管理员、速率限制
├── db.php                 # 数据库层 — PDO 连接、查询封装、建表与自动迁移
├── helpers.php            # 工具函数 — JSON 响应、JWT 认证、邮件发送（三级降级）、验证码
├── security.php           # 安全工具 — HTTP 安全头、IP 匿名化、XSS 防护、速率限制、错误掩码
├── seed.php               # 管理员种子脚本 — 独立管理页面，部署后建议删除
├── nginx-rewrite.txt      # Nginx 伪静态规则参考文件
├── .gitignore             # Git 忽略规则
│
├── css/
│   └── style.css          # 全局样式表 — 6 套主题 CSS Variables 定义
├── js/
│   └── common.js          # 通用 JavaScript 工具 — 导航栏、Token 管理等
├── assets/
│   └── default-avatar.svg # 默认头像 SVG 图标
│
├── uploads/               # 用户上传文件目录（需可写权限）
│   ├── avatars/           #   用户头像
│   │   └── .gitkeep
│   └── backgrounds/       #   用户背景图
│       └── .gitkeep
│
├── index.html             # 发现页（首页）— 展示所有公开用户
├── login.html             # 登录/注册页
├── dashboard.html         # 用户管理后台 — 编辑资料、链接、音乐、主题
├── profile.html           # 用户公开主页 — 由前端 JS 读取 URL 参数动态渲染
├── admin.html             # 管理员后台 — 用户管理、数据统计
├── qrcode.html            # 二维码分享页
├── about.html             # 关于我们（静态 HTML，由 index.php 路由）
├── privacy.html           # 隐私政策（静态 HTML，由 index.php 路由）
├── terms.html             # 服务条款（静态 HTML，由 index.php 路由）
│
├── about.php              # 关于我们（独立 PHP 页面，带 session_start）
├── admin.php              # 管理后台（独立 PHP 页面）
├── dashboard.php          # 用户后台（独立 PHP 页面）
├── login.php              # 登录页（独立 PHP 页面）
├── privacy.php            # 隐私政策（独立 PHP 页面）
├── qrcode.php             # 二维码页面（独立 PHP 页面）
├── terms.php              # 服务条款（独立 PHP 页面）
│
└── DEPLOY.md              # 本文件 — 运维部署手册
```

### 文件职责说明

| 文件 | 类型 | 说明 |
|------|------|------|
| `index.php` | PHP 入口 | 前端主路由。通过 `path` 参数分发 HTML 页面，自动初始化数据库和创建管理员。非用户名路径按映射表分发，其他路径统一返回 `profile.html` 作为用户主页 |
| `p.php` | PHP 入口 | 页面路由器备用入口。通过 `page` 参数分发 HTML 页面，可脱离 `index.php` 直接使用 |
| `u.php` | PHP 入口 | 兼容旧版 URL 格式 `/u.php?u=username`，内部设置 `page=profile` 后转发到 `p.php` |
| `api.php` | PHP 入口 | 所有 API 接口的统一入口。解析 `path` 参数路由到具体处理函数，处理 CORS、方法覆盖、安全头 |
| `config.php` | 配置 | 全站配置常量，包含站点、数据库、JWT、SMTP、Resend API、上传限制、管理员信息、速率限制 |
| `db.php` | 核心库 | PDO 数据库连接（单例模式）、`queryAll`/`queryOne`/`dbRun` 查询封装、`initDatabase()` 自动建表与增量迁移 |
| `helpers.php` | 核心库 | JSON 响应格式化、JWT 生成/验证（HMAC-SHA256）、邮件发送（SMTP 直连/Resend API/PHP mail 三级降级）、验证码生成与校验、速率限制、HTML 转义 |
| `security.php` | 核心库 | HTTP 安全头设置（CSP/HSTS/X-Frame-Options 等 7 项）、IP 匿名化（SHA-256 + 盐值）、URL 净化（协议白名单）、社交链接净化、输入净化、速率限制（文件系统滑动窗口）、数据库错误掩码 |
| `seed.php` | 工具脚本 | 独立的管理员账号创建/管理页面。支持创建、重置密码、删除管理员。部署完成后应删除或重命名 |
| `nginx-rewrite.txt` | 参考文件 | Nginx 伪静态规则文本，供面板配置时复制粘贴 |
| `*.html` | 前端页面 | 纯静态 HTML 页面，由 PHP 路由通过 `readfile()` 输出 |
| `*.php`（页面） | 前端页面 | 各页面的独立 PHP 入口版本，包含 `session_start()` 和完整 HTML，可直接通过 URL 访问 |

> **注意**：`*.html` 和对应的 `*.php` 是同一页面的两种入口方式。`index.php` 路由使用 `readfile()` 输出 HTML 文件；而独立 PHP 文件可绕过路由直接访问（如 `/about.php`）。注意：项目没有 `public/` 目录，所有文件均位于根目录下。

---

## 四、部署步骤

### 4.1 上传文件

1. 将整个 `huiyun/` 目录上传至 Web 根目录
2. 确保 `uploads/` 及其子目录具有写入权限：

```bash
chmod 755 uploads/
chmod 755 uploads/avatars/
chmod 755 uploads/backgrounds/
```

### 4.2 配置 config.php

编辑 `config.php`，根据实际环境修改以下配置项（详见第六章完整说明）：

```php
// 站点 URL（必须与实际访问地址一致，用于邮件链接和二维码生成）
define('SITE_URL', 'https://your-domain.com');
define('SITE_NAME', '灰·韵 HUI·YUN');

// 数据库连接信息（需提前在主机面板创建数据库）
define('DB_HOST', 'localhost');
define('DB_PORT', '3306');
define('DB_NAME', 'your_database_name');
define('DB_USER', 'your_database_user');
define('DB_PASS', 'your_database_password');
define('DB_CHARSET', 'utf8mb4');

// JWT 密钥（必须修改为随机字符串！）
define('JWT_SECRET', 'your_random_secret_key_at_least_32_chars');
define('JWT_EXPIRES', 30 * 24 * 3600); // 30天

// SMTP 邮件配置（Resend SMTP 中继）
define('SMTP_HOST', 'smtp.resend.com');
define('SMTP_PORT', '465');
define('SMTP_SECURE', 'ssl');
define('SMTP_USER', 'resend');
define('SMTP_PASS', 'your_resend_smtp_key');
define('SMTP_FROM', 'noreply@your-domain.com');

// Resend API（备用，SMTP 优先）
define('RESEND_API_KEY', 'your_resend_api_key');
define('RESEND_FROM', 'noreply@your-domain.com');

// 管理员信息
define('ADMIN_USERNAME', 'admin');
define('ADMIN_NICKNAME', '灰·韵管理员');
define('ADMIN_EMAIL', 'admin@your-domain.com');
define('ADMIN_PASSWORD', 'your_secure_admin_password');

// 速率限制
define('RATE_LIMIT_API', 200);      // 每15分钟
define('RATE_LIMIT_AUTH', 20);      // 每15分钟
```

> **安全警告**：
> - `JWT_SECRET` 必须改为不可预测的随机字符串，否则所有 Token 可被伪造
> - `ADMIN_PASSWORD` 部署后必须立即修改
> - 生成随机密钥：`openssl rand -hex 32`

### 4.3 访问首页自动初始化数据库

无需手动建表或导入 SQL。系统在首次访问时自动执行：

1. **访问任意页面**（如 `https://your-domain.com/`），触发 `index.php` 或 `p.php`
2. `initDatabase()` 自动创建全部 6 张数据库表（`users`、`email_verifications`、`social_links`、`music_settings`、`likes`、`visit_stats`）
3. 自动运行增量迁移（添加新列、新索引，兼容旧版本数据库升级）
4. 自动创建管理员账号（使用 `config.php` 中定义的管理员信息）
5. 自动清理过期验证码（`cleanExpiredVerifications()`）

### 4.4 运行 seed.php 管理管理员账号

也可通过独立的种子脚本手动管理管理员：

```
https://your-domain.com/seed.php
```

该页面提供以下操作：
- 自动创建管理员（如果不存在）
- 重置管理员密码为 `config.php` 中的默认值
- 删除管理员账号（刷新后重新创建）

### 4.5 配置 Nginx 伪静态

登录虚拟主机面板（小兔互联 / 宝塔），在网站设置中找到「伪静态」选项，粘贴 `nginx-rewrite.txt` 中的规则（详见第五章）。

### 4.6 验证部署

按以下顺序检查各功能是否正常：

| 步骤 | 访问地址 | 预期结果 |
|------|---------|---------|
| 1 | `https://your-domain.com/` | 显示发现页 |
| 2 | `https://your-domain.com/login` | 显示登录/注册页 |
| 3 | `https://your-domain.com/api/discover` | 返回 JSON 数据 |
| 4 | 注册新账号 | 收到验证码邮件，完成注册 |
| 5 | `https://your-domain.com/dashboard` | 进入用户管理后台 |
| 6 | `https://your-domain.com/你的用户名` | 显示个人主页 |
| 7 | `https://your-domain.com/admin` | 管理员后台（需管理员登录） |
| 8 | `https://your-domain.com/seed.php` | 显示管理员管理页面 |

---

## 五、Nginx 配置

### 5.1 伪静态规则

从 `nginx-rewrite.txt` 中提取的规则：

```nginx
# API 请求 → api.php
location /api/ {
    rewrite ^/api/(.*)$ /api.php?path=$1 last;
}

# 二维码页面 /qrcode/用户名 → qrcode.html
location ~ ^/qrcode/(.+)$ {
    try_files $uri $uri/ /qrcode.html;
}

# 静态文件直接返回（CSS/JS/图片/上传文件/字体/音频）
location ~* \.(css|js|png|jpg|jpeg|gif|svg|webp|ico|woff|woff2|ttf|eot|mp3)$ {
    try_files $uri =404;
}

# 其他所有请求 → index.php（前端路由）
# 支持: /login, /dashboard, /about, /admin, /用户名 等
location / {
    try_files $uri $uri/ /index.php?path=$uri&$args;
}
```

### 5.2 完整 Nginx 站点配置示例

```nginx
server {
    listen 80;
    server_name your-domain.com www.your-domain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com www.your-domain.com;

    # SSL 证书配置（路径根据实际环境修改）
    ssl_certificate     /path/to/fullchain.pem;
    ssl_certificate_key /path/to/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    # 网站根目录
    root /path/to/huiyun;
    index index.php index.html;

    # 字符编码
    charset utf-8;

    # ---------- 安全头 ----------
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-XSS-Protection "1; mode=block";
    add_header Referrer-Policy strict-origin-when-cross-origin;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=(), payment=()";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # ---------- 静态文件缓存 ----------
    location ~* \.(css|js|png|jpg|jpeg|gif|svg|webp|ico|woff|woff2|ttf|eot|mp3)$ {
        try_files $uri =404;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # ---------- API 请求 ----------
    location /api/ {
        rewrite ^/api/(.*)$ /api.php?path=$1 last;
    }

    # ---------- 二维码页面 ----------
    location ~ ^/qrcode/(.+)$ {
        try_files $uri $uri/ /qrcode.html;
    }

    # ---------- 上传文件保护 ----------
    location /uploads/ {
        try_files $uri =404;
        # 防止 PHP 在上传目录执行
        location ~* \.php$ {
            deny all;
        }
    }

    # ---------- PHP 处理 ----------
    location ~ \.php$ {
        fastcgi_pass unix:/tmp/php-cgi.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_read_timeout 300;
        fastcgi_buffer_size 64k;
        fastcgi_buffers 4 64k;
    }

    # ---------- 前端路由 ----------
    location / {
        try_files $uri $uri/ /index.php?path=$uri&$args;
    }

    # ---------- 禁止访问敏感文件 ----------
    location ~ \.(env|log|sql|bak|md|txt)$ {
        deny all;
    }

    # 禁止访问隐藏文件
    location ~ /\. {
        deny all;
    }
}
```

### 5.3 伪静态规则说明

| 规则 | 匹配示例 | 转发目标 | 说明 |
|------|---------|---------|------|
| `/api/*` | `/api/discover`, `/api/auth/login` | `api.php?path=discover` | 所有 API 请求统一转发到 `api.php`，通过 `path` 参数路由 |
| `/qrcode/*` | `/qrcode/alice` | `qrcode.html` | 二维码页面由前端 JS 读取 URL 路径获取用户名 |
| 静态文件后缀 | `*.css`, `*.js`, `*.png` 等 | 直接返回 | 不经过 PHP 处理，提升性能 |
| `/*` (其他) | `/login`, `/dashboard`, `/alice` | `index.php?path=xxx` | 前端路由分发，`index.php` 根据 path 映射到对应 HTML 页面或用户主页 |

> **虚拟主机环境**：PHP CGI 的 socket 路径（`fastcgi_pass`）由主机商预设，通常无需手动配置。只需粘贴伪静态规则即可。

### 5.4 备选方案

如果伪静态规则不生效，可使用最简版本：

```nginx
location / {
    try_files $uri $uri/ /index.php?path=$uri&$args;
}

location /api/ {
    rewrite ^/api/(.*)$ /api.php?path=$1 last;
}
```

或直接使用参数格式访问：
- 页面：`https://your-domain.com/p.php?page=login`
- 用户主页：`https://your-domain.com/u.php?u=username`

---

## 六、config.php 配置说明

以下是 `config.php` 中所有配置常量的完整说明：

### 6.1 站点配置

| 常量 | 示例值 | 说明 |
|------|--------|------|
| `SITE_URL` | `https://153900.xyz` | 站点完整 URL，用于邮件中的链接、二维码生成、CORS 白名单。必须与实际访问地址一致 |
| `SITE_NAME` | `灰·韵 HUI·YUN` | 站点显示名称，用于邮件模板中的品牌展示 |

### 6.2 数据库配置

| 常量 | 示例值 | 说明 |
|------|--------|------|
| `DB_HOST` | `localhost` | 数据库主机地址。虚拟主机通常为 `localhost`，远程数据库填写 IP 或域名 |
| `DB_PORT` | `3306` | 数据库端口，MySQL 默认 3306 |
| `DB_NAME` | `r5w6suhuvinwe` | 数据库名称，需提前在主机面板创建 |
| `DB_USER` | `r5w6suhuvinwe` | 数据库用户名 |
| `DB_PASS` | `your_password` | 数据库密码 |
| `DB_CHARSET` | `utf8mb4` | 字符集，**必须为 `utf8mb4`** 以支持中文、Emoji 和特殊字符 |

### 6.3 JWT 配置

| 常量 | 示例值 | 说明 |
|------|--------|------|
| `JWT_SECRET` | `huiyun_secret_key_...` | **必须修改**。JWT HMAC-SHA256 签名密钥，任意随机字符串，建议至少 32 字符。修改后所有旧 Token 立即失效 |
| `JWT_EXPIRES` | `2592000` (30天) | Token 有效期（秒）。默认 30 天（`30 * 24 * 3600`） |

### 6.4 SMTP 邮件配置

| 常量 | 示例值 | 说明 |
|------|--------|------|
| `SMTP_HOST` | `smtp.resend.com` | SMTP 服务器地址 |
| `SMTP_PORT` | `465` | SMTP 端口。`465` = SSL 直连，`587` = STARTTLS |
| `SMTP_SECURE` | `ssl` | 加密方式：`ssl`（对应 465 端口）或 `tls`（对应 587 端口） |
| `SMTP_USER` | `resend` | SMTP 认证用户名（Resend 固定为 `resend`） |
| `SMTP_PASS` | `re_xxxxx` | SMTP 认证密码（Resend 的 SMTP 密钥） |
| `SMTP_FROM` | `noreply@153900.xyz` | 发件人邮箱，必须是在 Resend 中验证过的域名邮箱 |

### 6.5 Resend API 配置（备用）

| 常量 | 示例值 | 说明 |
|------|--------|------|
| `RESEND_API_KEY` | `re_xxxxx` | Resend API Key，当 SMTP 直连失败时自动降级使用 |
| `RESEND_FROM` | `noreply@153900.xyz` | API 模式的发件人地址 |

**邮件发送优先级**：SMTP 直连 → Resend API → PHP `mail()`，任一方式成功即停止。如果所有方式都失败，验证码会记录到 PHP `error_log` 中。

### 6.6 上传配置

| 常量 | 值 | 说明 |
|------|-----|------|
| `UPLOAD_DIR` | `__DIR__ . '/uploads'` | 上传文件根目录（绝对路径，基于 `config.php` 所在目录） |
| `AVATAR_MAX_SIZE` | `5242880` (5MB) | 头像最大文件大小（字节） |
| `BG_MAX_SIZE` | `10485760` (10MB) | 背景图最大文件大小（字节） |

### 6.7 管理员配置

| 常量 | 示例值 | 说明 |
|------|--------|------|
| `ADMIN_USERNAME` | `admin` | 管理员用户名，用于自动创建和检测 |
| `ADMIN_NICKNAME` | `灰·韵管理员` | 管理员显示昵称 |
| `ADMIN_EMAIL` | `admin@153900.xyz` | 管理员邮箱 |
| `ADMIN_PASSWORD` | `admin123456` | 管理员初始密码。**部署后务必立即登录修改** |

> 管理员账号在以下时机自动创建/维护：
> - 首次访问 `index.php`（通过 `seedAdmin()` 函数）
> - 首次访问 `p.php`（通过 `seedAdminAuto()` 函数）
> - 手动访问 `seed.php`（提供完整管理界面）

### 6.8 速率限制配置

| 常量 | 值 | 说明 |
|------|-----|------|
| `RATE_LIMIT_API` | `200` | API 通用速率限制：每 15 分钟最大请求数 |
| `RATE_LIMIT_AUTH` | `20` | 认证接口速率限制：每 15 分钟最大请求数 |

---

## 七、数据库结构

系统使用 InnoDB 引擎，utf8mb4 字符集。所有表在首次访问时由 `initDatabase()` 自动创建。

### 7.1 users — 用户表

```sql
CREATE TABLE IF NOT EXISTS users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(20) NOT NULL UNIQUE,
    nickname VARCHAR(30) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(10) DEFAULT 'user',
    avatar_url VARCHAR(255) DEFAULT '',
    bio TEXT,
    is_public TINYINT(1) DEFAULT 1,
    custom_bg_url VARCHAR(255) DEFAULT '',
    theme_name VARCHAR(20) DEFAULT 'sakura',
    accent_color VARCHAR(10) DEFAULT '#ff6b9d',
    gender VARCHAR(10) DEFAULT 'secret',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_username (username),
    INDEX idx_email (email),
    INDEX idx_role (role),
    INDEX idx_gender (gender)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `id` | INT AUTO_INCREMENT PK | — | 用户 ID |
| `username` | VARCHAR(20) UNIQUE | — | 用户名（2-20 位，允许字母/数字/下划线/中文） |
| `nickname` | VARCHAR(30) | — | 显示昵称 |
| `email` | VARCHAR(100) UNIQUE | — | 邮箱 |
| `password_hash` | VARCHAR(255) | — | bcrypt 加密后的密码哈希 |
| `role` | VARCHAR(10) | `'user'` | 角色：`user`（普通用户）或 `admin`（管理员） |
| `avatar_url` | VARCHAR(255) | `''` | 头像 URL（相对路径或绝对 URL） |
| `bio` | TEXT | — | 个人简介 |
| `is_public` | TINYINT(1) | `1` | 主页是否公开（`0`=隐藏，`1`=公开） |
| `custom_bg_url` | VARCHAR(255) | `''` | 自定义背景图 URL |
| `theme_name` | VARCHAR(20) | `'sakura'` | 主题名称（`latte`/`slate`/`forest`/`sunset`/`ocean`/`sakura`） |
| `accent_color` | VARCHAR(10) | `'#ff6b9d'` | 自定义强调色（十六进制颜色值） |
| `gender` | VARCHAR(10) | `'secret'` | 性别：`male`（男）/ `female`（女）/ `secret`（保密） |
| `created_at` | DATETIME | `CURRENT_TIMESTAMP` | 注册时间 |

**索引**：`idx_username`、`idx_email`、`idx_role`、`idx_gender`

### 7.2 email_verifications — 邮箱验证码表

```sql
CREATE TABLE IF NOT EXISTS email_verifications (
    id INT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(100) NOT NULL,
    code VARCHAR(10) NOT NULL,
    type VARCHAR(20) NOT NULL DEFAULT 'register',
    expires_at DATETIME NOT NULL,
    used TINYINT(1) DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `id` | INT AUTO_INCREMENT PK | — | 自增 ID |
| `email` | VARCHAR(100) | — | 目标邮箱 |
| `code` | VARCHAR(10) | — | 6 位数字验证码 |
| `type` | VARCHAR(20) | `'register'` | 验证类型：`register`（注册）或 `login`（登录） |
| `expires_at` | DATETIME | — | 过期时间（验证码 10 分钟有效） |
| `used` | TINYINT(1) | `0` | 是否已使用（`0`=未使用，`1`=已使用） |
| `created_at` | DATETIME | `CURRENT_TIMESTAMP` | 创建时间 |

**索引**：`idx_email`

### 7.3 social_links — 社交链接表

```sql
CREATE TABLE IF NOT EXISTS social_links (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    platform VARCHAR(30) NOT NULL,
    platform_label VARCHAR(30) NOT NULL,
    url VARCHAR(500) NOT NULL,
    icon_svg TEXT,
    sort_order INT DEFAULT 0,
    link_type VARCHAR(10) DEFAULT 'link',
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_user (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `id` | INT AUTO_INCREMENT PK | — | 自增 ID |
| `user_id` | INT NOT NULL | — | 所属用户 ID（外键 → users.id） |
| `platform` | VARCHAR(30) | — | 平台标识（如 `github`、`twitter`、`wechat`） |
| `platform_label` | VARCHAR(30) | — | 平台显示名称（如 `GitHub`、`微信`） |
| `url` | VARCHAR(500) | — | 链接地址（`link_type=text` 时可为空） |
| `icon_svg` | TEXT | — | 自定义 SVG 图标代码 |
| `sort_order` | INT | `0` | 排序顺序（数字越小越靠前） |
| `link_type` | VARCHAR(10) | `'link'` | 链接类型：`link`（链接）或 `text`（纯文本展示） |

**索引**：`idx_user`
**外键**：`user_id` → `users.id` ON DELETE CASCADE

### 7.4 music_settings — 音乐设置表

```sql
CREATE TABLE IF NOT EXISTS music_settings (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL UNIQUE,
    type VARCHAR(20) DEFAULT 'netease',
    music_url VARCHAR(500) DEFAULT '',
    music_id TEXT,
    title VARCHAR(100) DEFAULT '',
    artist VARCHAR(100) DEFAULT '',
    autoplay TINYINT(1) DEFAULT 0,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `id` | INT AUTO_INCREMENT PK | — | 自增 ID |
| `user_id` | INT NOT NULL UNIQUE | — | 所属用户 ID（每个用户唯一一条记录） |
| `type` | VARCHAR(20) | `'netease'` | 音乐类型（`netease` 网易云 / 自定义 URL） |
| `music_url` | VARCHAR(500) | `''` | 外部音乐 URL |
| `music_id` | TEXT | — | 网易云音乐歌曲 ID |
| `title` | VARCHAR(100) | `''` | 歌曲标题 |
| `artist` | VARCHAR(100) | `''` | 艺术家名称 |
| `autoplay` | TINYINT(1) | `0` | 是否自动播放（`0`=否，`1`=是） |

**外键**：`user_id` → `users.id` ON DELETE CASCADE

### 7.5 likes — 点赞表

```sql
CREATE TABLE IF NOT EXISTS likes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    liked_by_ip VARCHAR(64) NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE KEY uk_user_ip (user_id, liked_by_ip),
    INDEX idx_user (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `id` | INT AUTO_INCREMENT PK | — | 自增 ID |
| `user_id` | INT NOT NULL | — | 被点赞的用户 ID |
| `liked_by_ip` | VARCHAR(64) | — | 点赞者 IP 的 **SHA-256 匿名哈希**（不存储原始 IP） |
| `created_at` | DATETIME | `CURRENT_TIMESTAMP` | 点赞时间 |

**索引**：`idx_user`
**唯一索引**：`uk_user_ip`（`user_id` + `liked_by_ip`，确保同一 IP 只能点赞一次）
**外键**：`user_id` → `users.id` ON DELETE CASCADE

### 7.6 visit_stats — 访问统计表

```sql
CREATE TABLE IF NOT EXISTS visit_stats (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    visitor_ip VARCHAR(64) NOT NULL,
    visited_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_user (user_id),
    INDEX idx_visited (user_id, visitor_ip, visited_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `id` | INT AUTO_INCREMENT PK | — | 自增 ID |
| `user_id` | INT NOT NULL | — | 被访问的用户 ID |
| `visitor_ip` | VARCHAR(64) | — | 访客 IP 的 **SHA-256 匿名哈希**（不存储原始 IP） |
| `visited_at` | DATETIME | `CURRENT_TIMESTAMP` | 访问时间 |

**索引**：`idx_user`、`idx_visited`（`user_id` + `visitor_ip` + `visited_at` 复合索引，用于 5 分钟去重查询）
**外键**：`user_id` → `users.id` ON DELETE CASCADE

### 7.7 ER 关系图

```
users (1) ──< social_links (N)     ON DELETE CASCADE
users (1) ──< music_settings (1)   ON DELETE CASCADE
users (1) ──< likes (N)            ON DELETE CASCADE
users (1) ──< visit_stats (N)      ON DELETE CASCADE
```

删除用户时，所有关联数据通过外键 `ON DELETE CASCADE` 自动清除，无需手动清理。

### 7.8 自动迁移

`initDatabase()` 在每次请求时执行以下增量迁移（通过 `try-catch` 静默跳过已执行的迁移）：

| 迁移内容 | 目的 |
|---------|------|
| `ALTER TABLE users ADD COLUMN role` | v2.0 新增角色字段 |
| `ALTER TABLE users ADD INDEX idx_role` | 角色查询索引 |
| `ALTER TABLE social_links ADD COLUMN link_type` | v2.1 新增纯文本链接类型 |
| `ALTER TABLE likes MODIFY COLUMN liked_by_ip VARCHAR(64)` | v2.0 IP 哈希扩展存储 |
| `ALTER TABLE visit_stats MODIFY COLUMN visitor_ip VARCHAR(64)` | v2.0 IP 哈希扩展存储 |
| `ALTER TABLE users ADD COLUMN gender` | v2.1 新增性别字段 |
| `ALTER TABLE users ADD INDEX idx_gender` | 性别查询索引 |

---

## 八、API 接口文档

所有 API 接口通过 `api.php` 统一分发，基础路径为 `/api/`。

### 认证方式

在请求头中携带 Token：

```
Authorization: Bearer <token>
```

**虚拟主机兼容备选头**（部分虚拟主机不传递 `Authorization` 头）：

```
X-Authorization: Bearer <token>
X-HTTP-Authorization: Bearer <token>
```

### 方法覆盖

部分虚拟主机不支持 `PUT`/`DELETE` HTTP 方法，可通过 `_method` 参数覆盖：

```
POST /api/user/password?_method=PUT
POST /api/user/account?_method=DELETE
```

`_method` 参数支持从 `$_GET` 或 `$_POST` 中读取。

### 统一响应格式

**成功响应**：
```json
{
  "message": "操作成功",
  "token": "...",
  "user": { ... }
}
```

**错误响应**：
```json
{
  "error": "错误描述信息"
}
```

**HTTP 状态码**：

| 状态码 | 含义 |
|--------|------|
| 200 | 成功 |
| 204 | 成功无内容（OPTIONS 预检请求） |
| 400 | 请求参数错误 |
| 401 | 未登录或 Token 已过期 |
| 403 | 无权限（非管理员 / 页面已隐藏） |
| 404 | 接口或资源不存在 |
| 429 | 请求频率超限 |
| 500 | 服务器内部错误 |

---

### 8.1 认证模块

#### POST /api/auth/send-code — 发送验证码

发送邮箱验证码（用于注册或通用场景）。

- **认证**：否
- **速率限制**：每邮箱 60 秒 1 次

**请求体**（JSON）：
```json
{
  "email": "user@example.com",
  "type": "register"
}
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `email` | string | 是 | 邮箱地址，需符合格式 |
| `type` | string | 否 | 验证类型：`register`（默认）或 `login` |

**成功响应**：
```json
{ "message": "验证码已发送" }
```

---

#### POST /api/auth/get-code — 获取验证码

获取最近一次发送的验证码（开发调试用）。

- **认证**：否

**请求体**（JSON）：
```json
{
  "email": "user@example.com",
  "type": "register"
}
```

**成功响应**：
```json
{ "code": "123456", "message": "验证码已获取" }
```

---

#### POST /api/auth/register — 注册

注册新账号，需先发送验证码。

- **认证**：否

**请求体**（JSON）：
```json
{
  "username": "alice",
  "nickname": "小明",
  "email": "alice@example.com",
  "password": "123456",
  "code": "123456",
  "gender": "secret"
}
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `username` | string | 是 | 用户名，2-20 位字母/数字/下划线/中文 |
| `nickname` | string | 是 | 显示昵称 |
| `email` | string | 是 | 邮箱地址 |
| `password` | string | 是 | 密码，至少 6 位 |
| `code` | string | 是 | 邮箱验证码 |
| `gender` | string | 否 | 性别：`male`/`female`/`secret`（默认 `secret`） |

**成功响应**：
```json
{
  "message": "注册成功",
  "token": "eyJ...",
  "user": {
    "id": 1,
    "username": "alice",
    "nickname": "小明",
    "email": "alice@example.com",
    "avatar_url": "",
    "bio": "",
    "theme_name": "sakura",
    "accent_color": "#ff6b9d",
    "role": "user",
    "gender": "secret"
  }
}
```

---

#### POST /api/auth/login — 密码登录

通过邮箱 + 密码登录。

- **认证**：否

**请求体**（JSON）：
```json
{
  "email": "alice@example.com",
  "password": "123456"
}
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `email` | string | 是 | 邮箱地址 |
| `password` | string | 是 | 密码 |

**成功响应**：
```json
{
  "message": "登录成功",
  "token": "eyJ...",
  "user": {
    "id": 1, "username": "alice", "nickname": "小明",
    "email": "alice@example.com", "avatar_url": "", "bio": "",
    "gender": "secret", "theme_name": "sakura", "accent_color": "#ff6b9d",
    "is_public": 1, "role": "user"
  }
}
```

---

#### POST /api/auth/login/send-code — 发送登录验证码

发送登录专用验证码（需邮箱已注册）。

- **认证**：否

**请求体**（JSON）：
```json
{ "email": "alice@example.com" }
```

**成功响应**：
```json
{ "message": "验证码已发送" }
```

---

#### POST /api/auth/login/verify — 验证码登录

通过邮箱 + 验证码免密登录。

- **认证**：否

**请求体**（JSON）：
```json
{
  "email": "alice@example.com",
  "code": "123456"
}
```

**成功响应**：与密码登录相同格式。

---

#### GET /api/auth/me — 获取当前用户信息

获取当前登录用户的完整信息，包含社交链接、音乐设置、统计数据。

- **认证**：是

**成功响应**：
```json
{
  "user": { "id": 1, "username": "alice", ... },
  "socialLinks": [ { "platform": "github", "url": "...", ... } ],
  "music": { "type": "netease", "music_id": "...", ... },
  "likeCount": 10,
  "visitCount": 128
}
```

---

### 8.2 用户模块

#### GET /api/user/profile/{username} — 获取用户公开主页

获取指定用户的公开主页信息。

- **认证**：否
- **限制**：用户 `is_public=0` 时返回 403

**成功响应**：
```json
{
  "user": { "id": 1, "username": "alice", "avatar_url": "...", ... },
  "socialLinks": [ ... ],
  "music": { ... },
  "likeCount": 10,
  "visitCount": 128
}
```

---

#### PUT /api/user/profile — 更新个人资料

更新当前登录用户的个人资料。

- **认证**：是

**请求体**（JSON，所有字段可选）：
```json
{
  "nickname": "新昵称",
  "bio": "个人简介",
  "avatar_url": "/uploads/avatars/1_1720000000.jpg",
  "is_public": 1,
  "theme_name": "ocean",
  "accent_color": "#0891b2",
  "custom_bg_url": "/uploads/backgrounds/1_1720000000.jpg"
}
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `nickname` | string | 昵称（最多 30 字符，自动净化） |
| `bio` | string | 简介（最多 500 字符，自动净化） |
| `avatar_url` | string | 头像 URL（自动净化，拒绝危险协议） |
| `is_public` | int | 主页可见性：`0`=隐藏，`1`=公开 |
| `theme_name` | string | 主题名称 |
| `accent_color` | string | 强调色（十六进制） |
| `custom_bg_url` | string | 背景图 URL（自动净化） |

---

#### PUT /api/user/social-links — 更新社交链接

全量替换当前用户的社交链接列表。

- **认证**：是

**请求体**（JSON）：
```json
{
  "links": [
    {
      "platform": "github",
      "platform_label": "GitHub",
      "url": "https://github.com/alice",
      "link_type": "link"
    },
    {
      "platform": "wechat",
      "platform_label": "微信号",
      "url": "alice_wx",
      "link_type": "text"
    }
  ]
}
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `links[].platform` | string | 平台标识（最多 30 字符） |
| `links[].platform_label` | string | 显示名称（最多 30 字符） |
| `links[].url` | string | 链接地址（最多 500 字符，自动净化） |
| `links[].link_type` | string | 类型：`link` 或 `text` |

---

#### PUT /api/user/music — 更新音乐设置

更新当前用户的音乐播放器设置。

- **认证**：是

**请求体**（JSON）：
```json
{
  "type": "netease",
  "music_url": "",
  "music_id": "123456",
  "title": "晴天",
  "artist": "周杰伦",
  "autoplay": 0
}
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `type` | string | 音乐类型（默认 `netease`） |
| `music_url` | string | 外部音乐 URL |
| `music_id` | string | 网易云音乐歌曲 ID |
| `title` | string | 歌曲标题 |
| `artist` | string | 艺术家 |
| `autoplay` | int | 是否自动播放 |

---

#### PUT /api/user/theme — 更新主题设置

更新当前用户的主题和强调色。

- **认证**：是

**请求体**（JSON）：
```json
{
  "theme_name": "ocean",
  "accent_color": "#0891b2",
  "custom_bg_url": "/uploads/backgrounds/1_1720000000.jpg"
}
```

---

#### POST /api/user/visit/{username} — 记录访问

记录用户主页的访问（5 分钟内同一 IP 不重复计数）。

- **认证**：否

**成功响应**：
```json
{ "message": "ok" }
```

---

### 8.3 发现模块

#### GET /api/discover — 获取发现页用户列表

获取所有公开用户列表，支持分页和多维排序。

- **认证**：否

**查询参数**：

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `page` | int | `1` | 页码（从 1 开始） |
| `sort` | string | `newest` | 排序方式：`newest`（最新注册）、`likes`（最多赞）、`visits`（最多访问） |

**成功响应**：
```json
{
  "users": [
    {
      "id": 1, "username": "alice", "nickname": "小明",
      "avatar_url": "...", "bio": "...", "theme_name": "sakura",
      "accent_color": "#ff6b9d", "custom_bg_url": "",
      "created_at": "2026-01-01 00:00:00",
      "like_count": 10, "visit_count": 128, "likeCount": 10, "visitCount": 128
    }
  ],
  "total": 50, "page": 1, "pageSize": 20,
  "pagination": { "currentPage": 1, "totalPages": 3, "total": 50 }
}
```

---

#### POST /api/discover/like/{username} — 点赞/取消点赞

对指定用户点赞或取消点赞（toggle 操作）。

- **认证**：否

**成功响应（点赞）**：
```json
{ "liked": true, "likeCount": 11 }
```

**成功响应（取消点赞）**：
```json
{ "liked": false, "likeCount": 10 }
```

---

#### GET /api/discover/like/{username}/check — 检查点赞状态

检查当前 IP 是否已点赞指定用户。

- **认证**：否

**成功响应**：
```json
{ "liked": true }
```

---

### 8.4 社交链接模块

社交链接通过 `PUT /api/user/social-links` 接口管理（见 8.2 用户模块），不设独立模块路由。

---

### 8.5 音乐模块

音乐设置通过 `PUT /api/user/music` 接口管理（见 8.2 用户模块），不设独立模块路由。

---

### 8.6 主题模块

主题设置通过 `PUT /api/user/theme` 接口管理（见 8.2 用户模块），不设独立模块路由。

---

### 8.7 上传模块

#### POST /api/upload/avatar — 上传头像

上传用户头像图片。

- **认证**：是
- **Content-Type**：`multipart/form-data`

**表单字段**：`file` 或 `avatar`（兼容多种字段名）

| 限制 | 值 |
|------|-----|
| 允许格式 | `jpg`、`jpeg`、`png`、`gif`、`webp` |
| 最大大小 | 5MB |
| 文件名规则 | `{userId}_{timestamp}.{ext}` |

**成功响应**：
```json
{ "url": "/uploads/avatars/1_1720000000.jpg", "message": "上传成功" }
```

---

#### POST /api/upload/background — 上传背景图

上传用户背景图片。

- **认证**：是
- **Content-Type**：`multipart/form-data`

**表单字段**：`file` 或 `background`

| 限制 | 值 |
|------|-----|
| 允许格式 | `jpg`、`jpeg`、`png`、`gif`、`webp` |
| 最大大小 | 10MB |
| 文件名规则 | `{userId}_{timestamp}.{ext}` |

**成功响应**：
```json
{ "url": "/uploads/backgrounds/1_1720000000.jpg", "message": "上传成功" }
```

---

#### GET /api/upload/qrcode/{username} — 获取二维码

生成指定用户主页的二维码图片（PNG 格式）。

- **认证**：否
- **响应类型**：`image/png`

二维码内容为 `{SITE_URL}/{username}`，通过 `api.qrserver.com` 外部服务生成。

---

### 8.8 管理员模块

所有管理员接口需要 `role=admin` 的 Token。

#### GET /api/admin/stats — 获取平台统计

- **认证**：管理员

**成功响应**：
```json
{
  "totalUsers": 100, "publicUsers": 80, "totalLikes": 500,
  "totalVisits": 10000, "adminCount": 2, "todayUsers": 5
}
```

---

#### GET /api/admin/users — 获取用户列表

分页获取所有用户，支持搜索和排序。

- **认证**：管理员

**查询参数**：

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `page` | int | `1` | 页码 |
| `search` | string | — | 搜索关键词（匹配用户名/昵称/邮箱，模糊搜索） |
| `sort` | string | `created_at` | 排序字段：`created_at`、`username`、`role`、`id` |
| `order` | string | `DESC` | 排序方向：`ASC` 或 `DESC` |

---

#### GET /api/admin/users/{id} — 获取用户详情

- **认证**：管理员
- **注意**：响应中不包含 `password_hash` 字段

---

#### PUT /api/admin/users/{id}/toggle-public — 切换公开状态

切换用户主页的公开/隐藏状态。

- **认证**：管理员

**成功响应**：
```json
{ "message": "更新成功", "is_public": 0 }
```

---

#### PUT /api/admin/users/{id}/role — 修改用户角色

- **认证**：管理员

**请求体**（JSON）：
```json
{ "role": "admin" }
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `role` | string | 角色值：`admin` 或 `user` |

---

#### DELETE /api/admin/users/{id} — 删除用户

删除指定用户及其所有关联数据（外键级联删除）。

- **认证**：管理员
- **限制**：不能删除自己

---

#### POST /api/admin/users/{id}/reset-password — 重置密码

为指定用户生成随机新密码。

- **认证**：管理员

**成功响应**：
```json
{ "message": "密码已重置", "newPassword": "a1b2c3d4" }
```

> `newPassword` 为 8 位随机十六进制字符串，由 `random_bytes(4)` 生成。

---

#### POST /api/admin/users/create — 添加用户

管理员直接创建新用户（无需邮箱验证码）。

- **认证**：管理员

**请求体**（JSON）：
```json
{
  "username": "newuser",
  "nickname": "新用户",
  "email": "new@example.com",
  "password": "123456",
  "role": "user"
}
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `username` | string | 是 | 用户名（2-20 位字母/数字/下划线/中文） |
| `nickname` | string | 是 | 昵称 |
| `email` | string | 是 | 邮箱（需符合格式，需唯一） |
| `password` | string | 是 | 密码（至少 6 位） |
| `role` | string | 否 | 角色：`user`（默认）或 `admin` |

**成功响应**：
```json
{
  "message": "用户添加成功",
  "user": {
    "id": 2, "username": "newuser", "nickname": "新用户",
    "email": "new@example.com", "role": "user"
  }
}
```

---

### 8.9 隐私与安全模块

#### PUT /api/user/password — 修改密码

修改当前登录用户的密码（需验证旧密码）。

- **认证**：是

**请求体**（JSON）：
```json
{
  "oldPassword": "old_pass",
  "newPassword": "new_pass"
}
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `oldPassword` / `old_password` | string | 是 | 旧密码（支持两种参数名） |
| `newPassword` / `new_password` | string | 是 | 新密码（6-100 位，支持两种参数名） |

**限制**：新密码不能与旧密码相同。

---

#### DELETE /api/user/account — 注销账户

永久删除当前用户及所有关联数据。

- **认证**：是

**请求体**（JSON）：
```json
{ "password": "current_password" }
```

需验证当前密码确认操作。删除后，`social_links`、`music_settings`、`likes`、`visit_stats` 通过外键级联自动清除。

---

#### GET /api/user/export — 导出个人数据

导出当前用户的所有数据为 JSON 文件下载（GDPR 数据可携带权）。

- **认证**：是
- **响应类型**：`application/json`，触发文件下载

**文件名格式**：`huiyun_data_{username}_{YYYYMMDD}.json`

**导出内容**：
```json
{
  "exported_at": "2026-07-25T12:00:00+08:00",
  "platform": "灰·韵 HUI·YUN",
  "user": { "id": 1, "username": "alice", ... },
  "social_links": [ ... ],
  "music_settings": { ... },
  "statistics": { "like_count": 10, "visit_count": 128 }
}
```

---

### 8.10 接口路由总表

| 方法 | 路径 | 认证 | 模块 | 说明 |
|------|------|------|------|------|
| POST | `/api/auth/send-code` | 否 | 认证 | 发送验证码 |
| POST | `/api/auth/get-code` | 否 | 认证 | 获取验证码（调试用） |
| POST | `/api/auth/register` | 否 | 认证 | 注册 |
| POST | `/api/auth/login` | 否 | 认证 | 密码登录 |
| POST | `/api/auth/login/send-code` | 否 | 认证 | 发送登录验证码 |
| POST | `/api/auth/login/verify` | 否 | 认证 | 验证码登录 |
| GET | `/api/auth/me` | 是 | 认证 | 获取当前用户信息 |
| GET | `/api/user/profile/{username}` | 否 | 用户 | 获取用户公开主页 |
| PUT | `/api/user/profile` | 是 | 用户 | 更新个人资料 |
| PUT | `/api/user/social-links` | 是 | 用户 | 更新社交链接 |
| PUT | `/api/user/music` | 是 | 用户 | 更新音乐设置 |
| PUT | `/api/user/theme` | 是 | 用户 | 更新主题设置 |
| POST | `/api/user/visit/{username}` | 否 | 用户 | 记录访问 |
| PUT | `/api/user/password` | 是 | 隐私 | 修改密码 |
| DELETE | `/api/user/account` | 是 | 隐私 | 注销账户 |
| GET | `/api/user/export` | 是 | 隐私 | 导出个人数据 |
| GET | `/api/discover` | 否 | 发现 | 发现页用户列表 |
| POST | `/api/discover/like/{username}` | 否 | 发现 | 点赞/取消点赞 |
| GET | `/api/discover/like/{username}/check` | 否 | 发现 | 检查点赞状态 |
| POST | `/api/upload/avatar` | 是 | 上传 | 上传头像 |
| POST | `/api/upload/background` | 是 | 上传 | 上传背景图 |
| GET | `/api/upload/qrcode/{username}` | 否 | 上传 | 获取二维码 |
| GET | `/api/admin/stats` | 管理员 | 管理 | 平台统计 |
| GET | `/api/admin/users` | 管理员 | 管理 | 用户列表 |
| GET | `/api/admin/users/{id}` | 管理员 | 管理 | 用户详情 |
| PUT | `/api/admin/users/{id}/toggle-public` | 管理员 | 管理 | 切换公开状态 |
| PUT | `/api/admin/users/{id}/role` | 管理员 | 管理 | 修改角色 |
| DELETE | `/api/admin/users/{id}` | 管理员 | 管理 | 删除用户 |
| POST | `/api/admin/users/{id}/reset-password` | 管理员 | 管理 | 重置密码 |
| POST | `/api/admin/users/create` | 管理员 | 管理 | 添加用户 |

---

## 九、主题系统

### 9.1 6 套预设主题

项目内置 6 套预设主题，通过 CSS Variables 实现，用户可在管理后台自由切换。

| 主题标识 | 名称 | 主背景色 | 强调色 | 风格描述 |
|---------|------|---------|--------|---------|
| `latte` | 拿铁 | `#faf8f5`（暖米白） | `#c4956a`（焦糖色） | 温暖米白色背景，柔和阴影，温馨咖啡馆氛围 |
| `slate` | 石板 | `#0f1117`（深灰） | `#22d3ee`（青色） | 冷灰色调深色背景，高对比度，科技感与专业感 |
| `forest` | 森林 | `#fafcf9`（浅绿） | `#16a34a`（绿色） | 自然绿色系，清新自然色调，森林与植物灵感 |
| `sunset` | 日落 | `#fefbf7`（暖橙） | `#ea580c`（橙色） | 暖橙红色调，渐变暖色背景，日落黄昏意境 |
| `ocean` | 海洋 | `#f7fafa`（浅青） | `#0891b2`（青蓝） | 深海蓝色调搭配波纹效果，清爽海洋风格 |
| `sakura` | 樱花 | `#fef8f9`（粉白） | `#e11d73`（玫红） | 樱花粉色搭配柔和渐变，浪漫樱花意境 |

### 9.2 主题变量体系

每个主题定义以下 CSS Variables：

| 变量 | 说明 |
|------|------|
| `--bg-primary` | 主背景色 |
| `--bg-secondary` | 次要背景色 |
| `--bg-card` | 卡片背景色 |
| `--bg-elevated` | 浮层背景色 |
| `--border` | 边框色 |
| `--border-subtle` | 次要边框色 |
| `--text-primary` | 主文字色 |
| `--text-secondary` | 次要文字色 |
| `--text-muted` | 弱化文字色 |
| `--accent` | 强调色（可被用户自定义覆盖） |
| `--accent-secondary` | 辅助强调色 |
| `--accent-bg` | 强调色背景（低透明度） |
| `--accent-bg-strong` | 强调色背景（较高透明度） |
| `--accent-border` | 强调色边框（低透明度） |
| `--success` / `--error` / `--info` | 状态色（成功/错误/信息） |
| `--shadow-sm` / `--shadow-md` / `--shadow-lg` | 阴影层级 |
| `--radius-sm` / `--radius` / `--radius-lg` | 圆角大小（8px / 12px / 16px） |
| `--navbar-bg` | 导航栏背景（85% 透明度） |
| `--overlay` | 遮罩层颜色 |
| `--qrcode-bg` | 二维码页面背景色 |

### 9.3 默认主题分配规则

新注册用户的默认主题根据注册时选择的性别自动分配：

| 性别 | 值 | 默认主题 | 说明 |
|------|-----|---------|------|
| 男 | `male` | `slate`（石板） | 深色科技感主题 |
| 女 | `female` | `latte`（拿铁） | 暖色温馨主题 |
| 保密 | `secret` | `latte`（拿铁） | 暖色温馨主题 |

> **注意**：默认主题分配由前端注册表单逻辑控制，后端注册接口的 `theme_name` 硬编码为 `sakura`。用户可在注册后通过管理后台或 API 切换到任意主题。

### 9.4 用户自定义

用户可通过 `PUT /api/user/theme` 接口自由修改：

1. **主题切换**：在 6 套预设主题间切换（修改 `theme_name`）
2. **强调色**：自定义任意十六进制颜色值（修改 `accent_color`）
3. **背景图**：上传自定义背景图片（修改 `custom_bg_url`）

---

## 十、安全措施

### 10.1 HTTP 安全响应头

由 `security.php` 中的 `setSecurityHeaders()` 函数在每个 PHP 入口文件开头统一设置：

| 响应头 | 值 | 作用 |
|--------|------|------|
| `X-Content-Type-Options` | `nosniff` | 防止浏览器 MIME 类型嗅探，避免将非文本文件当作脚本执行 |
| `X-Frame-Options` | `SAMEORIGIN` | 防止页面被其他网站通过 `<iframe>` 嵌入（点击劫持防护） |
| `X-XSS-Protection` | `1; mode=block` | 启用浏览器内置 XSS 过滤器（已弃用但仍有一定防御作用） |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | 强制 HTTPS 访问 1 年（仅在 HTTPS 连接时生效） |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | 跨域请求仅发送 Origin，不暴露完整 URL 路径 |
| `Permissions-Policy` | `geolocation=(), microphone=(), camera=(), payment=()` | 禁止浏览器 API（地理位置、麦克风、摄像头、支付）未经授权访问 |
| `Content-Security-Policy` | (见下) | 内容安全策略，限制页面可加载的资源来源 |

**CSP 策略详情**：

```
default-src 'self';
script-src 'self' 'unsafe-inline' https://music.163.com;
style-src 'self' 'unsafe-inline';
img-src 'self' data: https:;
media-src 'self' https:;
frame-src 'self' https://music.163.com;
font-src 'self' data:;
connect-src 'self' https://api.qrserver.com;
object-src 'none';
base-uri 'self';
```

| CSP 指令 | 允许来源 | 说明 |
|-----------|---------|------|
| `default-src` | `'self'` | 默认仅允许同源 |
| `script-src` | `'self' 'unsafe-inline' music.163.com` | 允许内联脚本（项目需要）和网易云音乐 |
| `style-src` | `'self' 'unsafe-inline'` | 允许内联样式 |
| `img-src` | `'self' data: https:` | 允许同源、Base64 图片、所有 HTTPS 图片 |
| `media-src` | `'self' https:` | 允许同源和 HTTPS 音频/视频 |
| `frame-src` | `'self' music.163.com` | 允许嵌入网易云音乐播放器 |
| `font-src` | `'self' data:` | 允许同源和 Base64 字体 |
| `connect-src` | `'self' api.qrserver.com` | 允许 API 请求和二维码服务 |
| `object-src` | `'none'` | 禁止 `<object>`/`<embed>` 标签 |
| `base-uri` | `'self'` | 限制 `<base>` 标签 |

### 10.2 IP 匿名化（SHA-256）

**实现位置**：`security.php` 中的 `anonymizeIP()` 和 `getAnonymizedIP()`

**算法**：`sha256(JWT_SECRET + IP + JWT_SECRET)`

**特性**：
- 使用 `JWT_SECRET` 作为盐值，确保不同站点即使相同 IP 也产生不同哈希
- 同一 IP 始终产生相同哈希，可用于去重但无法反推原始 IP
- 哈希结果为 64 字符十六进制字符串，存储在 `VARCHAR(64)` 字段中

**应用场景**：
- `likes.liked_by_ip`：点赞去重（唯一索引 `uk_user_ip`）
- `visit_stats.visitor_ip`：访问去重（5 分钟窗口，复合索引 `idx_visited`）

### 10.3 XSS 防护

| 措施 | 实现函数 | 说明 |
|------|---------|------|
| URL 净化 | `sanitizeUrl()` | 仅允许 `http`/`https`/`mailto`/`tel` 协议；拒绝 `javascript:`、`data:text/html`、`vbscript:`、`file:` 等危险协议；检测双重编码绕过 |
| 输出转义 | `h()` | 所有输出使用 `htmlspecialchars(ENT_QUOTES, UTF-8)` 转义 |
| 输入净化 | `sanitizeText()` | 移除控制字符（`\x00-\x08`、`\x0B`、`\x0C`、`\x0E-\x1F`、`\x7F`），限制最大长度 |
| 社交链接净化 | `sanitizeSocialLinks()` | 批量净化链接数组，截断平台名（30 字符）、标签（30 字符）、URL（500 字符） |
| 用户名过滤 | `sanitizeUsername()` | 仅保留字母、数字、下划线、中文字符，限制 20 字符 |

### 10.4 CORS 限制

**实现位置**：`api.php` 开头

- **白名单域名**：仅允许 `https://153900.xyz`、`https://www.153900.xyz` 跨域请求
- **同源请求**：无 `Origin` 头时，使用当前 `HTTP_HOST` 作为允许来源
- **允许方法**：`GET, POST, PUT, DELETE, OPTIONS`
- **允许请求头**：`Content-Type, Authorization, X-Authorization, X-HTTP-Authorization`
- **预检缓存**：86400 秒（24 小时）

### 10.5 速率限制

| 实现 | 说明 |
|------|------|
| `checkApiRateLimit()` | 基于 IP 匿名哈希的文件系统滑动窗口限流（`security.php`），默认每动作每 60 秒 30 次 |
| `checkRateLimit()` | 基于邮箱的验证码发送频率限制（`helpers.php`），每邮箱 60 秒 1 次 |
| `RATE_LIMIT_API` | API 通用限流配置：每 15 分钟 200 次（`config.php`） |
| `RATE_LIMIT_AUTH` | 认证接口限流配置：每 15 分钟 20 次（`config.php`） |

实现原理：使用 `sys_get_temp_dir()` 下的临时文件存储请求时间戳数组，通过 `LOCK_EX` 文件锁保证并发安全。

### 10.6 密码安全

- 使用 PHP `password_hash($password, PASSWORD_BCRYPT)` 加密
- bcrypt 算法自动加盐，每次生成不同哈希
- 使用 `password_verify()` 验证，内部使用恒定时间比较防止时序攻击
- 管理员重置密码使用 `random_bytes(4)` 生成 8 位随机十六进制密码
- 修改密码时验证旧密码，且不允许新旧密码相同

### 10.7 输入净化

- 所有用户文本输入（昵称、简介）通过 `sanitizeText()` 净化
- 所有 URL 输入（头像、背景图、社交链接）通过 `sanitizeUrl()` 净化
- 用户名通过正则 `/^[a-zA-Z0-9_\x{4e00}-\x{9fa5}]{2,20}$/u` 严格校验
- 邮箱通过 `filter_var($email, FILTER_VALIDATE_EMAIL)` 校验
- 性别字段通过白名单 `in_array($gender, ['male', 'female', 'secret'])` 校验

### 10.8 数据库安全

- **SQL 注入防护**：所有查询使用 PDO 预处理语句（Prepared Statements）
- `PDO::ATTR_EMULATE_PREPARES` 设为 `false`，使用数据库原生预处理
- **错误掩码**：`handleDBError()` 记录完整错误到 `error_log`，仅返回"服务器内部错误，请稍后重试"给用户

### 10.9 文件上传安全

| 措施 | 说明 |
|------|------|
| 类型白名单 | 仅允许 `jpg`、`jpeg`、`png`、`gif`、`webp` |
| 大小限制 | 头像 5MB，背景 10MB |
| 文件重命名 | `{userId}_{timestamp}.{ext}`，防止路径遍历和文件名注入 |
| 目录隔离 | 头像和背景分别存储在 `uploads/avatars/` 和 `uploads/backgrounds/` |
| PHP 执行禁止 | Nginx 配置中 `location /uploads/` 下的 `.php` 文件 `deny all` |
| 目录权限 | 上传目录 755 权限，PHP 自动创建（`mkdir($dir, 0755, true)`） |

---

## 十一、数据备份

### 11.1 数据库备份

```bash
# 完整备份（包含表结构和数据）
mysqldump -h localhost -u DB_USER -p'DB_PASS' DB_NAME > backup_$(date +%Y%m%d_%H%M%S).sql

# 仅备份数据（不含表结构）
mysqldump -h localhost -u DB_USER -p'DB_PASS' --no-create-info DB_NAME > data_$(date +%Y%m%d).sql

# 仅备份表结构
mysqldump -h localhost -u DB_USER -p'DB_PASS' --no-data DB_NAME > structure_$(date +%Y%m%d).sql
```

### 11.2 文件备份

```bash
# 备份用户上传文件
tar -czf uploads_backup_$(date +%Y%m%d).tar.gz /path/to/huiyun/uploads/

# 备份配置文件
tar -czf config_backup_$(date +%Y%m%d).tar.gz /path/to/huiyun/config.php /path/to/huiyun/seed.php
```

### 11.3 数据库恢复

```bash
# 从完整备份恢复
mysql -h localhost -u DB_USER -p'DB_PASS' DB_NAME < backup_20260725_120000.sql
```

> **注意**：恢复后无需手动重建表。`initDatabase()` 使用 `CREATE TABLE IF NOT EXISTS`，已有表不会被覆盖。

### 11.4 自动备份脚本

```bash
#!/bin/bash
# backup.sh — 自动备份脚本
# 建议通过 crontab 每日执行

BACKUP_DIR="/path/to/backups"
DB_HOST="localhost"
DB_USER="your_db_user"
DB_PASS="your_db_pass"
DB_NAME="your_db_name"
SITE_DIR="/path/to/huiyun"

DATE=$(date +%Y%m%d_%H%M%S)
mkdir -p "$BACKUP_DIR"

# 数据库备份
mysqldump -h "$DB_HOST" -u "$DB_USER" -p"$DB_PASS" "$DB_NAME" \
  > "$BACKUP_DIR/db_${DATE}.sql"

# 上传文件备份
tar -czf "$BACKUP_DIR/uploads_${DATE}.tar.gz" "$SITE_DIR/uploads/"

# 保留最近 30 天的备份
find "$BACKUP_DIR" -name "*.sql" -mtime +30 -delete
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +30 -delete

echo "[$DATE] 备份完成"
```

设置 crontab 每日凌晨 3:00 执行：

```bash
crontab -e
# 添加以下行
0 3 * * * /path/to/backup.sh >> /path/to/backups/backup.log 2>&1
```

> **虚拟主机环境**：可能不支持 crontab，请使用主机面板提供的「计划任务」功能替代。

---

## 十二、常见问题（FAQ）

### Q1: 访问首页显示 500 错误

**排查步骤**：

1. 检查 `config.php` 中的数据库配置（`DB_HOST`、`DB_NAME`、`DB_USER`、`DB_PASS`）是否正确
2. 确认数据库已创建且用户有权限（在主机面板检查）
3. 检查 PHP 版本：`php -v`，需 >= 7.4
4. 检查 PHP 扩展：`php -m | grep -iE 'pdo|openssl|mbstring|json|curl'`
5. 查看服务器错误日志中的 `[DB Error]` 信息

### Q2: API 请求返回 404

**排查步骤**：

1. 确认 Nginx 伪静态规则已正确配置（第五章）
2. 检查 `api.php` 文件是否存在
3. 直接访问 `https://your-domain.com/api.php?path=discover` 测试（绕过伪静态）
4. 检查 URL 格式是否正确（必须以 `/api/` 开头）

### Q3: 登录后 Token 无效 / 401 错误

**排查步骤**：

1. 确认 `JWT_SECRET` 已设置且未在部署后被修改（修改密钥会导致所有旧 Token 失效）
2. 检查 `JWT_EXPIRES` 是否设置合理（默认 30 天）
3. 虚拟主机可能不传递 `Authorization` 头：前端需使用备用头 `X-Authorization` 或 `X-HTTP-Authorization`
4. 重新登录获取新 Token

### Q4: 邮件发送失败 / 收不到验证码

**排查步骤**：

1. 检查 SMTP 配置（`SMTP_HOST`、`SMTP_PORT`、`SMTP_USER`、`SMTP_PASS`）
2. 确认 Resend API Key 有效且未过期
3. 确认发件域名已在 Resend 控制台中验证通过
4. 查看服务器错误日志中的 `[SMTP]`、`[Resend]`、`[mail]` 错误信息
5. 系统会自动降级到 PHP `mail()`；如果全部失败，验证码会记录到 `error_log`
6. 开发环境可使用 `POST /api/auth/get-code` 接口直接获取验证码

### Q5: 文件上传失败

**排查步骤**：

1. 检查 `uploads/` 及子目录权限（需 755 或 775）
2. 检查文件大小是否超限（头像 5MB、背景 10MB）
3. 检查文件格式（仅支持 `jpg`/`jpeg`/`png`/`gif`/`webp`）
4. 检查 PHP 配置：`upload_max_filesize`（建议 >= 10M）和 `post_max_size`（建议 >= 12M）
5. 检查 PHP `temp` 目录是否可写

### Q6: 用户主页 `/username` 显示 404

**排查步骤**：

1. 确认 Nginx 伪静态规则中 `location /` 规则已生效
2. 使用备用入口测试：`https://your-domain.com/u.php?u=username`
3. 确认用户已注册且 `is_public = 1`（管理员可在后台切换）
4. 确认用户名中不包含特殊字符（仅支持字母/数字/下划线/中文）

### Q7: 伪静态规则不生效

**排查步骤**：

1. 确认虚拟主机支持 Nginx 自定义配置
2. 联系主机客服，让其手动添加规则到 Nginx 配置
3. 使用备用方案直接通过参数 URL 访问：
   - 页面：`/p.php?page=login`
   - 用户主页：`/u.php?u=username`
   - API：`/api.php?path=discover`

### Q8: 管理员账号无法登录

**排查步骤**：

1. 访问 `/seed.php` 检查管理员账号状态
2. 在 `seed.php` 页面点击「重置密码」
3. 确认管理员 `role` 字段为 `admin`（`index.php` 和 `p.php` 的 `seedAdmin()`/`seedAdminAuto()` 会自动修正）
4. 检查 `config.php` 中 `ADMIN_USERNAME` 和 `ADMIN_PASSWORD` 是否正确

### Q9: PUT/DELETE 请求返回 405 Method Not Allowed

**原因**：部分虚拟主机的 Nginx 不支持 `PUT`/`DELETE` 方法。

**解决方案**：使用 `_method` 参数覆盖：

```bash
# 替代 PUT /api/user/profile
POST /api/user/profile?_method=PUT

# 替代 DELETE /api/user/account
POST /api/user/account?_method=DELETE
```

`_method` 参数支持从 `$_GET` 或 `$_POST` 中读取。前端代码中已实现此兼容机制。

### Q10: 部署后需要清理的文件

| 操作 | 文件 | 原因 |
|------|------|------|
| **删除或重命名** | `seed.php` | 防止未授权用户访问管理员管理页面 |
| **删除**（可选） | `nginx-rewrite.txt` | 仅参考文件，无运行时作用 |
| **立即修改** | `config.php` 中的 `ADMIN_PASSWORD` | 默认密码存储在代码中，存在安全风险 |
| **立即修改** | `config.php` 中的 `JWT_SECRET` | 使用 `openssl rand -hex 32` 生成随机密钥 |

---

## 十三、更新日志

### v2.0.0 — 安全增强与隐私保护

- 新增 IP 匿名化存储（SHA-256 哈希 + JWT_SECRET 盐值，不记录原始 IP）
- 新增完整 CSP 内容安全策略（允许网易云音乐嵌入和内联代码）
- 新增 HSTS 强制 HTTPS（max-age=31536000，包含子域名）
- 新增 `X-Content-Type-Options`、`X-Frame-Options`、`Referrer-Policy`、`Permissions-Policy` 等安全头
- 新增 XSS URL 净化（`sanitizeUrl`，协议白名单 + 双重编码检测）
- 新增输入净化函数（`sanitizeText`、`sanitizeUsername`）
- 新增速率限制（基于文件系统的滑动窗口，`checkApiRateLimit`）
- 新增数据库错误掩码（`handleDBError`，不泄露数据库结构）
- CORS 收紧为指定域名白名单
- 所有数据库查询使用原生预处理语句（`PDO::ATTR_EMULATE_PREPARES = false`）

### v2.1.0 — 账户安全功能

- 新增修改密码接口（`PUT /api/user/password`，需验证旧密码）
- 新增注销账户接口（`DELETE /api/user/account`，需验证密码，级联删除所有数据）
- 新增数据导出接口（`GET /api/user/export`，GDPR 数据可携带权，JSON 文件下载）
- 新增管理员添加用户接口（`POST /api/admin/users/create`，无需验证码）
- 新增 `gender` 字段（`VARCHAR(10) DEFAULT 'secret'`，支持 `male`/`female`/`secret`）
- 新增社交链接 `link_type` 字段（支持 `link` 链接和 `text` 纯文本两种类型）
- 自动迁移兼容旧数据库（`ALTER TABLE` 带 try-catch）

### v2.2.0 — 邮件与部署优化

- 邮件发送三级降级机制（SMTP 直连 → Resend API → PHP mail()）
- SMTP 直连实现（不依赖 PHP 扩展，使用 `stream_socket_client`，兼容虚拟主机）
- `seed.php` 改为独立管理员管理页面（创建/重置/删除功能）
- Nginx 伪静态规则完善（API 路由 + 二维码页面 + 静态文件分离）
- `_method` 参数覆盖支持（兼容不支持 PUT/DELETE 的虚拟主机）
- `Authorization` 头多重备选读取（`Authorization` / `X-Authorization` / `X-HTTP-Authorization` / `REDIRECT_HTTP_AUTHORIZATION`）
- 首次访问自动建表与迁移（`initDatabase()` 在 `index.php` 和 `p.php` 中自动调用）

---

> 灰·韵 HUI·YUN · 樱花别院
> 最后更新：2026-07-25
