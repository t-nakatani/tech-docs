---
title: AI開発の先頭集団の技術選定は「検証できる単純さ」に収束している
date: 2026-10-18
tags: [ai-agents, architecture, tech-selection, mcp, sqlite, typescript, rust]
---

## 背景

コーディングエージェントやAIアプリビルダーを作っている企業（Cursor、Anthropic／Claude Code、OpenAI／Codex、Cognition、Amp、Vercel、Lovable、StackBlitz、Replit、Supabase、Cloudflare、Linear など）が、言語・フレームワーク・DB・入口・検証をどう選び、なぜそう選んだのかを公開情報から横断的に整理したメモ。

各社の公式ブログ・ドキュメント・本人の発言は「本人談」、第三者の報道は「二次情報」、自分の解釈は「推測」と書き分ける。確認できなかった事項は未確認のまま残す。

## 結論

- 技術選定は流行ではなく、次の4条件に収束している。
  1. **モデルが既に知っている**（教えなくてよい）
  2. **依存が少なくレビューできる**
  3. **秒単位で機械的に検証できる**
  4. **分離して安全に試行できる**
- 言語はその帰結として、製品・UI層の **TypeScript** と、性能・分離層の **Rust（または Go）** に二極化している。
- データ層では、分散DBやサーバー側ベクトル索引からの撤退が目立つ。一方で **SQLite の本番採用**が進んでいる。
- 「人間の画面とエージェントが同じAPIを叩く」と一次情報で明言しているのは Linear が最も明確。MCP は正規APIの上に乗る薄い層として扱われている。
- 2024〜2025年は「学習データが多い技術を選べ」だった基準が、2026年には「**エージェントが自分で検証でき、全体を読め、安全に試せる技術を選べ**」に置き換わった（推測）。

## 1. 言語とランタイム

### TypeScript は「教えなくてよい」から

- Claude Code は TypeScript、React + Ink（端末UI）、Yoga（レイアウト）、Bun で作られている。理由は「教える必要のないスタック、Claude Code が自分自身を作れるスタック」、つまりモデルにとって on distribution なスタックだから（本人談）。同記事では、コードの約90%が Claude Code 自身で書かれていると報じられている。
- Bun の位置づけは段階的に変わった。2025年5月は「ビルドとテストに使う、ランタイムではまだ」。2025年12月に Bun チームが Anthropic に合流した時点では「Bun の実行ファイルとして配布」。現行ドキュメントでは「`claude` バイナリは Node を起動しない」。

### Rust は「依存とGCを消す」ため

- OpenAI の Codex CLI は 2025年4月に TypeScript で公開され、同年5月に Rust への書き直しが表明された。理由は4点（本人談）。
  - ゼロ依存インストール（Node 22+ の要件が一部ユーザーの障害だった）
  - ネイティブなセキュリティバインディング（Linux サンドボックスは既に Rust）
  - GC のない性能
  - 他言語から拡張できるワイヤープロトコル
- 2026年の取材では「npm 依存は中身を把握しきれない。Rust なら依存を少なく、レビュー可能に保てる」とも説明している。
- Amp も 2026年5月から CLI を Bun でコンパイルした単一実行ファイルで配布している。

**注意点:** 「ゼロ依存配布」「起動速度」という理由は、**数百万台のマシンに配るCLI製品に固有の制約**である（推測）。社内で動く業務アプリには、Node のバージョン要件を消す動機はほぼない。

### 層ごとの分業

- Cursor：業務ロジックの大半は TypeScript。索引やサンドボックス起動など性能が要る部品だけ Rust で書き、Node API ブリッジでつなぐ。クラウドエージェントのオーケストレータは全面 Rust（EC2 + Firecracker）。
- Cloudflare：コアプロキシ Pingora は「C ができることをメモリ安全に」できるので Rust。Workers 上の自社製品（Workflows、Agents SDK など）は TypeScript。
- AIアプリビルダー：Lovable の生成物は React/TypeScript だが、自社バックエンドは求人票で Go と Rust。StackBlitz の WebContainer は TypeScript・Rust・Wasm。
- 「LLM が書きやすいスタック」と「自社が運用しやすいスタック」は別々に選ばれている（推測）。

### 言語論争は鎮まりつつある

- Armin Ronacher は 2025年6月に「新規バックエンドは Go を強く勧める」と書いた。しかし14か月後には「LLM は言語選択の重要性を以前よりはるかに小さくした」と書いている。
- Simon Willison も「最新モデルと良いハーネスでは、言語による差が続いているか疑わしい」と述べている。
- Stripe のコードベースは数億行の Ruby に Sorbet で型を付けたもので、LLM に馴染みのない社内ライブラリも多い。それでも自律エージェント Minions が週1,000件超のPRをマージしている。
- 効いているのは言語の学習データ量ではなく、**型・lint・テストが機械的に返るループの有無**だと読める（推測）。静的型と動的型で成功率が何%違うかという定量比較は見つからなかった。

## 2. フレームワークと自作

### ハーネスは薄く、足場はモデルの進化に合わせて捨てる

- Claude Code の作者は、Claude Code を「モデルの上の可能な限り薄いラッパー」と表現している（本人談）。モデルのリリースごとにコードやツールを削除し、4.0系ではシステムプロンプトの約半分を消した。
- Amp はエージェントを「an LLM, a loop, and enough tokens」と要約し、400行未満の教材を公開している。マニュアルには「使っていて好きでない機能は殺す」「後方互換なし」と書いている。

### MCP を内部プロトコルにしようとして断念した例

- OpenAI は VS Code 拡張で TUI と同じエージェントループを使うため、まず **Codex を MCP サーバーとして公開する実験をした**。しかし「MCP のセマンティクスを維持するのが難しかった」ので断念し、独自の JSON-RPC プロトコル（App Server）に切り替えた（本人談）。
- エージェントロジックは `codex-rs/core` に集約され、Web・CLI・IDE・デスクトップアプリが同じハーネスを共有する。TypeScript の型定義は Rust のプロトコルから生成している。
- MCP はエージェント内部のストリーミングや承認フローを表すには表現力が足りなかった、という実装者の判断と読める（推測）。

### 選択肢を減らして生成のばらつきを抑える

- Lovable は 2025年2月に「選択肢を極力減らす」「バックエンドは Supabase のみ」「Tailwind は人間の開発者の間のシェリングポイント」「学習データが多い人気の選択肢を取るべき」と述べている（本人談）。
- ところが 2026年には新規プロジェクトを React + Vite の SPA から TanStack Start に切り替えた。理由は SEO に加えて「**エンドツーエンドの型安全が、コードを生成するAIに明確なガードレールを与える**」（本人談）。理由の順序が「人気」から「型による制約」に移っている。
- Vercel は「フロンティアモデルはフレームワーク更新に即座に遅れる」ことを問題として明記し、自社学習のモデルと RAG で補っている（本人談）。人気のフレームワークでも、APIが頻繁に変わるとLLMには不利になる。

### 自作に寄る個人実践者とモデル

- Ronacher は「依存を追加するよりコードを生成する方を好む」と書いている。ORM ではなく素の SQL を使い（エージェントは SQL がうまく、ログとも一致する）、権限チェックはルートのローカルに置く（新しいルートでエージェントが忘れない）。
- Claude Code にツール名を含まない指示を2,430回与えた実測では、20カテゴリ中12で「自作」が最も多かった。ただし著者は「好みであって採用実態ではない」と注記している。
- 逆に Willison は、各ツールが公式の Agent Skills を公開したことで新しいツールもエージェントに使わせられるようになったと書いている。「Choose Boring Technology」が最強の論拠になるという自身の予想は外れた、という。
- 自作の副作用として、エージェントは「抽象を膨らませ、デッドコードを残す」傾向が指摘されている（二次情報）。

## 3. データ

### Postgres 回帰と索引の廃止

- Cursor は分散DB Yugabyte から PostgreSQL（AWS RDS）に戻った（理由の詳細は未確認）。
- ベクトル索引は大規模障害の最中に「数時間で」turbopuffer へ移した。索引は再生成できるデータなので、短時間で移せたと考えられる（推測）。
- 2026年半ばには「モデルが grep と索引検索をうまく使えるようになり、専用の semantic search は意味のある助けにならなくなった」として、**サーバー側の embedding 索引を段階的に廃止**し、ローカルの grep に移行した（本人談）。

### Postgres を中核に据える論理

- Supabase は Postgres を中核にして、周辺は既存の OSS を言語を問わず束ねている（認証は Go、PostgREST は Haskell、Realtime は Elixir など）。
- 設計原則は「既存ツールを使う」「各システムが単体で動く」「API と webhook を公開する」「標準形式（pg_dump、CSV）でロックインを避ける」。
- 一方で 2025年後半には、Lovable・Bolt・Replit がそれぞれ自社のバックエンドやDBを出しており、外部 BaaS 依存から自社バックエンドへ戻る動きもある。

### SQLite の本番採用

- Rails 8 は SQLite を本番の既定にし、WAL と IMMEDIATE トランザクションを既定化した。Solid Cable/Cache/Queue で Redis も不要にした。前提は「ディスクが十分速くなった」こと。
- Fly.io は、マルチリージョン向けの LiteFS より「Litestream の方が動かしやすく理解しやすい」というユーザーの判断を認めている。
- Cloudflare は Durable Objects の SQLite ストレージを GA にし、自社の Workflows エンジンも SQLite-backed Durable Object 上に作っている（本人談）。

### SQLite の限界（当事者の公言）

- Turso は SQLite の弱点として「同時書き込みを扱えない」「変更ストリームを取る良い手段がない」「同期APIしかない」「スキーマの進化が難しい」を挙げている。
- WAL モードは「読み手は多数、書き手は1つ」。2つ目の書き手は `SQLITE_BUSY` になるので、アプリ側で再試行や直列化キューが必要になる。単一の書き手の処理能力が上限になる。
- Cloudflare の「DO ごとに SQLite」は、テナントやセッション単位で書き手を1つに固定し、この制約を設計の前提に変えている（推測）。

### 同期エンジンは正規APIを置き換えない

- Figma の LiveGraph は「永続層ではなくクエリエンジン」という位置づけ。
- Zero は「just a fancy cache」で「段階的に採用・離脱できる」と説明されている。
- Linear は同期エンジンを持ちながら、外部には同じ GraphQL API と webhook を提供している。

## 4. 入口と認証

### 「同じAPI」を明言した Linear

- 公開 API について「It's the same API we use internally for developing our applications」と明言している。更新の検知はポーリングではなく webhook。
- その上に公式 MCP サーバー（Cloudflare 上でホスト）を置き、読み取り専用エンドポイント `/mcp/readonly` や `read` スコープのトークンを用意している。
- Agents API では、エージェントを OAuth の `actor=app` を付けたアプリユーザーとしてインストールする。**課題をエージェントに割り当てると `assignee` ではなく `delegate` に入る**ので、人間が所有権を持ったままエージェントが作業を代行する。
- MCP の内部実装が GraphQL を叩いているかは未確認。ただし MCP は GraphQL を置き換えるものではなく、その上に乗った薄いツール群と読める（推測）。

### Supabase：MCP は開発用、本番はDBロールで守る

- リモート MCP にした理由は「Web型エージェントはリモート HTTP MCP しか使えない」「個人トークンはソース管理に誤コミットされやすい」「`npx` 方式は Node のバージョン差の手間がある」（本人談）。
- 同時に「**開発用途であり、本番DBには繋ぐべきでない**」と明言している。読み取り専用モードは Postgres のロールで強制する。
- 脅威モデルとして「DBのフィールドに埋め込まれた文によるプロンプトインジェクション」を挙げている。読み取り専用・プロジェクトスコープ・手動承認・データラッピングを重ねたうえで、結論は「**ガードレールだけでは不十分**」。

### Cloudflare：人間とエージェントを同じ Worker に入れる基盤

- Access は「アカウント内の全Worker」「単一Worker」「特定ホスト名/パス」の3つの粒度で保護できる。認証済みリクエストでは Worker 内の `ctx.access` からユーザーを取得できる。
- ハマりどころ:
  - Worker 単位のポリシーでは WebSocket のアップグレードが 403 になる。ホスト名ベースのポリシーを使う。
  - Service Binding/RPC 経由では `ctx.access` が渡らない。
- リモート MCP 向けに `workers-oauth-provider` と `McpAgent` を公開しており、各 MCP セッションは専用の Durable Object に紐付く。

### MCP をどこまで使うか

- Stripe は社内 MCP サーバーに400超のツールを集約し、エージェントには厳選したサブセットだけを渡す。
- Ronacher は「MCP はほとんど使わない。代替が信頼できないときだけ使う」として、DB には `psql` を直接使う。Mitchell Hashimoto は `gh` CLI でエージェントに issue/PR をトリアージさせている。
- Amp はエージェントに30分の短命・読み取り専用トークンを渡す（二次情報）。

## 5. 検証と分離

### サンドボックスで承認疲れを消す

- Anthropic は Linux で bubblewrap、macOS で Seatbelt を使い、ファイルシステムとネットワークを OS レベルで分離している。社内では **許可プロンプトが84%減った**（本人談）。
- OpenAI も同様に Seatbelt と bubblewrap + seccomp を既定にしている。既定で有効にすると一般ユーザーの採用を損なうと認めたうえで、詳しくないユーザーに安全でないデフォルトを与えない方を選んだ。
- Cognition は「信頼できないワークロードには完全な分離が要る」「Docker in Docker は難しい」ので自社ハイパーバイザーの VM を選んだ。20GB ディスクのスナップショットを約200msで取る。
- Vercel は Firecracker microVM 上でサンドボックスを動かし、環境変数・DB接続・クラウドリソースへのアクセスを遮断している。
- 分離の重さは、製品が預かる作業の長さと権限に比例している（推測）。

### 秒単位のフィードバックループ

- Stripe：エージェントのステップの間に決定論的なコード（git操作、lint、テスト）を挟む。**push ごとに5秒以内で lint を走らせ、CI は2ラウンドで打ち切る**（それ以上は収穫逓減）。原則は「人間に良いものはLLMにも良い」（本人談）。
- Anthropic：型検査・lint・テスト・実行時エラーで拾えない手動チェックを、スキルとして検証ループにする。CLAUDE.md にビルド・テストコマンドを正確に書く。
- OpenAI：テスト、明確なモジュール境界、AGENTS.md の検証手順を整え、自身の機能に対する self-test を夜間に自動で実行する。
- Hashimoto：「エージェントに自分の作業を検証する手段を与えれば、たいてい自分で直す」。失敗のたびに AGENTS.md を更新するか、ツールを作る。
- Ronacher：「ツールは速く。クラッシュは許容できるがハングは問題」「誤用を明確に報告する」「ログを常にファイルにも出してエージェントが読めるようにする」。
- ファインディ：Git hooks で「タスクランナーが通らないとコミットできない」ようにした。「人間には遅く感じても、AIエージェント前提ではむしろ速くなる」。スキーマは「境界に対する信頼できる唯一の情報源」。

### 検証の対象は「製品として動くか」へ

- Replit Agent 3 は実ブラウザで自己テストして直すループを持つ。
- Cursor は「公開ベンチマークは実際の仕事を反映しない」として、自社エンジニアの実セッションからベンチマークを作った。
- Anthropic の社内調査では、エンジニアが委任するのは「検証が容易で、低リスクか反復的な」タスクだった。

### 検証が機械化されると、人間の同期工程が減る

- Amp（約20名）は main へのマージ前の必須レビューを廃止し、MTTR を MTTF より優先した（二次情報）。
- OpenAI の Codex チームは、非クリティカルなコードは AI レビューだけでマージしてよいことにしている。
- Cursor は「コードが速く書かれるほど、ボトルネックはレビューと安全なマージになる」として Graphite を買収した。

## 小さなチームへの示唆

- **検証が秒単位で返る面を広げる。** 型・lint・テスト・シナリオテストを、エージェントが自分で回せる形にする。エージェントに任せられる範囲は、検証できる範囲と一致する。
- **入口を1本の正規APIに絞る。** 画面・CLI・MCP は同じAPIを叩く。MCP はそのAPIを写すだけの薄い層にして、読み取り用と書き込み用のツールを分ける。
- **権限はデータ層で物理的に絞る。** エージェントは「権限が狭く、責任を持たない、別の利用者」として扱う。各操作に「誰が（人間か、どのエージェントか）」を記録し、所有権は人間に残す（Linear の `delegate`、Supabase のロールで強制する読み取り専用）。
- **依存ゼロ・自作は時代遅れではない。** ただし先頭集団は、その単純さをOSレベルのサンドボックス、秒単位のフィードバック、データ層の権限制限という見えない投資で支えている。単純さだけを真似ると、型なし・認証なし・単一の状態ファイルといった形で、検証と分離の欠如が先に表に出る（推測）。
- **標準形式でダンプ・再構築できる状態を保つ。** 後でDBを変える自由になる。

## 参考リンク

- [How Claude Code is built (Pragmatic Engineer)](https://newsletter.pragmaticengineer.com/p/how-claude-code-is-built)
- [Claude Code (Latent Space)](https://www.latent.space/p/claude-code)
- [Bun joins Anthropic](https://bun.com/blog/bun-joins-anthropic)
- [openai/codex Discussion #1174](https://github.com/openai/codex/discussions/1174)
- [How Codex is built (Pragmatic Engineer)](https://newsletter.pragmaticengineer.com/p/how-codex-is-built)
- [Unlocking the Codex harness (OpenAI)](https://openai.com/index/unlocking-the-codex-harness/)
- [Amp: npm package changes](https://ampcode.com/news/npm-package-changes) / [How to build an agent](https://ampcode.com/how-to-build-an-agent)
- [Cursor (Pragmatic Engineer)](https://newsletter.pragmaticengineer.com/p/cursor)
- [How we built Pingora (Cloudflare)](https://blog.cloudflare.com/how-we-built-pingora-the-proxy-that-connects-cloudflare-to-the-internet/)
- [Agentic Coding (Armin Ronacher, 2025-06)](https://lucumr.pocoo.org/2025/6/12/agentic-coding/)
- [Not so boring (Simon Willison, 2026-03)](https://simonwillison.net/2026/Mar/9/not-so-boring/)
- [Minions (Stripe)](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents)
- [Building apps using TanStack Start (Lovable)](https://lovable.dev/blog/building-apps-using-tanstack-start)
- [v0 composite model family (Vercel)](https://vercel.com/blog/v0-composite-model-family)
- [How AI coding agents choose your stack (Amplifying.ai)](https://amplifying.ai/research/how-ai-coding-agents-choose-your-stack)
- [Supabase Architecture](https://supabase.com/docs/guides/getting-started/architecture)
- [Rails 8.0 Beta 1](https://rubyonrails.org/2024/9/27/rails-8-beta1-no-paas-required)
- [Litestream v0.5.0 (Fly.io)](https://fly.io/blog/litestream-v050-is-here/)
- [Building Workflows (Cloudflare)](https://blog.cloudflare.com/building-workflows-durable-execution-on-workers/)
- [Turso: the next evolution of SQLite](https://turso.tech/blog/turso-the-next-evolution-of-sqlite) / [Concurrent writes on Turso Cloud](https://turso.tech/blog/concurrent-writes-on-turso-cloud)
- [LiveGraph (Figma)](https://www.figma.com/blog/livegraph-real-time-data-fetching-at-figma/)
- [Linear GraphQL API](https://linear.app/developers/graphql) / [Linear MCP](https://linear.app/docs/mcp) / [Linear Agents API](https://linear.app/docs/agents-api)
- [Remote MCP server (Supabase)](https://supabase.com/blog/remote-mcp-server) / [Defense in depth for MCP (Supabase)](https://supabase.com/blog/defense-in-depth-mcp)
- [Cloudflare Access for Workers](https://developers.cloudflare.com/workers/configuration/cloudflare-access/)
- [Remote MCP servers (Cloudflare)](https://blog.cloudflare.com/remote-model-context-protocol-servers-mcp/)
- [Claude Code sandboxing (Anthropic)](https://www.anthropic.com/engineering/claude-code-sandboxing)
- [blockdiff (Cognition)](https://cognition.com/blog/blockdiff)
- [Vercel Sandbox GA](https://vercel.com/blog/vercel-sandbox-is-now-generally-available)
- [Building verification loops with skills (Anthropic)](https://claude.com/blog/building-verification-loops-in-claude-code-with-skills)
- [My AI adoption journey (Mitchell Hashimoto)](https://mitchellh.com/writing/my-ai-adoption-journey)
- [ファインディの登壇資料 (2025-08, Speaker Deck)](https://speakerdeck.com/bicstone/ai-human-friendly-code)
- [How AI is transforming work at Anthropic](https://www.anthropic.com/research/how-ai-is-transforming-work-at-anthropic)
- [Composer 2 technical report (Cursor)](https://cursor.com/blog/composer-2-technical-report)
