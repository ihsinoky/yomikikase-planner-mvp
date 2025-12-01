# 絵本読み聞かせ活動支援システム – システム設計書

## 1. 目的と概要

幼稚園における絵本読み聞かせ活動を支援するため、  
保護者がアンケートに回答し、運営者が日程を管理し、  
絵本の履歴を蓄積できる統合システムを設計する。

現状の Googleフォーム、LINEグループ、Excel に散在する情報をまとめ、

- 保護者の手間軽減  
- 管理者の集計と調整作業の効率化  
- 絵本履歴のナレッジ化  
- 個人情報の年度単位でのクリア  

を実現する。

---

## 2. システム全体アーキテクチャ

### 2.1 構成方針

- **フロントエンド：軽量（HTML + CSS + Alpine.js + LIFF SDK）**
  - SPAフレームワークは使わず、ページ構造をシンプルにする
  - 動的UIは Alpine.js が担当（フィルタ・リスト更新など）
  - LIFF SDK で LINE userId を取得し、ユーザー識別を行う

- **バックエンド：Hono（TypeScript）**
  - REST API を提供し、LIFF フロントから叩く
  - サーバレス運用に向く（Cloudflare Workers / Bun / Nodeなどで稼働可能）

- **データベース：Cloudflare D1 または SQLite / Postgres**
  - 年度単位でデータを持つ要件に適した軽量 DB を採用
  - D1 はバックアップ（Time Travel）が標準で利用可能

### 2.2 アーキテクチャ図（概念）

```
[LINE アプリ]
│（LIFF起動）
▼
[フロントエンド：HTML + Alpine.js + LIFF SDK]
│ fetch()
▼
[Hono API Server]
│ SQL
▼
[Cloudflare D1（SQL）]
```


---

## 3. 機能設計

### 3.1 フロント（LIFF）

#### 1. プロフィール登録
- 名前
- 学年
- クラス
- LINE userId（LIFF SDK で取得）

#### 2. アンケート回答
- 各候補日ごとの「参加可否」を選択
- API に回答を保存

#### 3. 絵本管理（将来追加）
- ISBN入力 → 書誌情報取得API呼び出し

---

### 3.2 管理 Web アプリ

#### 1. アンケート作成
- 年度
- タイトル
- 説明文
- 開催候補日（複数）

#### 2. 回答一覧
- 学年／クラスフィルタ
- CSV エクスポート

#### 3. 日程確定
- 学年フィルタ → クラスフィルタ → ユーザー選択
- 確定日程登録（複数人可）

#### 4. 絵本管理
- ISBNベースの書誌情報登録
- 読み聞かせ記録登録

---

## 4. データモデル

### 4.1 User
| Column | Type | Note |
|--------|------|------|
| id | string | LINE userId |
| created_at | datetime |  |

### 4.2 UserYearProfile
| Column | Type |
|--------|------|
| id | pk |
| user_id | fk(User) |
| year | number |
| name | string |
| grade | string |
| class | string |

### 4.3 Survey
| id | year | title | description |

### 4.4 SurveyResponse
| id | survey_id | user_id | response(JSON) | created_at |

### 4.5 ConfirmedEvent
| id | date | year | grade | class | participants(array<user_id>) |

### 4.6 Book
| isbn | title | author | publisher | image_url |

### 4.7 BookReadingRecord
| id | date | year | grade | class | book_isbn | note |

---

## 5. API 設計（Hono）

### `/auth/login`
- LIFFから送られるIDトークンを検証
- ユーザー登録 or ログイン

### `/profiles`
- GET：年度・学年・クラスでフィルタ
- POST：プロフィール登録

### `/surveys`
- POST：アンケート作成
- GET：一覧取得

### `/surveys/:id/responses`
- POST：回答登録
- GET：結果取得

### `/events`
- POST：日程確定
- GET：一覧取得

### `/books`
- POST：ISBN 登録
- GET：一覧取得

### `/books/records`
- POST：読み聞かせ記録保存
- GET：一覧取得

---

## 6. 非機能要件

### パフォーマンス
- 数十名規模の同時利用を想定
- API 応答 3 秒以内

### セキュリティ
- LIFF ID トークン検証
- 管理画面は ID/PW or OAuth2
- 年度終了後の個人データ削除可能

### 可用性
- Cloudflare Workers + D1 でサーバレス運用可
- D1 の Time Travel により 30 日の PITR（ポイントインタイムリカバリ）

---

## 7. フロントエンド UI モック（HTML + Alpine.js）

### 7.1 日程確定 UI モック（学年フィルタあり）

```html
<div x-data="scheduleApp()">

  <!-- 学年フィルタ -->
  <select x-model="selectedGrade">
    <option value="">全学年</option>
    <template x-for="g in grades">
      <option :value="g" x-text="g"></option>
    </template>
  </select>

  <!-- 候補日一覧 -->
  <div class="dates">
    <template x-for="d in filteredDates()" :key="d">
      <div>
        <span x-text="d"></span>
        <button @click="selectDate(d)">決定</button>
      </div>
    </template>
  </div>

  <!-- 選択後の参加者選択UI -->
  <div x-show="selectedDate" class="panel">
    <h3>選択日程：<span x-text="selectedDate"></span></h3>

    <template x-for="m in filteredMembers()" :key="m.id">
      <label>
        <input type="checkbox" :value="m.id" x-model="selectedMembers">
        <span x-text="m.name"></span>
      </label>
    </template>

    <button @click="confirm()">確定登録</button>
  </div>

</div>

<script>
function scheduleApp() {
  return {
    grades: ['年少','年中','年長'],
    selectedGrade: '',
    dates: ['2025-05-10','2025-06-12','2025-07-14'],
    members: [
      {id:1,name:'Yamada',grade:'年長'},
      {id:2,name:'Suzuki',grade:'年長'},
      {id:3,name:'Sato',grade:'年中'},
    ],

    selectedDate: null,
    selectedMembers: [],

    filteredDates() {
      return this.dates;
    },

    filteredMembers() {
      if (!this.selectedGrade) return this.members;
      return this.members.filter(m => m.grade === this.selectedGrade);
    },

    selectDate(d) {
      this.selectedDate = d;
      this.selectedMembers = [];
    },

    async confirm() {
      await fetch('/events', {
        method: 'POST',
        body: JSON.stringify({
          date: this.selectedDate,
          members: this.selectedMembers
        })
      });
      alert("登録しました");
    }
  }
}
</script>
```

---

## 8. 今後の拡張ポイント

- 自動割当アルゴリズム（Python）をAPI化
- 絵本バーコードスキャン（LIFF カメラ API）
- 管理画面の UI 強化（班分け・年間レポート）

