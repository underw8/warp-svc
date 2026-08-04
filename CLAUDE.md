# CLAUDE.md

このファイルは、このリポジトリのコードを操作する際に Claude Code (claude.ai/code) に対するガイダンスを提供します。

## これが何か

warp-svc は公式の Cloudflare WARP Linux クライアントを Docker イメージにパッケージングし、`0.0.0.0:1080` で SOCKS5 プロキシとして公開しています。上流の `warp-svc`/`warp-cli` バイナリは localhost にのみバインドするため、このリポジトリの役割は、それらの周辺のパイプ処理に限定されています：プロセス監視、ポート転送、登録、ヘルスチェック、ログローテーション。アプリケーションのソースコードはなく、すべてシェルスクリプト、Dockerfile、および supervisor/logrotate の設定です。

## アーキテクチャ

4 つのプロセスが supervisord (`configs/supervisord.conf`) の下で実行され、各プロセスは `autorestart=true` を持ちます：

- **warp-svc** (`scripts/warp.sh`) — 実際の `warp-svc` デーモンを起動し、その後 `warp-cli` を操作して：古い `warp-svc` プロセスを終了、新しい WARP アカウントを登録（最大 5 回まで再試行）、内部プロキシをポート 40000 にピン留め、プロキシモードを設定、DNS ログを無効化、`FAMILIES_MODE` を適用、`WARP_LICENSE` が設定されている場合は適用、接続。`warp-cli status` が接続状態を報告すると、`supervisorctl` 経由で `healthcheck` プログラムを起動します（healthcheck は `autostart=false` です — WARP が実際に起動している場合のみ実行されます）。
- **socat** — `tcp-listen:1080`（コンテナのパブリックポート）を `tcp:localhost:40000`（warp-cli の内部プロキシポート）に転送します。この転送は、localhost のみへのバインド制限に対する実際の修正です。
- **healthcheck** (`scripts/healthcheck.sh`) — 60 秒ごとに、ローカルプロキシのポート 40000 を経由して `https://www.cloudflare.com/cdn-cgi/trace/` に curl を実行し、`warp=on` をチェックします。チェックが失敗した場合、`supervisorctl restart warp-svc` を実行して再登録/再接続を強制します。
- **logrotate** (`scripts/logrotate.sh`) — 60 秒ごとに、`/var/lib/cloudflare-warp/*.txt` に対して `logrotate /etc/logrotate.conf` を実行します（`configs/logrotate.conf`：10M サイズ、5 ローテーション、圧縮、copytruncate）。

すべてのスクリプトは、コンテナ停止時に supervisord が子プロセスにシグナルを転送するため、`SIGTERM`/`SIGINT` をグレースフルシャットダウンのためにトラップします。

状態（`/var/lib/cloudflare-warp`）は Docker ボリュームです — WARP 登録を保持するため、ホストで永続化されるか、コンテナが再起動するたびに新しいデバイスとして再登録される必要があります。WARP+ ライセンスは 4 つのデバイスのみをサポートするため、このボリュームを失うとデバイススロットを使用してしまいます。

## 環境変数

- `WARP_LICENSE` — WARP+ ライセンスキー、空でない場合は `warp-cli registration license` 経由で適用されます。
- `FAMILIES_MODE` — `off`、`malware`、`full` のいずれか；`warp-cli dns families` 経由で適用されます。

## 変更を加える

- 起動/登録ロジックへの変更は `scripts/warp.sh` に；プロセスライフサイクル/ログへの変更は `configs/supervisord.conf` に含めます。
- このリポジトリにはテストスイートやリンターがありません — 変更は、イメージをビルドして実行することで検証します（以下を参照）。
- イメージはマルチアーキテクチャ（`linux/amd64,linux/arm64`）です；1 つのアーキテクチャでのみ機能するロジックの追加は避けてください。

### ローカルでビルドして実行

```bash
docker build -t warp-svc .
docker run -d --name=warp -p 127.0.0.1:1080:1080 -v "$(pwd)/warp:/var/lib/cloudflare-warp" warp-svc
```

プロキシが機能していることを確認します：

```bash
curl -x socks5h://127.0.0.1:1080 -sL https://cloudflare.com/cdn-cgi/trace | grep warp
# 期待される出力：warp=on
```

WARP 接続状態を直接チェックします：

```bash
docker exec warp warp-cli --accept-tos status
```

### CI/リリース

`.github/workflows/build.yml` は、`v*` に一致するタグがプッシュされるたびに、マルチアーキテクチャイメージをビルドして `ghcr.io/${{ github.repository }}` にプッシュします。
