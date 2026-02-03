# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是 MoviePilot 插件的一个分支版本,专门用于企业微信动态IP自动配置。包含两个版本:
- **WeWorkIP** (`plugins/weworkip/`): 使用 Selenium,适合 Windows 用户
- **WeWorkIPPW** (`plugins/weworkippw/`): 使用 Playwright,适合 Docker 用户

两个插件的核心功能相同:自动检测动态公网IP并配置到企业微信应用的可信IP列表中。

## 架构设计

### 插件结构

每个插件都是一个独立的 Python 包,包含:
- `__init__.py`: 主插件类(继承自 `_PluginBase`)
- `requirements.txt`: Python 依赖

### 核心流程

1. **初始化** (`init_plugin`): 加载配置、设置定时任务
2. **IP检测** (`check` → `CheckIP` → `get_ip_from_url`): 从多个备用URL获取公网IP
3. **IP变更** (`ChangeIP`): 使用浏览器自动化(Selenium/Playwright)登录企业微信并修改可信IP
4. **Cookie管理**:
   - `get_cookie()`: 优先从 CookieCloud 获取,其次使用手动配置
   - `refresh_cookie()`: 定时刷新cookie以保持有效
   - `login()`: Cookie失效时执行自动登录流程(扫码/验证码)

### 关键状态变量

- `_cookie_valid`: Cookie是否有效(影响是否执行IP检测)
- `_ip_changed`: IP是否已成功更新到企业微信(防止重复更新)
- `_schedule_login`: 是否启用自动登录(cookie失效时自动重试)

### 浏览器自动化差异

**Selenium 版** (`weworkip`):
- 支持 Edge/Chrome 切换
- 支持新旧无头模式切换
- 依赖本地浏览器驱动
- 代码位置: `plugins/weworkip/__init__.py:276-348`

**Playwright 版** (`weworkippw`):
- 使用 Chromium
- 仅支持无头模式
- 更适合 Docker 环境
- 代码位置: `plugins/weworkippw/__init__.py:254-302`

## 配置文件

### package.json

定义插件的元数据和版本信息:
- `WeWorkIP`: Selenium 版本(当前 v2.5)
- `WeWorkIPPW`: Playwright 版本(当前 v2.4.2)

修改插件版本时需要同步更新:
1. `package.json` 中的 `version` 和 `history`
2. `__init__.py` 中的 `plugin_version`

## 开发指南

### 添加新的IP检测源

在 `_ip_urls` 列表中添加新URL(两个插件都需要修改):

```python
_ip_urls = [
    "https://myip.ipip.net",
    "https://ddns.oray.com/checkip",
    "https://ip.3322.net",
    "https://4.ipw.cn",
    # 添加新URL
]
```

确保响应内容中包含IP地址(由 `_ip_pattern` 正则匹配)。

### 修改定时任务

所有定时任务在 `init_plugin` 中配置:
- `_check_cron`: IP检测周期(默认 `*/11 * * * *`)
- `_status_cron`: Cookie失效通知周期(默认 `0 * * * *`)
- `_refresh_cron`: Cookie刷新周期(默认 `*/5 * * * *`)

### 企业微信选择器变更

如果企业微信更新页面结构,需要修改以下XPath/CSS选择器:

**登录检测**:
- Selenium: `By.CLASS_NAME, "login_stage_title_text"` (`__init__.py:307`)
- Playwright: `.login_stage_title_text` (`__init__.py:273`)

**IP配置入口**:
- Selenium: `//div[contains(@class, "app_card_operate") and contains(@class, "js_show_ipConfig_dialog")]` (`__init__.py:325`)
- Playwright: `div.app_card_operate.js_show_ipConfig_dialog` (`__init__.py:286`)

**IP输入框**:
- Selenium: `//textarea[@class="js_ipConfig_textarea"]` (`__init__.py:329`)
- Playwright: `textarea.js_ipConfig_textarea` (`__init__.py:289`)

**确认按钮**:
- Selenium: `//a[@class="qui_btn ww_btn ww_btn_Blue js_ipConfig_confirmBtn"]` (`__init__.py:331`)
- Playwright: `.js_ipConfig_confirmBtn` (`__init__.py:290`)

### CookieCloud 配置

插件依赖 MoviePilot 内置的 `CookieCloudHelper`:
- 域名: `.work.weixin.qq.com`
- Cookie 格式: `name=value; name2=value2`

## 调试技巧

### 查看日志

插件使用 `app.log.logger`,日志输出到 MoviePilot 主日志。

关键日志点:
- `开始检测公网IP`: IP检测开始
- `检测到IP变化`: 发现新IP
- `cookie有效校验成功` / `cookie失效`: Cookie状态
- `更改可信IP成功`: 配置成功

### 本地测试

1. 确保已安装依赖: `pip install -r plugins/weworkip/requirements.txt`
2. 需要运行在 MoviePilot 环境中(依赖 `app.*` 模块)
3. 浏览器驱动需要预安装(ChromeDriver/EdgeDriver)

### 常见问题

**浏览器启动失败**:
- Selenium 版: 尝试切换 `_browser` 配置(edge/chrome)或 `_use_old_headless`
- Playwright 版: 确保 Playwright Chromium 已安装(`playwright install chromium`)

**二维码登录失败**:
- 检查 `_qr_send_users` 配置(企业微信成员ID)
- 确认微信通知代理地址为 `https://qyapi.weixin.qq.com/`

**Cookie频繁失效**:
- 减小 `_refresh_cron` 间隔(默认5分钟)
- 检查网络连接稳定性(`check_connect` 方法)

## 用户交互

### 通过 MoviePilot 应用发送命令

用户可以在 MoviePilot 微信应用中发送:
- `#登录企业微信`: 手动触发登录流程
- `#123456`: 提供登录验证码(格式: `#` + 6位数字)

这些命令通过 `EventType.UserMessage` 事件接收(`receive_message` 方法)。

### 插件状态显示

`get_page()` 方法返回插件状态页面:
- 显示 Cookie 有效状态(绿色=有效,红色=失效,黄色=未启用)
- Cookie失效时显示登录二维码(备用登录方式)
