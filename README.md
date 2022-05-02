# Docker compose templates for CMS on local development

## 使用 Joomla(Apache+PHP) + MariaDB(MySQL) + Traefik 來搭建 CMS

## 準備環境
- [Docker](https://docs.docker.com/engine/install/)

## 操作容器(服務)
```sh
# 當前路徑為專案根目錄

# 啟動容器
docker-compose up -d

# 停止並移除容器
docker-compose down

# 重啟容器
docker-compose restart
```

## 建立本機開發用的 SSL 憑證

可透過 [mkcert](https://github.com/FiloSottile/mkcert) 建立本機開發用的 SSL 憑證

```sh
# 安裝 mkcert
brew install mkcert

# 安裝本機開發用的憑證簽發證書
mkcert -install

# 產生 SSL 憑證
mkdir -p traefik/conf/ssl
mkcert -cert-file traefik/etc/ssl/cert.pem -key-file traefik/etc/ssl/key.pem '*.website'
```

## 啟用 HTTPS 連線
設定 `./traefik/etc/dynamic/tls.yml`。

```yaml
tls:
  stores:
    default:
      defaultCertificate:
        certFile: /etc/traefik/ssl/cert.pem
        keyFile: /etc/traefik/ssl/key.pem
```

## 問題說明

### 問題 1. 想直接修改容器內的網站內容，該怎麼做？
透過將本機路徑掛載至容器中，更新本機檔案時，容器內檔案同時更新。

```yaml
# ./google 為本機相對路徑，/var/www/html 為容器內網站路徑
volumes:
  - ./google:/var/www/html
```

### 問題 2. 想要開啟 HTTPS，需要自簽憑證，要如何製作？
請參考 **建立本機開發用的 SSL 憑證** 和 **啟用 HTTPS 連線**。
> 注意：
> 憑證存放路徑：`./traefik/etc/ssl`
> traefik 指定憑證路徑：`/etc/traefik/ssl`

### 問題 3. 同時開發多個網站要怎麼設定？
此架構下只需要複製多個網站容器(`Joomla`)，`MariaDB` 和 `Traefik` 只需要一個容器即可。

在 `docker-compose.yml` 中複製 `Joomla` 容器配置於既有配置容器下，將 `<name>` 替換成自定義名稱，可以是網站或服務名稱。
> 注意：不能和既有容器名稱重複。
```yaml
<name>:
  image: joomla:3.10.8
  container_name: <name>
  environment:
    JOOMLA_DB_HOST: mysql
    JOOMLA_DB_PASSWORD: root
    JOOMLA_DB_NAME: <name>
  labels:
    - traefik.enable=true
    # 啟用 HTTP
    - traefik.http.routers.<name>.rule=Host(`<name>.website`)
    # 啟用 HTTPS
    - traefik.http.routers.<name>-https.entrypoints=websecure
    - traefik.http.routers.<name>-https.tls=true
    - traefik.http.routers.<name>-https.rule=Host(`<name>.website`)
  restart: unless-stopped
  # 設定本地掛載目錄，可直接修改容器內網站內容
  volumes:
    - ./<name>:/var/www/html
```

### 問題 4. 透過 `Akeeba Kickstart Core` 還原網站，該如何操作？
準備 [Akeeba Kickstart Core](https://www.akeeba.com/products/akeeba-kickstart.html)，在網站目錄下放置 `kickstart.php` 和 `*.jpa` 檔案，並將服務啟動後，依照 `kickstart` 步驟還原。
> 注意：在設定資料庫主機時，請使用服務名稱，而非容器名稱。

```yaml
# 服務名稱
apple:
    image: joomla:3.10.8
    # 容器名稱
    container_name: apple-container-name
```

### 問題 5. `image:joomla:3.10.8` 的 `3.10.8` 是什麼？
是 [joomla image](https://hub.docker.com/_/joomla) 的 `tag`，可以是 `3.10, 3, 3.10.8-apache,3.10.8-php7.4`，可在各個不同 `image repository` 查詢。

### 問題 6. `MariaDB` 可以替換成 `MySQL` 嗎？
可以，把 `image` 替換成 `mysql` 即可，自行決定版本號選用和調整`container_name`(容器名稱是唯一值，不能重複)。

### 問題 7. 如何調整 `PHP` 設定？
將 `php/etc/php.ini` 掛載至容器內，容器內 `PHP` 設定會即時更新，必要時可重啟容器。