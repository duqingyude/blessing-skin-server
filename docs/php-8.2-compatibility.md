# PHP 8.2（8.2.35）适配说明

本文记录将本项目（Blessing Skin Server 6.0.3）改造为支持 PHP 8.2.35 的原因、改动内容和部署验证步骤。

- 兼容范围：`PHP >= 8.0.2 且 < 8.3`
- 重点验证版本：`PHP 8.2.35`
- 不在范围内：PHP 8.3 及以上（尚未验证，启动检查会直接拦截）

---

## 1. 原项目在 PHP 8.2 上会遇到什么

### 1.1 安装阶段：Composer 直接拒绝安装

改造前 `composer.lock` 中锁定的部分依赖带有 PHP 版本上限，在 PHP 8.2 上执行 `composer install` 会解析失败：

| 依赖 | 改造前版本 | 版本约束 | 结果 |
| --- | --- | --- | --- |
| `nette/schema` | 1.2.2 | `>=7.1 <8.2` | 不满足 PHP 8.2 |
| `nette/utils` | 3.2.6 | `>=7.2 <8.2` | 不满足 PHP 8.2 |
| `phpspec/prophecy`（开发依赖） | 1.15.0 | `~8.0, <8.2` | 原为 PHPUnit 9.5 的间接依赖；升级到 PHPUnit 9.6 后已不再引入 |

这三个包分别由 Laravel 的 Markdown/配置组件和 PHPUnit 间接引入，属于“硬约束”，必须升级或固定到兼容版本，不能只改根 `composer.json` 的 `php` 字段。

### 1.2 运行阶段：Laravel 8 早期版本与 PHP 8.1/8.2 弃用项

- PHP 8.1 起，向内置函数的非空字符串参数传 `null` 会产生 `E_DEPRECATED`。
- PHP 8.2 起，动态属性、字符串插值 `${...}`、`utf8_encode/decode` 等会产生 `E_DEPRECATED`。
- Laravel 8.83 的 `HandleExceptions` 会把 `E_DEPRECATED` 写入 `deprecations` 日志通道，而不是抛出异常中断请求；8.75 及更早版本没有这层保护。因此框架需要升级到 8.83.x。

本项目自身代码中实际受影响的调用已在第 3 节列出并修复。

---

## 2. 改造后的关键依赖版本

`composer.lock` 已重新解析，主要版本变化如下：

| 依赖 | 改造前 | 改造后 | 说明 |
| --- | --- | --- | --- |
| `laravel/framework` | 8.75.0 | 8.83.29 | Laravel 8 最后一个补丁版本，包含 PHP 8.1/8.2 修复与弃用日志处理 |
| `laravel/passport` | 10.2.2 | 10.4.2 | 与 Laravel 8.83 配套 |
| `symfony/*` | 5.4.0 / 5.4.1 | 5.4.54 | Symfony 5.4 后期补丁才完整支持 PHP 8.2 |
| `doctrine/dbal` | 2.13.6 | 2.13.9 | 2.x 最后补丁 |
| `nesbot/carbon` | 2.55.2 | 2.73.0 | 支持 PHP 8.2 |
| `twig/twig` | 2.14.8 | 2.16.1 | Twig 2 最后版本，修复 PHP 8.2 弃用项 |
| `guzzlehttp/guzzle` | 7.4.1 | 7.16.0 | 7.x 最新补丁 |
| `intervention/image` | 2.7.0 | 2.7.2 | 2.x 最后补丁 |
| `facade/ignition` | 2.17.2 | 2.17.7 | 2.x 最后补丁 |
| `phpunit/phpunit` | 9.5.10 | 9.6.38 | 9.6 系列支持 PHP 8.2，且不再依赖 `prophecy` |
| `nette/schema` | 1.2.2 | 1.2.5 | 支持 PHP 7.1 - 8.3 |
| `nette/utils` | 3.2.6 | 3.2.10 | 支持 PHP 7.2 - 8.3 |

> `nette/schema` 和 `nette/utils` 在 `composer.json` 中被显式固定到 `~1.2.5` / `~3.2.10`。它们是间接依赖，固定版本是为了避免以后执行 `composer update` 时又被解析到只支持 PHP 8.1+ 或 8.2+ 的新版本，从而丢掉 PHP 8.0/8.1 兼容性。

---

## 3. 本次代码与配置改动

### 3.1 `composer.json`

- PHP 约束由 `>=8.0.2` 改为 `>=8.0.2 <8.3`。
- 升级上述依赖的版本约束。
- 新增 `nette/schema`、`nette/utils` 两个兼容性固定项。
- 新增 `config.platform.php = 8.0.2`。

`config.platform.php` 的作用是让 Composer 始终按 PHP 8.0.2 这个最低支持版本解析依赖。这样在 PHP 8.2.35 机器上执行 `composer install/update` 时，Composer 不会把间接依赖升级到只支持 PHP 8.1+ 或 8.2+ 的新版本（例如 `brick/math`、`symfony/string`、`symfony/translation`、`dragonmantank/cron-expression` 的新版本），从而同时保住 PHP 8.0/8.1/8.2 三条线。

本次生成的 `composer.lock` 已用 Composer Semver 逐包校验：所有包的 `php` 约束同时满足 `8.0.2` 和 `8.2.35`。

### 3.2 应用代码（PHP 8.1+ 空值传参弃用）

| 文件 | 改动 |
| --- | --- |
| [app/Services/Option.php](../app/Services/Option.php) | `strtolower($value)` → `strtolower((string) $value)`，避免配置值为 `null` 时触发弃用 |
| [app/Services/PluginManager.php](../app/Services/PluginManager.php) | `json_decode($option)` → `json_decode((string) $option)` |
| [app/Http/Middleware/EnforceEverGreen.php](../app/Http/Middleware/EnforceEverGreen.php) | 请求没有 `User-Agent` 时 `userAgent()` 返回 `null`，先转为字符串再做 `preg_match` |
| [app/Rules/PlayerName.php](../app/Rules/PlayerName.php) | 校验值统一转为字符串后再做 `preg_match` / `mb_check_encoding` |
| [app/Http/Controllers/MarketController.php](../app/Http/Controllers/MarketController.php) | `config('plugins.registry')` 转为字符串后再 `explode` |
| [app/Http/Controllers/UserController.php](../app/Http/Controllers/UserController.php) | `option('sign_score')` 转为字符串后再 `explode` |
| [app/Http/Controllers/OptionsController.php](../app/Http/Controllers/OptionsController.php) | 同上，补全签到分数表单的默认值解析 |
| [app/Exceptions/Handler.php](../app/Exceptions/Handler.php) | 异常堆栈中的 `file` 字段先转字符串再 `str_replace` |

### 3.3 运行环境与工程配置

| 文件 | 改动 |
| --- | --- |
| [.github/workflows/CI.yml](../.github/workflows/CI.yml) | Lint 任务改用 PHP 8.2；测试矩阵新增 `8.2`；构建任务显式安装 PHP 8.2，避免 runner 默认 PHP 8.3+ 与 `<8.3` 约束冲突 |
| [Dockerfile](../Dockerfile) | 运行时镜像 `php:8-apache` → `php:8.2-apache` |
| [.devcontainer/Dockerfile](../.devcontainer/Dockerfile) | 默认 `VARIANT` 改为 `8.2-bullseye` |
| [.devcontainer/docker-compose.yml](../.devcontainer/docker-compose.yml) | 开发容器 PHP 版本改为 `8.2-bullseye` |
| [bootstrap/chkenv.php](../bootstrap/chkenv.php) | 增加 PHP 上限检查，`>= 8.3.0` 时给出明确提示并停止 |
| [README.md](../README.md) / [README-zh.md](../README-zh.md) | 环境要求更新为 `>= 8.0.2 且 < 8.3` |
| `.github/ISSUE_TEMPLATE/*.yml` | PHP 版本下拉新增 `8.2` |

---

## 4. 部署与升级步骤

### 4.1 全新部署（源码包）

1. 安装 PHP 8.2.35，并启用以下扩展：

   ```text
   openssl, pdo, mbstring, tokenizer, gd, xml, ctype, json, fileinfo, zip
   ```

   - 使用 SQLite 时还需要 `pdo_sqlite`。
   - 不要禁用 `symlink`、`readlink`、`putenv`、`realpath`。
   - 不要开启 `open_basedir`。

2. 安装依赖：

   ```bash
   composer install --no-dev --optimize-autoloader
   ```

   > Composer 2.10 及以上版本在执行 `composer update`（重新解析依赖）时会拦截存在安全公告的包。本项目仍使用 Laravel 8 / Twig 2 等已停止维护的组件，`composer audit` 会报告公告。`composer install` 从现有 `composer.lock` 安装不受影响；只有在确实需要刷新锁文件时才使用：
   >
   > ```bash
   > composer update --no-blocking
   > ```
   >
   > 该参数只跳过“解析阻断”，不会修复漏洞。生产环境请评估第 6 节的风险。

3. 初始化配置：

   ```bash
   cp .env.example .env
   php artisan key:generate
   ```

4. 确保 `bootstrap/cache`、`storage`、`plugins`、`public` 可写。

5. 建表并生成缓存：

   ```bash
   php artisan migrate --force
   php artisan optimize
   ```

6. Web 服务器站点根目录指向 `public/`，并开启 URL 重写。

### 4.2 已有站点升级

1. 备份数据库、`storage/`、`.env` 和当前代码目录。
2. 停止写入（维护模式）：

   ```bash
   php artisan down
   ```

3. 替换代码后安装依赖：

   ```bash
   composer install --no-dev --optimize-autoloader
   php artisan migrate --force
   php artisan optimize:clear
   php artisan optimize
   ```

4. 恢复服务：

   ```bash
   php artisan up
   ```

5. 首次启动后检查日志，确认没有 `Fatal error`；`Deprecated` 级别的记录属于预期现象，见第 5 节。

### 4.3 Docker

```bash
docker build -t blessing-skin:php82 .
docker run -d -p 8080:80 -v "$PWD/storage:/app/storage" blessing-skin:php82
```

---

## 5. 验证清单

在项目根目录执行：

```bash
# 1. 版本
php -v                      # 应为 8.2.35
composer validate
# 注意：由于 config.platform.php = 8.0.2，check-platform-reqs 显示的是解析平台，
# 实际运行时版本仍以 php -v 与 php artisan --version 为准
composer check-platform-reqs

# 2. 依赖与自动加载
composer install --dry-run

# 3. 框架启动
php artisan --version
php artisan twig:lint -v

# 4. 测试（需要 sqlite 扩展，CI 已覆盖 8.0 / 8.1 / 8.2）
./vendor/bin/phpunit
```

手工验收建议覆盖：

- 首页、登录、注册（含验证码）
- 上传皮肤/披风，皮肤库列表与详情
- 角色创建、改名、删除
- 管理后台选项保存
- 插件启用/禁用
- 邮件发送（注册验证、找回密码）

关于弃用日志：Laravel 8.83 会把 `E_DEPRECATED` 记录到 `deprecations` 日志通道（默认丢弃），因此第三方库（如已停止维护的 Swiftmailer）产生的弃用不会中断请求。若将日志通道改为 `stack`/`single`，可在 `storage/logs/laravel.log` 中看到 `Deprecated:` 行；只有出现 `Fatal error` 或功能异常时才需要继续处理。

---

## 6. 兼容性边界与风险

1. **Laravel 8 / Symfony 5.4 / Twig 2 / DBAL 2 / Swiftmailer 均已停止维护**，`composer audit` 会报告安全公告。本次改造只解决“能在 PHP 8.2.35 上安装和运行”，不等于消除安全风险。后续建议单独立项升级到 Laravel 10/11、Twig 3、Passport 12，并替换 Swiftmailer 为 Symfony Mailer。
2. **PHP 8.3+ 未验证**：`composer.json` 与 `bootstrap/chkenv.php` 都已限制 `< 8.3`，避免在未测试的环境中出现动态属性等问题。
3. **插件兼容性**：插件 `package.json` 的 `require.php` 会参与启用检查。若插件声明了 `"php": ">=8.0 <8.2"` 之类的约束，升级到 8.2 后会被自动禁用。需要联系插件作者或修改插件清单后重新启用。
4. **Composer 2.10+ 安全阻断**：见 4.1 节说明；该策略只影响 `composer update`，不影响按锁文件 `composer install`。若确需刷新锁文件，可使用 `--no-blocking` 临时绕过，但应记录原因并尽快规划依赖升级。
5. **PHP 8.0/8.1 仍可用**：本次没有把最低版本提高到 8.2。`config.platform.php` 与 `nette/schema`、`nette/utils` 的固定项共同保证锁文件不会被解析到更高 PHP 要求的版本。

---

## 7. 回退方案

如果升级后出现无法定位的兼容问题：

1. 恢复备份的代码目录与数据库。
2. 恢复旧版 `composer.lock` 后执行：

   ```bash
   composer install --no-dev --optimize-autoloader
   php artisan optimize:clear
   ```

3. 将 PHP 切回 8.1，并重新生成缓存。

回退不会修改数据库结构（本次改造没有数据库迁移变更）。
