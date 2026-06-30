# 部署到 wordtest1.tumei.online（腾讯云 · 大陆访问）

参照高考词汇诊断（`wordtest.tumei.online`）与背单词项目（`word.tumei.online`）的方式，
把中考词汇诊断发布到**已备案的国内服务器**。GitHub（`GoodeSam/zhongkao-vocabulary-size-test`）作源代码备份。

## 现状
- **网址**：https://wordtest1.tumei.online/ （DNS 生效 + 证书签发后可用）
- **服务器**：腾讯云轻量，广州，`43.139.242.52`，系统 Nginx
  （与 `word` / `wordtest` / `home.tumei.online` 同一台）
- **站点根目录**：`/var/www/wordtest1.tumei.online`（只有一个 `index.html`，单文件应用）
- **Nginx 配置**：`/etc/nginx/conf.d/wordtest1.tumei.online.conf`
- **部署密钥**：本机 `~/.ssh/tumei_deploy`（与高考/背单词项目共用）

## 日常更新内容
改完 `index.html` 后，一条命令：
```bash
bash deploy/deploy-tumei.sh
```

## 首次部署步骤（备查）
1. **DNS**（DNSPod → `tumei.online`）：`wordtest1`　类型 `A`　记录值 `43.139.242.52`。
2. 服务器建目录 `/var/www/wordtest1.tumei.online`，rsync 上传 `index.html`。
3. 写 `/etc/nginx/conf.d/wordtest1.tumei.online.conf`（root + index.html 不缓存），reload。
4. DNS 生效后签发证书 + 强制 HTTPS：
   ```bash
   ssh -i ~/.ssh/tumei_deploy root@43.139.242.52 \
     'certbot --nginx -d wordtest1.tumei.online --redirect -n --agree-tos -m sgoode017@gmail.com'
   ```
