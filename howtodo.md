下面是一份**面向像我这样的小呆瓜的「自建邮局」保姆级方案**，目标是：  
✅ 用一台云服务器 + 一个域名，跑起能收发邮件、Web 网页客户端、手机客户端都能用的“私人邮局”。  
✅ 全程几乎都是 copy-paste 命令，30 行以内搞定，不纠结原理，**先做出一坨屎，再慢慢优化成屎山**
✅ 采用 Docker 版 Poste.io（开箱即用，自带反垃圾/病毒、Webmail、SSL 自动续期），省掉 80% 坑。

---

### 一、准备物资（5 分钟）
| 东西 | 最低要求 | 推荐 | 备注 |
|---|---|---|---|
| 云服务器 | 1 核 1 G 内存 | 1 核 2 G+ | 必须**开放 25 端口**（用来发信），最好支持 rDNS（防进垃圾箱）。 |
| 域名 | 任意后缀 | .com/.cn | 国内服务器要先备案。 |
| 操作系统 | Ubuntu 20.04+ / Debian 11+ | 同上 | 纯净系统，不要装宝塔。 |

⚠️ **25 端口检测**  
连上服务器先执行：  
```bash
telnet smtp.qq.com 25 
//这条命令其实就做一件最简单的事：
“从你的服务器‘伸手’去摸一下腾讯 QQ 邮箱的 25 端口，看能不能连通。”
```  
能出现 `220 … ESMTP` 字样才算通过；如果直接 `timeout`，换商家或发工单解封，否则后续白搭 。

---

### 二、一键装 Docker（2 分钟）
```bash
# 1. 官方一键脚本
curl -fsSL https://get.docker.com | bash
# 2. 开机自启
systemctl enable --now docker
# 3. 装 docker-compose（新版已集成插件）
apt install docker-compose-plugin -y
```

---

### 三、域名解析（3 分钟）
去你的域名注册商控制台，把下面 5 条全加齐（示例域名 `yourdomain.com`，替换成自己的）：

| 记录类型 | 主机记录 | 记录值 | 说明 |
|---|---|---|---|
| A | mail | 服务器 IP | Webmail 入口 |
| MX | @ | mail.yourdomain.com. | 10 | 收信路由 |
| TXT | @ | v=spf1 mx ~all | 防伪造 |
| TXT | _dmarc | v=DMARC1; p=none; rua=mailto:postmaster@yourdomain.com | 防伪造 |
| PTR | IP→mail.yourdomain.com | 找云厂商工单设置 | 反解，提高外站送达率 |

---

### 四、跑起邮局（1 分钟）
```bash
# 1. 建个目录
mkdir /opt/poste && cd /opt/poste

# 2. 一键启动（80/443 会被自动占用，如冲突先停 Nginx/Apache）
docker run -d \
  --name=poste \
  -p 25:25 -p 80:80 -p 443:443 -p 110:110 -p 143:143 \
  -p 465:465 -p 587:587 -p 993:993 -p 995:995 \
  -v /opt/poste/data:/data \
  -e "HTTPS=ON" \
  -e "VIRTUAL_HOST=mail.yourdomain.com" \
  --restart always \
  poste.io/poste
```

---

### 五、初始化配置（3 分钟）
1. 浏览器访问 `http://mail.yourdomain.com`  
2. 首次设置管理员邮箱：`postmaster@yourdomain.com` + 强密码  
3. 后台 → **System settings** → **TLS certificate** 点 **Issue Free LetsEncrypt**，30 秒后自动 HTTPS。  
4. **Virtual domains** 里把 `yourdomain.com` 设为默认域。  
5. **Virtual accounts** 里新增普通用户，如 `me@yourdomain.com`。

---

### 六、客户端收发
| 场景 | 服务器地址 | 端口 | SSL |
|---|---|---|---|
| 网页 | https://mail.yourdomain.com | — | 自动 |
| SMTP 发信 | mail.yourdomain.com | 465/587 | SSL |
| IMAP 收信 | mail.yourdomain.com | 993 | SSL |
| 账号 | 完整邮箱地址 | 密码 | 就是后台设的 |

---

### 七、冒烟测试
1. 用 Webmail 给自己 Gmail 发一封，能收到即 SMTP 通。  
2. 从 Gmail 回一封，能收到即 MX/IMAP 通。  
3. 检查垃圾邮件夹，若出现提示“未加密/未验证” → 继续看第八步加 DKIM。

---

### 八、进阶：加 DKIM 签名（10 分钟，可选但强烈建议）
Poste.io 后台 → **Virtual domains** → 选中域名 → **DKIM** → 生成 key → 把给出的 TXT 记录加到 DNS（主机名默认 `default._domainkey`）。  
生效后 Gmail 原文里能看到 `dkim=PASS`，进垃圾箱概率大幅下降。

---

### 九、常见坑速查
| 现象 | 排查 |
|---|---|
| 25 端口 telnet 超时 | 换 VPS 或发工单解封  |
| 只能收不能发 | 检查防火墙/安全组是否放通 587/465；看日志 `docker logs poste` |
| 发 Gmail 进垃圾箱 | 补全 SPF+DKIM+DMARC+PTR  |
| 证书不自动续 | 确认 80/443 一直能被外网访问 |
| 内存不足 | 给 2 G 以上或加 2 G swap  |

---

### 十、日常维护
- 升级邮局：`docker pull poste.io/poste && docker restart poste`  
- 备份：`tar -zcf poste-backup.tar.gz /opt/poste/data`  
- 监控：登录后台首页就有队列/硬盘/病毒库状态。

---

完成！你现在拥有 100% 自主控制的域名邮局，可以无限开账号、无限发件（只要 IP 信誉好），再也不用买企业邮箱了。玩得开心~
