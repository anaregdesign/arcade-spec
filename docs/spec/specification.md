# Workflow-Owned Azure Delivery Contract

## Result Recommendation Refresh

### Summary

リザルト画面の推薦エリアは、結果の補足説明よりも次に遊ぶゲームの発見を優先し、ホーム画面と共通のサムネイル付きカードで次のゲーム候補を案内する。

### User Problem

現状のリザルト画面は補足テキスト量が多く、推薦リストが画面下部で見つけづらい。特に推薦カードにサムネイルがないため、ホーム画面のゲーム一覧と見た目が分断され、次に遊ぶゲームを直感的に選びにくい。

### Users And Scenarios

- クリア直後に次のゲームを探したいプレイヤーが、リザルト画面からそのまま別ゲームへ移動する
- モバイル端末のプレイヤーが、長い説明文を読む前に推薦候補のゲームを視認する

### Scope

- リザルト画面の推薦セクションでホーム画面と共通のゲームカード表現を使う
- 推薦カードにゲームのサムネイルを表示する
- 推薦カードの上に並ぶ補足テキストを必要最小限まで整理して、推薦グリッドを見つけやすくする

### Non-Goals

- ホーム画面の検索、フィルター、並び順ロジックを変更すること
- レコメンド順位アルゴリズム自体を変更すること
- ゲームごとのサムネイルアセットを新規作成すること

### User-Visible Behavior

- リザルト画面の `Pick the next game` セクションは、ホーム画面と同系統のサムネイル付きカードを最大3件表示する
- 各推薦カードはゲーム名、サムネイル、短い補足説明、プレイ導線を表示し、ホーム画面の一覧と視覚的に連続した体験にする
- 推薦カードの直前にある結果補足テキストは簡潔になり、推薦グリッドがセクション内で優先的に視認できる

### Acceptance Criteria

- リザルト画面の推薦カードに各ゲームのサムネイルが表示される
- ホーム画面とリザルト画面のカード表現が共通コンポーネントで構成される
- リザルト画面で推薦カードより前に表示される補足文量が減り、推薦グリッドがより早く見つけられる
- サムネイル未設定ゲームでも既存のフォールバック表現でカードが成立する

### Edge Cases

- 推薦候補にサムネイルがない場合でも、ゲーム名のフォールバック表示でカードが崩れない
- 推薦候補が 3 件未満でもグリッドと導線が自然に表示される

### Constraints And Dependencies

- ホーム画面ですでに使われているプレビュー画像メタデータを再利用する
- リザルト画面のレコメンド順と件数は既存のサーバー側ロジックを維持する

### Links

- Active plan: `/docs/plans/plan.md`

## Summary

production / shared Azure delivery は GitHub Workflow を唯一の control plane entrypoint としつつ、`Private Endpoint` 配下の Azure SQL data-plane 操作は Azure-hosted `Container Apps Job` に限定する。`Container App runtime` は server process の起動だけを担当し、schema change は release / recovery workflow 上の dedicated migration job に分離する。migration job は `prisma migrate deploy` を直接叩かず、checked-in Prisma SQL migrations を Azure-hosted `mssql` runner で適用する。suffix-aware naming contract は維持し、`green` / `blue` / `dev` ごとに hosted environment を独立させる。`Bootstrap Azure Recovery` は routine production suffix とは別の suffix に side-by-side replica を構築し、既存環境は残したまま recovery target だけを収束させる。cross-suffix で使い回してよい shared state は GitHub Actions OIDC と OAuth contract に必要な Azure Application / Service Principal に限定する。

## User Problem

現状は `runtime startup`、`Prisma migration`、`Azure SQL principal bootstrap`、`GitHub-hosted workflow` の責務境界が曖昧で、以下が起きやすい。

- `Container App` の replica restart が schema change failure に巻き込まれる
- GitHub-hosted runner では到達できない `Private Endpoint` / `Managed Identity` 前提の処理を workflow 側で担ってしまい、failure reason がぶれる
- release / recovery で必要な permission と preconfiguration が暗黙化し、operator が drift を見落としやすい
- same-suffix rerun 時に partial state を再収束できても、runtime startup failure が hosted rollout 全体を止める

## Users And Scenarios

- operator が routine release を GitHub Release publish で実行する
- operator が `Bootstrap Azure Recovery` を routine production suffix とは別の recovery suffix に対して rerun する
- operator が Azure SQL principal drift や runtime crash の原因を permission / identity / preconfiguration 単位で切り分ける

## Scope

- `release` / `recovery` workflow の責務分離を明示する
- shared rollout responsibility を reusable workflow に集約し、entry workflow ごとの重複 job 定義をなくす
- GitHub-hosted job と Azure-hosted job の execution surface を分離する
- runtime / migration / SQL bootstrap identity の permission boundary を固定する
- suffix-aware naming contract と same-suffix rerun contract を維持する
- recovery workflow が routine production suffix を target にせず、side-by-side replica suffix だけへ deploy する contract を明示する
- cross-suffix reuse を Azure Application / Service Principal に必要な shared identity へ限定する
- required GitHub Environment variables, secrets, Azure RBAC, SQL grants, registry preconfiguration, network prerequisite を repo docs に明示する
- Azure Front Door 配下でも hashed static asset が `HTML fallback` や compressed-transfer abort に巻き込まれない runtime / edge contract を明示する

## Non-Goals

- production/shared deploy を local Azure CLI path に戻すこと
- `Private Endpoint` を外して GitHub-hosted runner から Azure SQL へ直接接続すること
- runtime identity と migration identity を 1 つに統合すること
- destructive rollback shortcut として `AllowAzureServices` や Azure SQL public access を再導入すること

## User-Visible Behavior

- `Release Azure Delivery` は routine release entry workflow として `publish -> classify_release_scope -> optional plan_infra -> optional deploy_infra` を担当し、release tag 間 diff に rollout-owned change があるときだけ shared reusable rollout workflow を call する
- `Release Azure Delivery` は release tag 間の repo diff を先に分類し、`infra/` と baseline-defining Azure parameter / name resolution scripts に変更がない app-only release では `plan_infra` と `deploy_infra` を skip して shared reusable rollout workflow へ直接進む
- `Release Azure Delivery` は rollout が必要な release でも、runtime-config-owned change がなければ `sync_runtime_config` を skip し、database-owned change がなければ `bootstrap_sql` と `migrate_database` を skip したまま `deploy_app -> smoke_test` を継続する
- `Release Azure Delivery` は docs / workflow-only など rollout-owned repo inputs を変えない release では GHCR への image publish と attestation だけを実行し、App Configuration / Key Vault / Azure SQL / Container App rollout を起こさない
- infra-owned file change を含む release だけが `plan_infra` で Azure `what-if` を実行し、`what-if` が実 change を返したときだけ `deploy_infra` を実行する
- `Bootstrap Azure Recovery` は recovery entry workflow として routine production suffix とは別の recovery suffix を validation したうえで `resolve_recovery_image -> ensure_resource_group -> deploy_bootstrap_infra -> restore_production_release_rbac` までを担当し、その後同じ shared reusable rollout workflow を call して `sync_runtime_config -> bootstrap_sql -> run_database_migration -> deploy_app -> smoke_test` を実行し、最後に `verify_runtime` を行う
- recovery の `verify_runtime` は hosted infra, private-link, identity, runtime env, smoke-test 後の contract verification を担当し、shared Entra app registration に recovery host callback がまだ追加されていない間は `/auth/start` redirect assertion を必須にしない
- GitHub-hosted workflow は Azure SQL に直接 login しない
- Azure SQL principal bootstrap は release / recovery の両 workflow で SQL bootstrap identity で動く Azure-hosted `Container Apps Job` が担当する
- schema migration は migration identity で動く Azure-hosted `Container Apps Job` が担当し、checked-in Prisma SQL files を direct `mssql` execution で適用する
- `Container App` の default startup path は migration を待たずに server process を起動する
- runtime `Container App` は runtime identity だけを attach し、migration identity は attach しない
- runtime container env には `AZURE_SQL_RUNTIME_CLIENT_ID` だけが残り、`AZURE_SQL_MIGRATION_CLIENT_ID` と `STARTUP_MIGRATION_DATABASE_URL` は残らない
- suffix-aware naming により `green` / `blue` / `dev` は cross-suffix collision を起こさない
- recovery workflow は routine production suffix `green` を fail fast で拒否し、current production environment を in-place mutate しない
- Front Door private-link approval watcher は ARM deployment と同じ長さの deadline で managed environment と pending connection を監視し、`5分固定リトライ窓` による premature timeout を起こさない
- runtime server は `build/client` と `public` の static file を application routing より前に直接返し、hashed asset には `public, max-age=31536000, immutable` を付ける
- Front Door の asset route は cache を維持しつつ edge compression を有効にせず、origin からの static asset body を途中切断なく配信する
- login / home / game routes の first paint は missing CSS による layout collapse を起こさず、root stylesheet request が practical latency で完了する

## Acceptance Criteria

- GitHub-hosted workflow から Azure SQL `Private Endpoint` への direct data-plane dependency がない
- Azure SQL principal bootstrap と schema migration はどちらも Azure-hosted `Container Apps Job` で実行される
- routine release / recovery entry workflow に duplicated rollout job definition が残らず、shared rollout path は reusable workflow 1 か所で管理される
- app-only release tag が infra-owned repo inputs を変更していない場合、routine release は Azure infra plan / deploy を起こさず shared rollout を継続する
- rollout-owned repo inputs を変更していない release tag は shared rollout 自体を起こさず、GHCR publish だけで完了する
- infra deploy は repo diff による infra-owned change detection と Azure `what-if` の両方を通過したときだけ走る
- runtime-config-owned change がない routine release は `sync_runtime_config` を起こさず、Key Vault / App Configuration data-plane access を要求しない
- database-owned change がない routine release は `bootstrap_sql` と `migrate_database` を起こさない
- `Container App` revision restart や scale-out は schema migration の成否に依存しない
- runtime `Container App` は runtime identity のみを attach し、migration identity attachment は不要になる
- workflow docs に required Azure RBAC, SQL grants, GitHub Environment variables/secrets, registry prerequisite, network prerequisite が列挙されている
- release / recovery workflow は touched YAML validation と local verification を通る
- recovery verification は shared Entra redirect URI reconciliation 前でも infra/runtime contract の検証を継続でき、auth redirect assertion は production verification から分離できる
- Front Door private-link approval は Azure deployment が継続中のあいだ deadline まで retry し、deployment failure は待機タイムアウトに埋もれずその場で surfacing される
- Front Door default domain から取得する `root-*.css` と route-level `*.css` / `*.js` asset が `200` と expected cache headers を返し、HTML fallback を返さない
- static asset GET は `accept-encoding: identity` workaround を必要とせず browser default request で完走する

## Edge Cases

- target resource group に同一 type resource が複数ある場合は fail fast する
- same-suffix rerun では recovery target suffix 内の active resource または recoverable resource を優先して reuse / recover する
- cross-suffix では hosted Azure resources を逆引き reuse せず、side-by-side replica resource set を別 suffix で持つ
- `App Configuration` / `Key Vault` の global-name collision は operator-managed suffix rotation で逃がせる
- registry access が `identity` か `username/password` のどちらでもない場合は `Container Apps Job` 作成前に fail する
- migration job が fail した場合、`deploy_app` は進まない

## Constraints And Dependencies

- production/shared environment の deploy は GitHub Workflow 経由のみ
- GitHub-hosted runner は Azure SQL `Private Endpoint` に直接到達しない前提を守る
- Azure SQL は `Entra-only` auth, `publicNetworkAccess=Disabled`, `Private Endpoint` を維持する
- `Container App` runtime config は GitHub Workflow が App Configuration / Key Vault / Container App env contract を収束させる
- shared app identity や Entra app registration など human-managed shared identity は suffix ごとに複製しない
- GitHub Actions OIDC deploy identity と OAuth app registration に必要な Azure Application / Service Principal は shared identity として reuse する

## Links

- Active plan: `/docs/plans/plan.md`
- Azure prerequisites: `/docs/azure-prerequisites.md`
- Production data path: `/docs/production-data-path.md`

## Repository Architecture Diagram Publication

### Summary

README から参照できるリポジトリ全体のアーキテクチャ図を追加し、Azure のホスティング構成と GitHub ベースのリリース経路を一目で把握できる状態にする。

### User Problem

現在の README には Azure Front Door, Container Apps, Azure SQL, App Configuration, Key Vault, GitHub Actions OIDC, GHCR の関係を一枚で示す図がなく、リポジトリを初見で開いたときに公開構成と責務分離を把握しづらい。

### Scope

- リポジトリの Azure 公開構成を示すアーキテクチャ図を追加する
- README から図を参照できるようにする
- 図では可能な範囲で Azure と GitHub の公式アイコンを使う
- GitHub release workflow から GHCR publish と Azure rollout へ流れる主要経路を表現する

### Non-Goals

- Azure リソース構成そのものを変更すること
- README 以外の運用手順書を全面改稿すること
- 実稼働の subscription, resource group, hostname を図に固定値として埋め込むこと

### User-Visible Behavior

- README にアーキテクチャ図セクションが追加される
- 図を見ると、利用者のアクセス経路、Azure Front Door, Container App runtime, Azure-hosted job, App Configuration, Key Vault, Azure SQL, observability, GitHub Actions と GHCR のつながりが把握できる
- 図はローカルのリポジトリアセットとして管理され、README から直接表示される

### Acceptance Criteria

- README からアーキテクチャ図が表示できる
- 図がこのリポジトリの現在の Azure delivery contract と矛盾しない
- 図中の主要 Azure サービスは Azure 公式 SVG アイコンで表現される
- GitHub 側の release and rollout 経路が図中で識別できる
- 仮想ネットワーク境界、delegated subnet、private endpoint subnet、Azure SQL private endpoint の位置関係が `infra/main.bicep` と矛盾しない

### Constraints And Dependencies

- 図は GitHub 上の README 表示を前提に、リポジトリ内のファイルとして参照できる形式にする
- 表現対象は `README.md`, `infra/main.bicep`, `.github/workflows/*.yml`, `docs/production-operations.md`, `docs/production-data-path.md` と整合させる
- `App Configuration` と `Key Vault` の公開状態は現行 Bicep の `publicNetworkAccess` 設定を優先して表現する

### Links

- Active plan: `/docs/plans/plan.md`
- README: `/README.md`
- Diagram target directory: `/docs/diagrams/`
