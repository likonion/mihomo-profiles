# OpenClash 私有机场订阅配置

本方案用于实现：

* GitHub 仓库使用 **Public**，公开保存 Mihomo/OpenClash 配置
* 机场真实订阅地址不写入 GitHub
* OpenClash 通过本地 `config_overwrite` 将真实机场地址注入配置
* Provider 名称统一使用 `airport`
* 机场订阅地址仅保存在 OpenWrt 本机 `/etc/config/openclash`
* 不需要 SubConverter 或额外 Docker 服务

## 一、目录结构

GitHub 仓库：

```text
mihomo-profiles/
├── config.yaml
└── overwrite/
    └── airport
```

其中：

* `config.yaml`：公开的 Mihomo 主配置
* `overwrite/airport`：OpenClash Overwrite 脚本

### `config.yaml`

Provider 必须使用统一名称：

```yaml
proxy-providers:
  airport:
    url: "机场订阅地址占位符"
    type: http
    interval: 86400
    health-check:
      enable: true
      url: https://www.gstatic.com/generate_204
      interval: 180
    proxy: 直连
```

这里的 URL 可以使用占位地址，不要放真实机场订阅地址。

### `overwrite/airport`

内容：

```ini
[Overwrite]

ruby_map_edit "$CONFIG_FILE" "['proxy-providers']" "airport" "['url']" "$EN_KEY1"
```

该脚本的作用是：

> 将最终配置中的 `proxy-providers.airport.url` 替换为 OpenWrt 本地提供的 `$EN_KEY1`。

---

# 二、OpenWrt 配置

## 1. 创建 OpenClash Overwrite 配置

+ 方式一： 在 OpenWrt SSH 中执行：

  ```bash
  uci add openclash config_overwrite
  ```

  然后配置：

  ```bash
  uci set openclash.@config_overwrite[-1].name='airport'
  uci set openclash.@config_overwrite[-1].enable='1'
  uci set openclash.@config_overwrite[-1].type='http'
  uci set openclash.@config_overwrite[-1].url='https://raw.githubusercontent.com/你的用户名/你的仓库/main/overwrite/airport'
  uci set openclash.@config_overwrite[-1].update_days='off'
  uci set openclash.@config_overwrite[-1].update_hour='off'
  uci set openclash.@config_overwrite[-1].order='100'
  uci add_list openclash.@config_overwrite[-1].config='all'
  uci set openclash.@config_overwrite[-1].param='EN_KEY1=你的真实机场订阅URL'
  ```
+ 方式二：直接编辑 `vi /etc/config/openclash`
  添加配置：
  ```bash
  config config_overwrite
      option name 'airport'
      option enable '1'
      option type 'http'
      option url 'https://raw.githubusercontent.com/你的用户名/你的仓库/main/overwrite/airport'
      option update_days 'off'
      option update_hour 'off'
      option order '100'
      list config 'all'
      option param 'EN_KEY1=你的真实机场订阅URL'
  ```


# 三、确认 OpenClash 配置

查看 Overwrite 配置：

```bash
uci show openclash | grep -A12 "config_overwrite"
```

正常情况下应该看到类似：

```text
openclash.@config_overwrite[1]=config_overwrite
openclash.@config_overwrite[1].name='airport'
openclash.@config_overwrite[1].enable='1'
openclash.@config_overwrite[1].type='http'
openclash.@config_overwrite[1].url='https://raw.githubusercontent.com/.../overwrite/airport'
openclash.@config_overwrite[1].update_days='off'
openclash.@config_overwrite[1].update_hour='off'
openclash.@config_overwrite[1].order='100'
openclash.@config_overwrite[1].config='all'
openclash.@config_overwrite[1].param='EN_KEY1=https://...'
```

检查真实订阅地址：

```bash
uci get openclash.@config_overwrite[1].param
```

应该输出：

```text
EN_KEY1=https://你的真实机场订阅URL
```

---

# 四、下载 Overwrite 文件

OpenClash 的 `config_overwrite` 配置中虽然指定了远程 URL，但实际使用时需要确保对应的 Overwrite 文件已经存在于：

```text
/etc/openclash/overwrite/
```

可以手动下载：

```bash
mkdir -p /etc/openclash/overwrite

curl -L \
'https://raw.githubusercontent.com/你的用户名/你的仓库/main/overwrite/airport' \
-o /etc/openclash/overwrite/airport
```

确认：

```bash
cat /etc/openclash/overwrite/airport
```

应该看到：

```ini
[Overwrite]

ruby_map_edit "$CONFIG_FILE" "['proxy-providers']" "airport" "['url']" "$EN_KEY1"
```

---

# 五、重启 OpenClash

配置完成后：

```bash
/etc/init.d/openclash restart
```

查看 OpenClash 日志：

```bash
tail -100 /tmp/openclash.log
```

或者：

```bash
grep -Ei 'overwrite|airport|EN_KEY1' /tmp/openclash.log
```

正常情况下可以看到类似：

```text
执行覆写模块【airport】
加载覆写命令【Ruby Script => ruby_map_edit "$CONFIG_FILE" "['proxy-providers']" "airport" "['url']" "$EN_KEY1"】
```

这说明 `airport` Overwrite 已经被执行。

---

# 六、验证最终配置

OpenClash 生成配置后，可以检查实际 YAML：

```bash
grep -n -A8 -B2 "airport:" /etc/openclash/config/*.yaml
```

正常情况下应该看到：

```yaml
proxy-providers:
  airport:
    url: "https://你的真实机场订阅URL"
    type: http
    interval: 86400
    health-check:
      enable: true
      url: https://www.gstatic.com/generate_204
      interval: 180
    proxy: 直连
```

此时说明：

```text
GitHub 公共 config.yaml
        │
        │ 下载
        ▼
OpenClash
        │
        │ 加载 overwrite/airport
        ▼
ruby_map_edit
        │
        │ $EN_KEY1
        ▼
本地真实机场订阅 URL
        │
        ▼
proxy-providers.airport
```

---

# 七、Provider 名称必须保持一致

整个方案中有三个地方必须统一使用：

```text
airport
```

### 1. 主配置

```yaml
proxy-providers:
  airport:
```

### 2. Overwrite 文件

```bash
ruby_map_edit "$CONFIG_FILE" "['proxy-providers']" "airport" "['url']" "$EN_KEY1"
```

### 3. OpenClash Overwrite 名称

```text
option name 'airport'
```

不要混用：

```text
vincent airport
vincent-airport
airport
```

其中：

* `airport`：Provider 名称
* `airport`：OpenClash Overwrite 名称
* `overwrite/airport`：GitHub 上的 Overwrite 文件

三者统一后最不容易出错。


# 八、安全注意事项

不要将真实机场订阅 URL 写入：

* `config.yaml`
* `overwrite/airport`
* GitHub README
* GitHub Commit
* GitHub Issue

真实订阅 URL 只应该存在 OpenWrt 本地：

```text
/etc/config/openclash
```

其中：

```text
option param 'EN_KEY1=真实机场订阅URL'
```

属于私有配置。

如果机场订阅 URL 曾经被提交到公开 GitHub，应立即更换机场订阅链接对应的 Token/路径。

---

# 九、最终效果

最终实现：

```text
                 GitHub Public
                       │
             ┌─────────┴─────────┐
             │                   │
       config.yaml        overwrite/airport
             │                   │
             └─────────┬─────────┘
                       │
                       ▼
                  OpenClash
                       │
              config_overwrite
                       │
                EN_KEY1=真实URL
                       │
                       ▼
             proxy-providers.airport
                       │
                       ▼
                 机场节点订阅
```

这样可以同时实现：

* GitHub 配置公开
* Mihomo 主配置公开
* Provider 配置结构公开
* 机场真实订阅地址不公开
* OpenClash 启动时自动进行 URL 覆写
* 不需要 SubConverter
* 不需要额外 Docker
* 不需要修改机场订阅内容
* 不影响现有 `proxy-groups` 对 `airport` Provider 的引用

