# FlexAgentの話

## フレックスエージェントのインストール
[手動インストールについて](https://docs.contrastsecurity.jp/ja/flex-agent-workflow-manual.html)
```bash
# ダウンロード
curl -L --output /tmp/contrast-flex-agent.tar.gz https://contrastsecurity.jfrog.io/artifactory/flex-agent-release/latest/contrast-flex-agent.tar.gz
# 展開
mkdir -p /tmp/contrast-flex-agent
tar -xpzf /tmp/contrast-flex-agent.tar.gz -C /tmp/contrast-flex-agent
# インストール
/tmp/contrast-flex-agent/install.sh --api-token $AGENT_TOKEN
```

## 各種コマンド
```bash
# インジェクト状態
contrast-flex app

# インジェクトされる各言語ごとエージェントバージョン
contrast-flex agents list

# Javaエージェントのアップデート
contrast-flex agents update java

# アートアタッチの停止
contrast-flex auto-attach set false
```
