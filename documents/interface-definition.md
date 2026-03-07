# 映画館窓口業務エージェント インタフェース仕様書

## 1. 文書の目的

本書は、以下の文書に基づき、映画館窓口業務エージェント向け MCP サーバーのインタフェースを定義する。

- [business-requirements.md](business-requirements.md)
- [operation-scenario.md](operation-scenario.md)
- [tool-example.json](tool-example.json)

本仕様は REST API ではなく MCP (Model Context Protocol) を対象とし、JSON-RPC 2.0 形式で `tools/list` および `tools/call` を定義する。

## 2. 適用範囲

本書で定義する MCP ツールは以下の 5 機能を対象とする。

1. 映画情報管理機能
2. 上映スケジュール管理機能
3. 座席空き状況取得機能
4. 座席予約機能
5. 予約確認機能

業務シナリオ上、予約処理は以下の順序で利用することを想定する。

1. `get_movie_list` で作品候補を取得する
2. `get_show_schedule` で上映回を確定する
3. `get_seat_availability` で空席を確認する
4. `reserve_seats` で予約を確定する
5. `get_reservation_details` で予約内容を確認する

## 3. 共通仕様

### 3.1 プロトコル

- 形式: JSON-RPC 2.0
- 対象: MCP サーバー
- 使用メソッド: `tools/list`, `tools/call`

### 3.2 共通リクエスト形式

#### `tools/list`

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {}
}
```

#### `tools/call`

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "get_movie_list",
    "arguments": {
      "date": "2026-03-07"
    }
  }
}
```

### 3.3 共通レスポンス形式

#### 正常応答

MCP のツール実行結果は、JSON-RPC の `result` 配下に返却する。ツール実行結果では、`content` に人間可読な要約、`structuredContent` に機械可読な業務データを格納する。

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "3 件の作品を取得しました。"
      }
    ],
    "structuredContent": {
      "count": 3,
      "items": []
    },
    "isError": false
  }
}
```

#### 異常応答

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "error": {
    "code": -32602,
    "message": "Invalid params",
    "data": {
      "field": "schedule_id",
      "reason": "required"
    }
  }
}
```

### 3.4 エラーコード方針

| 区分 | code | 説明 |
|---|---:|---|
| Parse error | -32700 | JSON 解析失敗 |
| Invalid Request | -32600 | JSON-RPC リクエスト形式不正 |
| Method not found | -32601 | 未対応メソッド |
| Invalid params | -32602 | 入力値不正 |
| Internal error | -32603 | サーバー内部エラー |
| Business error | -32000 | 業務エラー |
| Resource not found | -32001 | 対象データなし |
| Conflict | -32009 | 座席重複予約などの競合 |

## 4. データ構造

本章のデータ構造は [business-requirements.md](business-requirements.md) と整合する形で定義する。

### 4.1 映画データ

```json
{
  "movie_id": "MOV001",
  "title": "Sample Movie",
  "genre": "Drama",
  "duration": 120,
  "rating": 4.5,
  "description": "作品概要",
  "release_date": "2026-03-01"
}
```

### 4.2 上映スケジュールデータ

```json
{
  "schedule_id": "SCH001",
  "movie_id": "MOV001",
  "date": "2026-03-07",
  "start_time": "18:30",
  "end_time": "20:30",
  "theater_id": "THEATER01"
}
```

### 4.3 上映回座席状況データ

```json
{
  "schedule_id": "SCH001",
  "available_seats": [
    {
      "row": "A",
      "available_numbers": [1, 2, 4]
    },
    {
      "row": "B",
      "available_numbers": [2, 3]
    }
  ],
  "occupied_seats": [
    {
      "row": "A",
      "occupied_numbers": [3]
    },
    {
      "row": "B",
      "occupied_numbers": [1, 4]
    }
  ]
}
```

### 4.4 予約データ

```json
{
  "reservation_id": "RSV001",
  "schedule_id": "SCH001",
  "seat_ids": ["A1", "A2"],
  "reservation_time": "2026-03-07T12:34:56+09:00",
  "status": "confirmed"
}
```

## 5. ツール一覧

### 5.1 `tools/list` 応答例

`tools/list` は、MCP クライアントに対して利用可能なツール一覧と入力スキーマを返却する。

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "get_movie_list",
        "description": "現在上映中の映画一覧を取得する",
        "inputSchema": {
          "type": "object",
          "properties": {
            "date": {
              "type": "string",
              "description": "上映日。YYYY-MM-DD 形式"
            },
            "keyword": {
              "type": "string",
              "description": "映画名の検索キーワード"
            },
            "genre": {
              "type": "string",
              "description": "ジャンルによる絞り込み"
            }
          },
          "required": []
        }
      },
      {
        "name": "get_show_schedule",
        "description": "指定した映画の上映スケジュールを取得する",
        "inputSchema": {
          "type": "object",
          "properties": {
            "movie_id": {
              "type": "string",
              "description": "映画 ID"
            },
            "date": {
              "type": "string",
              "description": "上映日。YYYY-MM-DD 形式"
            }
          },
          "required": ["movie_id", "date"]
        }
      },
      {
        "name": "get_seat_availability",
        "description": "指定した上映回の座席空き状況を取得する",
        "inputSchema": {
          "type": "object",
          "properties": {
            "schedule_id": {
              "type": "string",
              "description": "上映スケジュール ID"
            }
          },
          "required": ["schedule_id"]
        }
      },
      {
        "name": "reserve_seats",
        "description": "指定した座席の予約を実行する",
        "inputSchema": {
          "type": "object",
          "properties": {
            "schedule_id": {
              "type": "string",
              "description": "上映スケジュール ID"
            },
            "seat_ids": {
              "type": "array",
              "description": "予約する座席 ID の一覧",
              "items": {
                "type": "string"
              },
              "minItems": 1
            }
          },
          "required": ["schedule_id", "seat_ids"]
        }
      },
      {
        "name": "get_reservation_details",
        "description": "予約 ID を指定して予約内容を取得する",
        "inputSchema": {
          "type": "object",
          "properties": {
            "reservation_id": {
              "type": "string",
              "description": "予約 ID"
            }
          },
          "required": ["reservation_id"]
        }
      }
    ]
  }
}
```

## 6. ツール個別仕様

### 6.1 `get_movie_list`

#### 目的

現在上映中の映画一覧、または条件に一致する映画一覧を取得する。

#### 入力

| 項目名 | 型 | 必須 | 説明 |
|---|---|---|---|
| `date` | string | 任意 | 上映日。`YYYY-MM-DD` 形式 |
| `keyword` | string | 任意 | 映画名の部分一致検索キーワード |
| `genre` | string | 任意 | ジャンルによる絞り込み |

#### 出力

| 項目名 | 型 | 説明 |
|---|---|---|
| `count` | integer | 取得件数 |
| `items` | array | 映画データの一覧 |

#### `tools/call` リクエスト例

```json
{
  "jsonrpc": "2.0",
  "id": 11,
  "method": "tools/call",
  "params": {
    "name": "get_movie_list",
    "arguments": {
      "date": "2026-03-07",
      "keyword": "Sample"
    }
  }
}
```

#### レスポンス例

```json
{
  "jsonrpc": "2.0",
  "id": 11,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "2 件の映画が見つかりました。"
      }
    ],
    "structuredContent": {
      "count": 2,
      "items": [
        {
          "movie_id": "MOV001",
          "title": "Sample Movie",
          "genre": "Drama",
          "duration": 120,
          "rating": 4.5,
          "description": "作品概要",
          "release_date": "2026-03-01"
        }
      ]
    },
    "isError": false
  }
}
```

### 6.2 `get_show_schedule`

#### 目的

指定した映画の上映スケジュールを取得する。

#### 入力

| 項目名 | 型 | 必須 | 説明 |
|---|---|---|---|
| `movie_id` | string | 必須 | 映画 ID |
| `date` | string | 必須 | 上映日。`YYYY-MM-DD` 形式 |

#### 出力

| 項目名 | 型 | 説明 |
|---|---|---|
| `movie_id` | string | 映画 ID |
| `date` | string | 上映日 |
| `items` | array | 上映スケジュールデータの一覧 |

#### `tools/call` リクエスト例

```json
{
  "jsonrpc": "2.0",
  "id": 12,
  "method": "tools/call",
  "params": {
    "name": "get_show_schedule",
    "arguments": {
      "movie_id": "MOV001",
      "date": "2026-03-07"
    }
  }
}
```

#### レスポンス例

```json
{
  "jsonrpc": "2.0",
  "id": 12,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "2026-03-07 の上映回を 3 件取得しました。"
      }
    ],
    "structuredContent": {
      "movie_id": "MOV001",
      "date": "2026-03-07",
      "items": [
        {
          "schedule_id": "SCH001",
          "movie_id": "MOV001",
          "date": "2026-03-07",
          "start_time": "18:30",
          "end_time": "20:30",
          "theater_id": "THEATER01"
        }
      ]
    },
    "isError": false
  }
}
```

### 6.3 `get_seat_availability`

#### 目的

指定した上映回の空席状況を取得する。

#### 入力

| 項目名 | 型 | 必須 | 説明 |
|---|---|---|---|
| `schedule_id` | string | 必須 | 上映スケジュール ID |

#### 出力

| 項目名 | 型 | 説明 |
|---|---|---|
| `schedule_id` | string | 上映スケジュール ID |
| `available_seats` | array | 空席リスト |
| `occupied_seats` | array | 予約済み座席リスト |

#### `tools/call` リクエスト例

```json
{
  "jsonrpc": "2.0",
  "id": 13,
  "method": "tools/call",
  "params": {
    "name": "get_seat_availability",
    "arguments": {
      "schedule_id": "SCH001"
    }
  }
}
```

#### レスポンス例

```json
{
  "jsonrpc": "2.0",
  "id": 13,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "空席状況を取得しました。"
      }
    ],
    "structuredContent": {
      "schedule_id": "SCH001",
      "available_seats": [
        {
          "row": "A",
          "available_numbers": [1, 2, 4]
        },
        {
          "row": "B",
          "available_numbers": [2, 3]
        }
      ],
      "occupied_seats": [
        {
          "row": "A",
          "occupied_numbers": [3]
        },
        {
          "row": "B",
          "occupied_numbers": [1, 4]
        }
      ]
    },
    "isError": false
  }
}
```

### 6.4 `reserve_seats`

#### 目的

指定した上映回に対して座席予約を確定する。

#### 設計上の補足

業務シナリオ上は映画名、日付、上映時間、座席番号をもとに予約を確定するが、MCP ツールでは直前の `get_show_schedule` の結果から得た `schedule_id` を使用して予約対象を一意に特定する。

#### 入力

| 項目名 | 型 | 必須 | 説明 |
|---|---|---|---|
| `schedule_id` | string | 必須 | 上映スケジュール ID |
| `seat_ids` | array[string] | 必須 | 予約する座席 ID 一覧 |

#### 出力

| 項目名 | 型 | 説明 |
|---|---|---|
| `reservation` | object | 予約データ |

#### `tools/call` リクエスト例

```json
{
  "jsonrpc": "2.0",
  "id": 14,
  "method": "tools/call",
  "params": {
    "name": "reserve_seats",
    "arguments": {
      "schedule_id": "SCH001",
      "seat_ids": ["A1", "A2"]
    }
  }
}
```

#### レスポンス例

```json
{
  "jsonrpc": "2.0",
  "id": 14,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "予約を確定しました。予約 ID は RSV001 です。"
      }
    ],
    "structuredContent": {
      "reservation": {
        "reservation_id": "RSV001",
        "schedule_id": "SCH001",
        "seat_ids": ["A1", "A2"],
        "reservation_time": "2026-03-07T12:34:56+09:00",
        "status": "confirmed"
      }
    },
    "isError": false
  }
}
```

#### 業務エラー例

指定された座席が既に埋まっている場合は、競合エラーを返却する。

```json
{
  "jsonrpc": "2.0",
  "id": 14,
  "error": {
    "code": -32009,
    "message": "Selected seats are no longer available",
    "data": {
      "schedule_id": "SCH001",
      "seat_ids": ["A2"]
    }
  }
}
```

### 6.5 `get_reservation_details`

#### 目的

予約 ID を指定して予約内容を取得する。

#### 入力

| 項目名 | 型 | 必須 | 説明 |
|---|---|---|---|
| `reservation_id` | string | 必須 | 予約 ID |

#### 出力

| 項目名 | 型 | 説明 |
|---|---|---|
| `reservation` | object | 予約データ |

#### `tools/call` リクエスト例

```json
{
  "jsonrpc": "2.0",
  "id": 15,
  "method": "tools/call",
  "params": {
    "name": "get_reservation_details",
    "arguments": {
      "reservation_id": "RSV001"
    }
  }
}
```

#### レスポンス例

```json
{
  "jsonrpc": "2.0",
  "id": 15,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "予約内容を取得しました。"
      }
    ],
    "structuredContent": {
      "reservation": {
        "reservation_id": "RSV001",
        "schedule_id": "SCH001",
        "seat_ids": ["A1", "A2"],
        "reservation_time": "2026-03-07T12:34:56+09:00",
        "status": "confirmed"
      }
    },
    "isError": false
  }
}
```

## 7. 実装上の注意事項

- `tools/list` の `inputSchema` は [tool-example.json](tool-example.json) と同様に JSON Schema 形式で返却する。
- `reserve_seats` は複数座席同時予約に対応する。
- 決済処理は本スコープに含めない。
- 映画名の揺れや推薦は、`get_movie_list` の検索条件解釈またはサーバー側補正で対応する。
- 返却データの項目名は、要件定義書のデータ構造に記載された `movie_id`, `schedule_id`, `seat_ids`, `reservation_id` などと一致させる。

## 8. 今後の実装対応

本仕様に基づき、Azure Functions 上の MCP サーバーに以下のツールを実装する。

1. `get_movie_list`
2. `get_show_schedule`
3. `get_seat_availability`
4. `reserve_seats`
5. `get_reservation_details`