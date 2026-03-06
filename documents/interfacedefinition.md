# 映画館の窓口業務エージェント インタフェース仕様書

## 1. 文書の目的
本書は、[documents/business-requirements.md](documents/business-requirements.md) に定義した要件に基づき、映画館の窓口業務エージェント用 MCP サーバーのインタフェース仕様を定義するものである。

本仕様は REST API ではなく MCP によるツール呼び出しを前提とし、JSON-RPC 2.0 形式でツールの列挙およびツール実行を行う。

## 2. 参照文書
- [documents/business-requirements.md](documents/business-requirements.md)
- [documents/operation-scenario.md](documents/operation-scenario.md)
- [documents/tool-example.json](documents/tool-example.json)

## 3. 前提
- [documents/operation-scenario.md](documents/operation-scenario.md) は現時点で空ファイルである。
- そのため、本仕様は要件定義書に基づき PoC 向けに定義する。
- 本仕様は MCP クライアントからの `tools/list` および `tools/call` を対象とする。
- 実装は Azure Functions 上の MCP サーバーを想定するが、本書では実装方式ではなくインタフェースを定義する。

## 4. MCP 共通仕様

### 4.1 プロトコル
- JSON-RPC バージョン: `2.0`
- 通信対象: MCP サーバー
- 主な利用メソッド:
  - `tools/list`
  - `tools/call`

### 4.2 共通リクエスト形式
#### ツール一覧取得
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {}
}
```

#### ツール実行
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "tool_name",
    "arguments": {}
  }
}
```

### 4.3 共通レスポンス形式
#### 正常応答
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {}
}
```

#### 異常応答
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32000,
    "message": "Business error message",
    "data": {
      "error_type": "validation_error"
    }
  }
}
```

### 4.4 エラーコード方針
本 PoC では、MCP / JSON-RPC の標準エラーと業務エラーを以下のように扱う。

| 区分 | code | 用途 |
|---|---:|---|
| Parse error | -32700 | JSON 解析失敗 |
| Invalid Request | -32600 | JSON-RPC 形式不正 |
| Method not found | -32601 | 未対応メソッド |
| Invalid params | -32602 | ツール入力不正 |
| Internal error | -32603 | サーバー内部異常 |
| Business error | -32000 | 業務エラー全般 |
| Resource not found | -32001 | 対象データなし |
| Conflict | -32009 | 満席、重複予約など |

### 4.5 ツール実行結果の返却方針
各ツールの `result` は、MCP クライアントで扱いやすいよう、以下の情報を基本に返却する。

- `success`: 実行成否
- `message`: 利用者向け要約メッセージ
- `data`: 業務データ本体
- `suggestions`: 代替候補や次アクション

例:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "success": true,
    "message": "作品一覧を取得しました。",
    "data": {},
    "suggestions": []
  }
}
```

## 5. 管理データ構造
要件定義書と整合する PoC 用のデータ構造を以下に示す。

### 5.1 `Movie`
```json
{
  "movie_id": "MOV001",
  "title": "Sample Movie",
  "description": "作品概要",
  "duration_minutes": 120,
  "genre": "Drama",
  "age_rating": "G",
  "release_start_date": "2026-03-01",
  "release_end_date": "2026-04-30",
  "status": "now_showing",
  "audio_type": "dubbed",
  "subtitle_type": "none"
}
```

### 5.2 `Screen`
```json
{
  "screen_id": "SCR01",
  "screen_name": "Screen 1",
  "seat_layout_name": "standard_120",
  "total_seat_count": 120
}
```

### 5.3 `Seat`
```json
{
  "seat_id": "SCR01-A-10",
  "screen_id": "SCR01",
  "row_label": "A",
  "seat_number": 10,
  "is_available_for_sale": true
}
```

### 5.4 `ShowTime`
```json
{
  "showtime_id": "SHT001",
  "movie_id": "MOV001",
  "screen_id": "SCR01",
  "show_date": "2026-03-06",
  "start_time": "14:00",
  "end_time": "16:00",
  "format_type": "2D",
  "sales_status": "on_sale"
}
```

### 5.5 `SeatInventory`
```json
{
  "showtime_id": "SHT001",
  "seat_id": "SCR01-A-10",
  "inventory_status": "available",
  "reserved_by_reservation_id": null,
  "hold_expire_at": null
}
```

### 5.6 `Reservation`
```json
{
  "reservation_id": "RSV001",
  "reservation_number": "R202603060001",
  "customer_name": "山田太郎",
  "customer_contact": "090-xxxx-xxxx",
  "showtime_id": "SHT001",
  "reserved_seat_ids": ["SCR01-H-10", "SCR01-H-11"],
  "reservation_status": "reserved",
  "payment_status": "pending",
  "created_at": "2026-03-06T10:00:00+09:00",
  "updated_at": "2026-03-06T10:00:00+09:00"
}
```

### 5.7 `TheaterInfo`
```json
{
  "theater_id": "THT001",
  "theater_name": "サンプルシネマ",
  "business_hours": "09:00-23:00",
  "address": "東京都千代田区...",
  "access_information": "JR○○駅から徒歩5分",
  "notices": ["館内では上映中の通話は禁止です。"],
  "faq_items": [
    {
      "question": "飲食物の持ち込みは可能ですか？",
      "answer": "PoC では劇場ルールに従う想定です。"
    }
  ]
}
```

## 6. MCP ツール一覧
本 PoC では、業務要件の各機能に対して 1 ツールを定義する。

| 機能 | ツール名 | 概要 |
|---|---|---|
| 作品案内 | `search_movies` | 作品一覧・作品検索 |
| 上映スケジュール案内 | `search_showtimes` | 作品・日付で上映回検索 |
| 空席照会 | `check_seat_availability` | 上映回の空席状況確認 |
| 座席案内 | `suggest_seats` | 条件に応じた座席候補提示 |
| チケット予約受付 | `create_reservation` | 新規予約登録 |
| 予約確認 | `get_reservation` | 予約内容照会 |
| 予約変更 | `update_reservation` | 上映回・座席・人数変更 |
| 予約取消 | `cancel_reservation` | 予約取消 |
| 劇場案内・注意事項案内 | `get_theater_information` | 劇場基本情報・注意事項・FAQ 取得 |

## 7. `tools/list` 応答仕様
`tools/list` では、少なくとも以下の形式で各ツールを列挙する。

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "search_movies",
        "description": "上映作品の一覧または条件検索を行う",
        "inputSchema": {
          "type": "object",
          "properties": {
            "keyword": {
              "type": "string",
              "description": "作品名の検索キーワード"
            },
            "status": {
              "type": "string",
              "description": "上映状態。例: now_showing, coming_soon"
            }
          }
        }
      }
    ]
  }
}
```

## 8. `tools/call` 個別仕様

### 8.1 `search_movies`
#### 目的
上映作品の一覧取得または検索を行う。

#### 対応機能
- 機能1: 作品案内

#### リクエスト
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "search_movies",
    "arguments": {
      "keyword": "ドラマ",
      "status": "now_showing"
    }
  }
}
```

#### 入力項目
| 項目 | 型 | 必須 | 説明 |
|---|---|---|---|
| `keyword` | string | 任意 | 作品名の部分一致検索キーワード |
| `status` | string | 任意 | `now_showing` / `coming_soon` |

#### 正常レスポンス
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "success": true,
    "message": "作品一覧を取得しました。",
    "data": {
      "movies": [
        {
          "movie_id": "MOV001",
          "title": "Sample Movie",
          "description": "作品概要",
          "duration_minutes": 120,
          "genre": "Drama",
          "age_rating": "G",
          "release_start_date": "2026-03-01",
          "release_end_date": "2026-04-30",
          "status": "now_showing",
          "audio_type": "dubbed",
          "subtitle_type": "none"
        }
      ],
      "count": 1
    },
    "suggestions": []
  }
}
```

#### 主なエラー
- 検索条件不正: `-32602`
- 該当作品なし: `-32001`

### 8.2 `search_showtimes`
#### 目的
作品および日付条件に基づき上映回を検索する。

#### 対応機能
- 機能2: 上映スケジュール案内

#### リクエスト
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "search_showtimes",
    "arguments": {
      "movie_id": "MOV001",
      "show_date": "2026-03-06"
    }
  }
}
```

#### 入力項目
| 項目 | 型 | 必須 | 説明 |
|---|---|---|---|
| `movie_id` | string | 任意 | 対象作品 ID |
| `keyword` | string | 任意 | 作品名検索用キーワード |
| `show_date` | string | 任意 | `YYYY-MM-DD` 形式 |
| `date_label` | string | 任意 | `today` / `tomorrow` |

#### 正常レスポンス
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "success": true,
    "message": "上映スケジュールを取得しました。",
    "data": {
      "showtimes": [
        {
          "showtime_id": "SHT001",
          "movie_id": "MOV001",
          "movie_title": "Sample Movie",
          "screen_id": "SCR01",
          "screen_name": "Screen 1",
          "show_date": "2026-03-06",
          "start_time": "14:00",
          "end_time": "16:00",
          "format_type": "2D",
          "sales_status": "on_sale"
        }
      ],
      "count": 1
    },
    "suggestions": [
      "別日程も確認できます。"
    ]
  }
}
```

#### 主なエラー
- 日付形式不正: `-32602`
- 該当上映回なし: `-32001`

### 8.3 `check_seat_availability`
#### 目的
上映回の空席状況を確認する。

#### 対応機能
- 機能3: 空席照会

#### リクエスト
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "check_seat_availability",
    "arguments": {
      "showtime_id": "SHT001",
      "include_seat_map": true
    }
  }
}
```

#### 入力項目
| 項目 | 型 | 必須 | 説明 |
|---|---|---|---|
| `showtime_id` | string | 必須 | 対象上映回 ID |
| `include_seat_map` | boolean | 任意 | 座席一覧を含めるか |

#### 正常レスポンス
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "success": true,
    "message": "空席状況を取得しました。",
    "data": {
      "showtime_id": "SHT001",
      "availability_status": "available",
      "available_count": 48,
      "remaining_level": "many",
      "seats": [
        {
          "seat_id": "SCR01-H-10",
          "row_label": "H",
          "seat_number": 10,
          "inventory_status": "available"
        }
      ]
    },
    "suggestions": []
  }
}
```

#### 主なエラー
- `showtime_id` 不正: `-32602`
- 対象上映回なし: `-32001`

### 8.4 `suggest_seats`
#### 目的
希望条件に応じた候補座席を提示する。

#### 対応機能
- 機能4: 座席案内

#### リクエスト
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "suggest_seats",
    "arguments": {
      "showtime_id": "SHT001",
      "seat_count": 2,
      "needs_consecutive": true
    }
  }
}
```

#### 入力項目
| 項目 | 型 | 必須 | 説明 |
|---|---|---|---|
| `showtime_id` | string | 必須 | 対象上映回 ID |
| `seat_count` | integer | 必須 | 希望席数 |
| `needs_consecutive` | boolean | 任意 | 連番優先有無 |

#### 正常レスポンス
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "success": true,
    "message": "候補座席を取得しました。",
    "data": {
      "showtime_id": "SHT001",
      "seat_count": 2,
      "candidate_groups": [
        {
          "group_id": "CAND001",
          "seats": [
            {
              "seat_id": "SCR01-H-10",
              "row_label": "H",
              "seat_number": 10
            },
            {
              "seat_id": "SCR01-H-11",
              "row_label": "H",
              "seat_number": 11
            }
          ],
          "reason": "連番で利用しやすい候補座席です。"
        }
      ]
    },
    "suggestions": []
  }
}
```

#### 主なエラー
- 席数不正: `-32602`
- 候補なし: `-32001`

### 8.5 `create_reservation`
#### 目的
新規予約を登録する。

#### 対応機能
- 機能5: チケット予約受付

#### リクエスト
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "create_reservation",
    "arguments": {
      "showtime_id": "SHT001",
      "customer_name": "山田太郎",
      "customer_contact": "090-1111-2222",
      "seat_ids": ["SCR01-H-10", "SCR01-H-11"],
      "ticket_count": 2,
      "confirmation_acknowledged": true
    }
  }
}
```

#### 入力項目
| 項目 | 型 | 必須 | 説明 |
|---|---|---|---|
| `showtime_id` | string | 必須 | 対象上映回 ID |
| `customer_name` | string | 必須 | 代表者名 |
| `customer_contact` | string | 必須 | 連絡先 |
| `seat_ids` | array[string] | 必須 | 予約対象座席 ID 配列 |
| `ticket_count` | integer | 必須 | 予約人数 |
| `confirmation_acknowledged` | boolean | 必須 | 利用者確認済みか |

#### 正常レスポンス
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "success": true,
    "message": "予約を登録しました。",
    "data": {
      "reservation": {
        "reservation_id": "RSV001",
        "reservation_number": "R202603060001",
        "customer_name": "山田太郎",
        "customer_contact": "090-1111-2222",
        "showtime_id": "SHT001",
        "reserved_seat_ids": ["SCR01-H-10", "SCR01-H-11"],
        "reservation_status": "reserved",
        "payment_status": "pending",
        "created_at": "2026-03-06T10:00:00+09:00",
        "updated_at": "2026-03-06T10:00:00+09:00"
      }
    },
    "suggestions": [
      "予約番号を控えてください。"
    ]
  }
}
```

#### 主なエラー
- 必須入力不足: `-32602`
- 確認未了: `-32602`
- 座席重複・満席: `-32009`

### 8.6 `get_reservation`
#### 目的
予約番号から予約内容を照会する。

#### 対応機能
- 機能6: 予約確認

#### リクエスト
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_reservation",
    "arguments": {
      "reservation_number": "R202603060001"
    }
  }
}
```

#### 入力項目
| 項目 | 型 | 必須 | 説明 |
|---|---|---|---|
| `reservation_number` | string | 必須 | 予約番号 |

#### 正常レスポンス
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "success": true,
    "message": "予約内容を取得しました。",
    "data": {
      "reservation": {
        "reservation_number": "R202603060001",
        "reservation_status": "reserved",
        "customer_name": "山田太郎",
        "showtime": {
          "showtime_id": "SHT001",
          "movie_id": "MOV001",
          "movie_title": "Sample Movie",
          "show_date": "2026-03-06",
          "start_time": "14:00",
          "end_time": "16:00",
          "screen_id": "SCR01",
          "screen_name": "Screen 1"
        },
        "reserved_seat_ids": ["SCR01-H-10", "SCR01-H-11"],
        "ticket_count": 2,
        "payment_status": "pending"
      }
    },
    "suggestions": []
  }
}
```

#### 主なエラー
- 予約番号不正: `-32602`
- 予約なし: `-32001`

### 8.7 `update_reservation`
#### 目的
既存予約の上映回、座席、人数を変更する。

#### 対応機能
- 機能7: 予約変更

#### リクエスト
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "update_reservation",
    "arguments": {
      "reservation_number": "R202603060001",
      "new_showtime_id": "SHT002",
      "new_seat_ids": ["SCR01-J-10", "SCR01-J-11"],
      "new_ticket_count": 2,
      "change_reason": "時間変更希望",
      "confirmation_acknowledged": true
    }
  }
}
```

#### 入力項目
| 項目 | 型 | 必須 | 説明 |
|---|---|---|---|
| `reservation_number` | string | 必須 | 対象予約番号 |
| `new_showtime_id` | string | 任意 | 変更後上映回 ID |
| `new_seat_ids` | array[string] | 任意 | 変更後座席 ID 配列 |
| `new_ticket_count` | integer | 任意 | 変更後人数 |
| `change_reason` | string | 任意 | 変更理由 |
| `confirmation_acknowledged` | boolean | 必須 | 利用者確認済みか |

#### 正常レスポンス
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "success": true,
    "message": "予約を変更しました。",
    "data": {
      "reservation": {
        "reservation_number": "R202603060001",
        "showtime_id": "SHT002",
        "reserved_seat_ids": ["SCR01-J-10", "SCR01-J-11"],
        "reservation_status": "reserved",
        "payment_status": "pending",
        "updated_at": "2026-03-06T11:00:00+09:00"
      },
      "change_summary": {
        "showtime_changed": true,
        "seats_changed": true,
        "ticket_count_changed": false
      }
    },
    "suggestions": []
  }
}
```

#### 主なエラー
- 変更内容なし: `-32602`
- 対象予約なし: `-32001`
- 変更後座席競合: `-32009`

### 8.8 `cancel_reservation`
#### 目的
既存予約を取消する。

#### 対応機能
- 機能8: 予約取消

#### リクエスト
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "cancel_reservation",
    "arguments": {
      "reservation_number": "R202603060001",
      "cancel_reason": "予定変更",
      "confirmation_acknowledged": true
    }
  }
}
```

#### 入力項目
| 項目 | 型 | 必須 | 説明 |
|---|---|---|---|
| `reservation_number` | string | 必須 | 対象予約番号 |
| `cancel_reason` | string | 任意 | 取消理由 |
| `confirmation_acknowledged` | boolean | 必須 | 利用者確認済みか |

#### 正常レスポンス
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "success": true,
    "message": "予約を取消しました。",
    "data": {
      "reservation_number": "R202603060001",
      "reservation_status": "cancelled",
      "cancel_reason": "予定変更",
      "cancelled_at": "2026-03-06T11:30:00+09:00"
    },
    "suggestions": []
  }
}
```

#### 主なエラー
- 予約番号不正: `-32602`
- 対象予約なし: `-32001`
- 取消不可状態: `-32000`

### 8.9 `get_theater_information`
#### 目的
劇場の基本情報、注意事項、FAQ を返却する。

#### 対応機能
- 機能9: 劇場案内・注意事項案内

#### リクエスト
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_theater_information",
    "arguments": {
      "category": "faq",
      "keyword": "飲食"
    }
  }
}
```

#### 入力項目
| 項目 | 型 | 必須 | 説明 |
|---|---|---|---|
| `category` | string | 任意 | `overview` / `notices` / `faq` / `access` |
| `keyword` | string | 任意 | FAQ や注意事項の絞り込み |

#### 正常レスポンス
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "success": true,
    "message": "劇場情報を取得しました。",
    "data": {
      "theater": {
        "theater_id": "THT001",
        "theater_name": "サンプルシネマ",
        "business_hours": "09:00-23:00",
        "address": "東京都千代田区...",
        "access_information": "JR○○駅から徒歩5分",
        "notices": ["館内では上映中の通話は禁止です。"],
        "faq_items": [
          {
            "question": "飲食物の持ち込みは可能ですか？",
            "answer": "PoC では劇場ルールに従う想定です。"
          }
        ]
      }
    },
    "suggestions": []
  }
}
```

#### 主なエラー
- カテゴリ不正: `-32602`
- 該当情報なし: `-32001`

## 9. 入力バリデーション方針
- 日付は `YYYY-MM-DD` 形式とする。
- 時刻は `HH:MM` 形式とする。
- `seat_ids` は同一上映回に属する座席であること。
- `ticket_count` は 1 以上とする。
- `seat_ids.length` と `ticket_count` は一致することを基本とする。
- 予約変更、取消、新規予約の確定操作では `confirmation_acknowledged=true` を要求する。

## 10. 業務エラー応答例
### 10.1 満席エラー
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32009,
    "message": "指定された座席は既に予約済みです。",
    "data": {
      "error_type": "seat_conflict",
      "showtime_id": "SHT001",
      "seat_ids": ["SCR01-H-10", "SCR01-H-11"],
      "suggestions": [
        "中央付近の別候補座席を案内できます。"
      ]
    }
  }
}
```

### 10.2 予約番号未存在エラー
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32001,
    "message": "指定された予約番号は存在しません。",
    "data": {
      "error_type": "reservation_not_found",
      "reservation_number": "R202603060999"
    }
  }
}
```

## 11. ツール命名方針
- 動詞 + 名詞の英語スネークケースで統一する。
- 業務上の責務が明確な単位で 1 ツールを定義する。
- 参照系と更新系を分離する。

## 12. 今後の実装への引き継ぎ事項
後続の実装では、本仕様に基づき以下を具体化する。

- Azure Functions 上の各ツール実装
- `tools/list` の動的生成または固定定義
- リクエスト入力検証
- 予約競合時の簡易排他
- 永続化方式
- テストデータ作成

以上。