# Stats.Hakodate data（JSON-LD 版、スキーマ v2）

これまでの CSV 一式を JSON-LD に置き換えたものです。JSON-LD が正本で、Neo4j にはこのファイルから読み込みます。

TBL0180 は試しに作ったものだったので、表のデータと、そのために追加した語彙（次元・コード・測度・単位・時点）や PageRegion をすべて外しました。今入っている表のデータは TBL0208 だけです。

## フォルダ構成

| ファイル | 内容 |
|---|---|
| context.jsonld | JSON のキーと RDF の語彙の対応表。次元を追加したら `tools/make_context.py` で作り直す |
| vocabulary/code_lists.jsonld | コードリストと、その中のコード（TimePeriod・Area を含む） |
| vocabulary/dimension_properties.jsonld, measures.jsonld, units.jsonld, obs_statuses.jsonld | 次元・測度・単位・観測値の状態 |
| archives/domains.jsonld, pages.jsonld, page_regions.jsonld | 部門・ページ・柱とノンブル |
| archives/tables.jsonld | 全608表と、そのセグメント（入れ子） |
| tables/{table_id}.jsonld | 表ごとのレイアウト層（セグメント > 表の部分・注記・行・列・セル）と意味層（観測値、データ構造） |
| tables/_template.jsonld | 新しい表の雛形（読み込み時には飛ばす） |
| workshop/workshops.jsonld | ワークショップと、その中のセッション（対象の表、提示したページ、道具、参加者） |
| workshop/persons.jsonld, instruments.jsonld | 人物・道具 |
| workshop/annotations.jsonld | アノテーション（書き込み・発話）と、そのコード付け・参照 |
| workshop/analysis_codes.jsonld | コーディングスキームと、その中のカテゴリ・分析コード |
| import.cypher / import_table.cypher / import_references.cypher | Neo4j への読み込みに使う Cypher（`tools/load.py` から実行） |
| tools/ | 読み込み（load.py）、context の再生成、RDF（Turtle）への書き出し、CSV からの移行スクリプト |

## JSON の書き方の決まり

- **1ファイル＝1つの JSON-LD 文書**：`"@context": "../context.jsonld"` と `"@graph": [...]` を持ちます。APOC は `@context` を無視し、`@graph` の中身だけを読みます。
- **キー名＝Neo4j のプロパティ名**：キーは基本的に snake_case で、Neo4j のプロパティ名と同じです。値のないキーは書きません（null や空文字は置かない）。
- **日英の対になる値は言語マップ**：`"title": {"ja": "…", "en": "…"}` のように書くと、Neo4j では `title_ja`・`title_en` になり、RDF では言語タグ付きの文字列になります。
- **ほかのノードへの参照は id の文字列**：`"page": "PAG0650"` のように書き、Neo4j では関係に、RDF では IRI になります。リストで複数指定できます（`"denotes": ["flow/03"]`）。
- **入れ子は所属関係を表す**：テーブル > セグメント、ワークショップ > セッション、コードリスト > コード、スキーム > カテゴリ・分析コード、のように親の中に子を書きます。
- **辺のプロパティは入れ子のオブジェクト**：参加記録（`participants`）、道具の使用（`instruments`）、コード付け（`codings`）、参照（`references`）、データ構造（`components`）は、辺のプロパティを持つオブジェクトとして親の中に書きます。
- **次元の値は `dims`**：`"dims": {"dim-flow": "flow/04", "dim-direction": "direction/01"}` と書きます。RDF では「次元＝述語」（QB のとおり）になります。時点（`time`）と地域（`area`）だけは、`dims` の外に書きます。
- **座標**：`coordinates` にリストで書きます。セルや領域は `[x1, y1, x2, y2]`、アノテーションは `[x, y]` で、どちらもページ画像に対する割合です。
- **キー名と Neo4j のプロパティ名が違う箇所**：JSON-LD の中で同じキー名を別の意味に使えないため、次の2つだけ名前が違います。
  - Cell の `cell_part`：Neo4j では `part`
  - CodeCategory・AnalysisCode の `label`：Neo4j では `value_ja` / `value_en`

## 読み込み（AuraDB を含む）

読み込みは Python のスクリプト `tools/load.py` で行います。スクリプトが JSON-LD を読み、Cypher にパラメータ（`$graph`）として渡します。Neo4j 側のファイル置き場や `apoc.conf` の設定は不要なので、設定を変えられない AuraDB でもそのまま使えます。APOC Core（ラベルの付与に使用）は、AuraDB に標準で入っています。

```bash
pip install neo4j
export NEO4J_URI=neo4j+s://xxxxxxxx.databases.neo4j.io   # AuraDB の接続 URI
export NEO4J_USER=neo4j
export NEO4J_PASSWORD=...
python tools/load.py --dry-run          # 流す内容の確認だけ（接続しない）
python tools/load.py                    # 全部：共有データ → 全ての表 → 参照と確認クエリ
python tools/load.py --table TBL0208    # その表だけ（表を追加・修正したとき）
```

- Cypher ファイルの中で `// @file <パス>` が付いた文には、その JSON-LD の `@graph` の中身が `$graph` として渡されます。
- 表ごとの文には、表の id も `$tbl` として渡されます。
- どの文も MERGE で書いているので、同じものを何度流しても重複しません。

### 読み込み後の確認（期待される件数）

| ノード | 件数 | 関係 | 件数 |
|---|---|---|---|
| Domain | 24 | HAS_PAGE | 1,218 |
| Page | 1,218 | HAS_TABLE | 608 |
| Table | 608 | HAS_SEGMENT / ON_PAGE | 1,351 |
| TableSegment | 1,351 | HAS_CELL（TablePart から 304、Row から 310、Column から 354） | 968 |
| TablePart / Row / Column / Cell | 12 / 40 / 9 / 304 | SUBHEADER_OF / DENOTES / REPRESENTS | 10 / 55 / 254 |
| Note | 4 | APPLIES_TO | 37 |
| Observation | 254 | HAS_DIMENSION_VALUE | 572 |
| CodeList / Code（うち TimePeriod） | 5 / 46（37） | HAS_COMPONENT | 9 |
| DimensionProperty / Measure / Unit / ObsStatus | 5 / 2 / 2 / 6 | TARGETS / PRESENTED / USED | 30 / 47 / 50 |
| Workshop / Session | 13 / 20 | PARTICIPATED_IN | 38 |
| Person / Instrument | 15 / 13 | ANNOTATED_ON / LABELED_AS | 244 / 237 |
| Annotation | 243 | CLASSIFIED_AS | 60 |
| CodeScheme / CodeCategory / AnalysisCode | 1 / 12 / 82 | | |

## 表を追加する手順

1. `tables/_template.jsonld` をコピーして `tables/{table_id}.jsonld` を作ります。
2. 新しい次元が必要なら、`vocabulary/dimension_properties.jsonld` とコードリストに追記し、`python tools/make_context.py .` で context を作り直します。
3. `python tools/load.py --table {table_id}` を実行します。同じ表を再実行しても重複しません。

## RDF として使う

JSON-LD はそのまま RDF なので、`python tools/jsonld_to_turtle.py . all.ttl` で Turtle に書き出せます（約3.5万トリプル）。
