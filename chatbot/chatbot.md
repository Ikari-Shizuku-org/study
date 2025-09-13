# 研修用チャットボット要件定義

## 目的
- 研修参加者がTeams上で質問できるチャットボットを提供し、事前に用意した文書やFAQから自動回答する。
- 本格運用時は、質問・回答の蓄積とナレッジの継続的な拡充を目指す。

## 技術構成（MVP）
- Microsoft Teams Bot（C#）
- Azure Cognitive Search
- 事前に用意したFAQやナレッジ文書（PDF/テキスト、Blob Storage格納）

## 機能要件
1. **ユーザー質問受付**
	- Teams Botにユーザーが自然言語で質問（例：「VPNつながらない」）
2. **質問解析・検索**
	- Botが質問内容を解析し、Azure Cognitive Searchに問い合わせ
	- ナレッジ文書（Blob Storage上のPDFやテキスト）を全文検索
3. **自動回答**
	- 該当する情報があれば、Botが要約やリンク付きで回答
	- 該当がなければ、メンターにエスカレーション
4. **メンター対応**
	- メンターがチャット上で直接回答、または管理UIから回答を登録
5. **やりとりの記録**
	- 質問内容、Botの回答（あれば）、メンターの回答、日時、ユーザー情報をDB（Azure SQL Database）に保存
6. **ナレッジ更新**
	- 運用担当がDBに溜まったQ&Aをレビューし、有効なものをテキストファイルやFAQとしてBlobに追加
	- Cognitive Searchのインデックスを更新
	- 翌年以降も同じ質問に自動回答できるようナレッジを拡充

## 非機能要件
- セキュリティ：Teams認証、Azure権限管理
- 拡張性：ナレッジ追加・インデックス更新が容易
- 保守性：Q&Aのレビュー・管理がしやすいUI

## 使用技術
- Azure SQL Database
- C#
- Teams Bot Framework
- Azure OpenAI + Cognitive Search
- Azure Blob Storage

---

# チャットボットMVPフロー（Mermaid）

```mermaid
flowchart TD
    A[ユーザーがTeams Botに質問] --> B[Botが質問を解析]
    B --> C[Azure Cognitive Searchでナレッジ検索]
    C -->|該当あり| D[Botが自動回答（要約・リンク付き）]
    C -->|該当なし| E[メンターにエスカレーション]
    E --> F[メンターがチャットまたは管理UIで回答]
    D & F --> G[やりとりをDBに保存]
    G --> H[運用担当がQ&Aをレビュー]
    H --> I[有効なQ&Aをナレッジに追加・インデックス更新]
    I --> C
```

---

# チャットボットシステム ER図

以下のERダイアグラムは、研修用チャットボットシステムのデータベース設計を表しています。

```mermaid
erDiagram
    User {
        int userId PK
        string userName
        string teamsId
        string email
        date createdAt
    }
    
    Question {
        int questionId PK
        int userId FK
        string content
        datetime timestamp
        boolean isAnswered
        string status
    }
    
    Answer {
        int answerId PK
        int questionId FK
        string content
        datetime timestamp
        string source
        boolean isAutomatic
        int mentorId FK
    }
    
    Mentor {
        int mentorId PK
        string userName
        string email
        boolean isActive
    }
    
    KnowledgeDocument {
        int documentId PK
        string title
        string filePath
        string documentType
        datetime uploadedAt
        datetime lastIndexedAt
    }
    
    FAQItem {
        int faqId PK
        string question
        string answer
        int categoryId FK
        boolean isActive
        datetime createdAt
        datetime updatedAt
    }
    
    Category {
        int categoryId PK
        string name
        string description
    }
    
    User ||--o{ Question : "asks"
    Question ||--o{ Answer : "has"
    Mentor ||--o{ Answer : "provides"
    KnowledgeDocument ||--o{ Answer : "sources"
    Category ||--o{ FAQItem : "contains"
    FAQItem ||--o{ Answer : "references"
```

## エンティティの説明

### User (ユーザー)
- Teams上でチャットボットに質問するユーザー
- Teams IDと基本情報を保持

### Question (質問)
- ユーザーが投げた質問内容
- 質問日時、回答状態などを記録

### Answer (回答)
- 質問に対する回答
- 自動回答かメンター回答かを区別
- 回答の出典情報も保持

### Mentor (メンター)
- 自動回答できない質問に対応するメンター
- メンターの基本情報を管理

### KnowledgeDocument (ナレッジ文書)
- Blob Storageに保存されたPDF/テキスト文書
- 検索インデックス更新情報も保持

### FAQItem (FAQ項目)
- よくある質問と回答のセット
- カテゴリで分類可能

### Category (カテゴリ)
- FAQ項目などを分類するためのカテゴリ

---

# シーケンス図（UML）

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant Bot as Teams Bot
    participant Search as Cognitive Search
    participant Blob as Blob Storage
    participant Mentor as メンター
    participant DB as Azure SQL DB
    participant Admin as 管理UI/運用担当

    User->>Bot: 質問
    Bot->>Search: 質問解析・検索
    Search->>Blob: ナレッジ文書全文検索
    Search-->>Bot: 検索結果
    alt 該当あり
        Bot->>User: 自動回答
        Bot->>DB: やりとり記録
    else 該当なし
        Bot->>Mentor: エスカレーション
        Mentor->>User: 回答
        Mentor->>DB: 回答記録
    end
    Admin->>DB: Q&Aレビュー
    Admin->>Blob: ナレッジ追加/更新
    Admin->>Search: インデックス更新
```
