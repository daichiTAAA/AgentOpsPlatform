---
title: サービス設計テンプレート
version: "0.1"
status: draft
owner: TBD
updated: 2025-09-07
related_requirements: []
---

# サービス配下の使い方

新規モジュールは `_template` をコピーして作成:

```
cp -R _template <module>
```

命名例: `auth`, `billing`, `inventory` など。各ファイルの `title`/`related_requirements` を更新する。

