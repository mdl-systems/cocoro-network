# cocoro-network

cocoro OS の全 Docker コンテナが通信する共有ネットワーク（`cocoro-net`）を一元管理するインフラ repo です。

## ネットワーク情報

| 項目 | 値 |
|------|-----|
| ネットワーク名 | `cocoro-net` |
| ドライバ | `bridge` |
| サブネット | `172.30.0.0/24` |

## セットアップ

ネットワークを作成します（初回のみ）：

```bash
cd ~/cocoro-network
docker compose up -d
```

確認：

```bash
docker network ls | grep cocoro-net
docker network inspect cocoro-net
```

## 他 repo からの参照方法

各 repo の `docker-compose.yml` に以下を追加してください：

```yaml
networks:
  cocoro-net:
    external: true
```

## 起動順序

```bash
# Step 1: ネットワーク作成（初回のみ）
cd ~/cocoro-network && docker compose up -d

# Step 2: cocoro-core 起動
cd ~/cocoro-core/infra/docker && docker compose up -d

# Step 3: cocoro-console 起動（nginx 含む）
cd ~/cocoro-console && docker compose up -d
```
