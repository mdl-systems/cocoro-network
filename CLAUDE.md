# CLAUDE.md — cocoro-network

## このrepoの役割
cocoro OS 全コンテナの共有ネットワークを定義する。
docker-compose.yml のみで構成。コードなし。

## ネットワーク情報
- ネットワーク名: cocoro-net
- サブネット: 172.30.0.0/24
- ドライバ: bridge

## 起動方法
```bash
cd ~/cocoro-network
docker compose up -d
```

## 他repoからの参照方法
```yaml
networks:
  cocoro-net:
    external: true
```

## コンテナ一覧
| コンテナ名 | repo | 内部ポート |
|-----------|------|-----------|
| cocoro-core | cocoro-core | 8000 |
| cocoro-console | cocoro-console | 3000 |
| cocoro-nginx | cocoro-console | 80→外部公開 |
| cocoro-postgres | cocoro-core | 5432 |
| cocoro-redis | cocoro-core | 6379 |

## 更新履歴
| 日付 | 内容 |
|------|------|
| 2026-03-11 | 初版作成 |
