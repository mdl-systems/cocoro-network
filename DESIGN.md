# cocoro-network 設計書

> cocoro OSの全Dockerコンテナのネットワークを一元管理するインフラrepo

---

## 設計方針

1. **単一ネットワーク**: 全repoのコンテナが1つのネットワークで通信
2. **名前解決**: コンテナ名でアクセス可能（IPアドレス不要）
3. **シンプル**: docker-compose.ymlのみで管理・コード不要
4. **拡張性**: 将来のcocoro-node・cocoro-cloudにも対応

---

## 現在の問題

```
cocoro-core    → docker_default ネットワーク
cocoro-console → cocoro-console_cocoro-net ネットワーク
cocoro-nginx   → cocoro-console_cocoro-net ネットワーク

→ cocoro-nginx から cocoro-core に名前解決できない
→ IPアドレス直打ちで暫定対応中（再起動でIPが変わるリスク）
```

---

## 解決策：共有ネットワークの作成

### ディレクトリ構成

```
cocoro-network/
├── docker-compose.yml    # 共有ネットワーク定義
├── CLAUDE.md
└── README.md
```

### docker-compose.yml

```yaml
# cocoro-network/docker-compose.yml
# cocoro OS 全コンテナの共有ネットワーク定義

networks:
  cocoro-net:
    driver: bridge
    name: cocoro-net          # 固定名（他repoから external: true で参照）
    ipam:
      config:
        - subnet: 172.30.0.0/24   # 他ネットワークと競合しない範囲
```

### 起動方法

```bash
# 最初に一度だけ実行（ネットワーク作成）
cd ~/cocoro-network
docker compose up -d

# 確認
docker network ls | grep cocoro-net
docker network inspect cocoro-net
```

---

## 各repoのdocker-compose.yml修正方針

### cocoro-core（infra/docker/docker-compose.yml）

```yaml
services:
  cocoro:
    container_name: cocoro-core
    networks:
      - cocoro-net          # 共有ネットワークに参加

networks:
  cocoro-net:
    external: true          # cocoro-networkが作成したネットワーク
```

### cocoro-console（docker-compose.yml）

```yaml
services:
  cocoro-console:
    container_name: cocoro-console
    networks:
      - cocoro-net

  nginx:
    container_name: cocoro-nginx
    networks:
      - cocoro-net

networks:
  cocoro-net:
    external: true
```

### nginx.conf の修正

共有ネットワーク導入後はコンテナ名で名前解決できるため、
IPアドレス直打ちを廃止できます：

```nginx
# 修正前（IPアドレス直打ち・再起動でIPが変わるリスクあり）
location ~ ^/(chat|health|...) {
    proxy_pass http://172.20.0.4:8000;
}

# 修正後（コンテナ名で安定アクセス）
location ~ ^/(chat|health|...) {
    proxy_pass http://cocoro-core:8000;
}
```

---

## 起動順序

```bash
# Step 1: ネットワーク作成（初回のみ）
cd ~/cocoro-network
docker compose up -d

# Step 2: cocoro-core 起動
cd ~/cocoro-core/infra/docker
docker compose up -d

# Step 3: cocoro-console 起動（nginx含む）
cd ~/cocoro-console
docker compose up -d
```

### systemdサービス化（miniPC自動起動用）

```ini
# /etc/systemd/system/cocoro.service
[Unit]
Description=Cocoro OS
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/cocoro-start.sh
ExecStop=/usr/local/bin/cocoro-stop.sh

[Install]
WantedBy=multi-user.target
```

```bash
# /usr/local/bin/cocoro-start.sh
#!/bin/bash
cd /home/cocoro-admin/cocoro-network && docker compose up -d
cd /home/cocoro-admin/cocoro-core/infra/docker && docker compose up -d
cd /home/cocoro-admin/cocoro-console && docker compose up -d
```

---

## コンテナ名・ポート一覧

| コンテナ名 | repo | 内部ポート | 外部公開 |
|-----------|------|-----------|---------|
| cocoro-core | cocoro-core | 8000 | なし |
| cocoro-console | cocoro-console | 3000 | なし |
| cocoro-nginx | cocoro-console | 80 | 0.0.0.0:80 |
| cocoro-postgres | cocoro-core | 5432 | なし |
| cocoro-redis | cocoro-core | 6379 | なし |

---

## 将来の拡張

### cocoro-agent 追加時

```yaml
# cocoro-agent/docker-compose.yml
services:
  cocoro-agent:
    container_name: cocoro-agent
    networks:
      - cocoro-net

networks:
  cocoro-net:
    external: true
```

nginx.confに追加：

```nginx
location ~ ^/(tasks|agent) {
    proxy_pass http://cocoro-agent:8002;
}
```

### cocoro-node（複数miniPC）対応時

```yaml
# cocoro-network/docker-compose.yml に追加
networks:
  cocoro-net:
    driver: overlay    # bridge → overlay に変更
    attachable: true
```

---

## 実装タスク

| タスク | 優先度 | 状態 |
|--------|--------|------|
| docker-compose.yml作成（ネットワーク定義） | 🔴 高 | ❌ 未着手 |
| cocoro-core docker-compose.yml修正 | 🔴 高 | ❌ 未着手 |
| cocoro-console docker-compose.yml修正 | 🔴 高 | ❌ 未着手 |
| nginx.conf をコンテナ名に修正 | 🔴 高 | ❌ 未着手 |
| systemdサービス化 | 🟡 中 | ❌ 未着手 |
| CLAUDE.md作成 | 🟡 中 | ❌ 未着手 |
| README.md作成 | 🟢 低 | ❌ 未着手 |

---

## 更新履歴

| 日付 | 内容 |
|------|------|
| 2026-03-11 | 初版作成 |
