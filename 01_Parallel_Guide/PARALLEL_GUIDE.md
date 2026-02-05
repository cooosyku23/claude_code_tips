# Claude Code 並列化ガイド

複数のClaude Codeセッションを同時に動かして、開発スピードを上げるためのガイドです。

---

## 目次

1. [このガイドについて](#このガイドについて)
2. [並列化で得られるもの、失うもの](#並列化で得られるもの失うもの)
3. [Git worktreeの基本操作](#git-worktreeの基本操作)
4. [競合を防ぐ設計](#競合を防ぐ設計)
5. [実践例：Todoアプリを並列開発する](#実践例todoアプリを並列開発する)
6. [Claudeへの指示テンプレート集](#claudeへの指示テンプレート集)
7. [トラブルシューティング](#トラブルシューティング)
8. [チーム開発での活用](#チーム開発での活用)
9. [補足：セッション管理とCLAUDE.md](#補足セッション管理とclaudemd)
10. [参考リンク](#参考リンク)

---

## このガイドについて

### 対象者

- Claude Codeを使い始めた人
- Gitの基本操作（clone, commit, push）ができる人
- ターミナル（コマンドを入力してコンピュータを操作する画面）を触ったことがある人

**前提環境：** Mac / Linux（このガイドのコマンドはMac/Linux向けです）

### このガイドで学べること

- Git worktree（1つのリポジトリから複数の作業フォルダを作る機能）の概念と基本操作
- 並列化のメリット・デメリットと判断基準
- 競合（同じファイルを同時編集して起きる問題）を防ぐ設計
- Claudeへの効果的な指示の出し方
- トラブル発生時の復旧方法

### 公式ドキュメントとの違い

| 項目 | 公式ドキュメント | このガイド |
|------|------------------|------------|
| worktree基本操作 | コマンド2つ、20行 | 詳細な手順 |
| 競合防止 | Tipsで1行 | 設計原則として詳述 |
| メリット・デメリット | 言及なし | 判断基準を明記 |
| トラブル対応 | なし | 具体的な復旧手順 |
| 実践例 | なし | Todoアプリで具体化 |

### 事前準備

始める前に、以下を確認してください。

```bash
# Gitがインストールされているか
git --version
# → git version 2.39.0 のように表示されればOK

# Claude Codeがインストールされているか
claude --version
# → バージョン番号が表示されればOK

# 練習用のGitリポジトリがあるか（なければ作成）
mkdir my-practice-project
cd my-practice-project
git init
echo "# Practice" > README.md
git add .
git commit -m "Initial commit"
```

---

## 並列化で得られるもの、失うもの

並列化は強力なテクニックですが、すべての状況で最適とは限りません。

### 得られるもの

```
従来の作業スタイル：
┌─────────────────┐
│ タスクA（20分） │
└─────────────────┘
          ↓
┌─────────────────┐
│ タスクB（15分） │
└─────────────────┘
          ↓
┌─────────────────┐
│ タスクC（10分） │
└─────────────────┘

合計：45分


並列化した作業スタイル：
┌─────────────────┐
│ タスクA         │ ← Claudeが自動で作業中
├─────────────────┤
│ タスクB         │ ← その間に別の作業
├─────────────────┤
│ タスクC         │ ← さらに別の作業
└─────────────────┘

合計：約25分（最も長いタスク + 切り替えコスト）
```

**具体的なメリット：**
- 待ち時間の有効活用（テスト実行中、ビルド中に別作業）
- 複数機能の同時進行
- PRレビュー待ち中の継続作業

### 失うもの

| 項目 | 影響 |
|------|------|
| APIトークン | セッション数に比例して消費が増加 |
| レートリミット | Pro/Maxプランの上限に達しやすくなる |
| 認知負荷 | 複数コンテキストの切り替えが必要 |
| セットアップ時間 | 各worktreeで環境初期化が必要（後述） |

### 判断基準

**並列化が割に合うケース：**

```
□ 1タスクに30分以上かかる
  （worktree作成・環境初期化に5〜10分かかるため、
   それより短いタスクでは並列化のメリットが薄い）
□ ビルドやテストの待ち時間が発生する
□ 別々のフォルダ（frontend / backend など）を触る独立したタスク
□ PRレビュー待ちで手が空く
```

**並列化が割に合わないケース：**

```
□ 10分で終わる小さな修正
□ 同じファイルを編集する複数タスク（競合リスク）
□ APIコストを気にする必要がある状況
□ 初めてのプロジェクトで全体像を把握したい段階
```

**判断フローチャート：**

```
タスクはいくつある？
    │
    ├─ 1つだけ → 並列化不要
    │
    └─ 2つ以上 → 同じファイルを編集する？
                    │
                    ├─ はい → 並列化は避ける（競合リスク）
                    │
                    └─ いいえ → 1タスク30分以上かかる？
                                  │
                                  ├─ はい → 並列化が効果的
                                  │
                                  └─ いいえ → 順次作業でOK
```

迷ったら、まず**2つのworktree**で試してみてください。

---

## Git worktreeの基本操作

### worktreeとは

Git worktreeは「同じリポジトリ（Gitで管理されているプロジェクト）を複数の場所で同時に開く」ための仕組みです。

普通にフォルダをコピーすると：
- 容量が2倍、3倍になる
- Git履歴が別々になり、管理が大変

Git worktreeを使うと：
- 必要最小限のファイルだけ複製
- Git履歴は1つを共有
- ブランチ（作業の分岐）の切り替え不要で並列作業

### 基本の3コマンド

```bash
# 作る
git worktree add <フォルダ> -b <ブランチ名> main

# 確認する
git worktree list

# 削除する
git worktree remove <フォルダ>
```

### 実際の操作手順

#### 1. worktreeを作成する

```bash
# プロジェクトのフォルダに移動
cd ~/my-practice-project

# worktreeを作成（1つ上の階層に作る）
git worktree add ../my-practice-project-a -b work-a main
```

**コマンドの意味：**

| 部分 | 意味 |
|------|------|
| `git worktree add` | 新しいworktreeを作る |
| `../my-practice-project-a` | 作成先のフォルダ |
| `-b work-a` | `work-a`という新しいブランチを作る |
| `main` | `main`ブランチを元にする |

成功すると以下のように表示されます：

```
Preparing worktree (new branch 'work-a')
HEAD is now at abc1234 Initial commit
```

#### 2. 作成されたか確認する

```bash
git worktree list
```

```
/Users/yourname/my-practice-project    abc1234 [main]
/Users/yourname/my-practice-project-a  abc1234 [work-a]
```

2つ表示されていれば成功です。

#### 3. worktreeでClaude Codeを使う

```bash
# 新しいworktreeに移動
cd ../my-practice-project-a

# Claude Codeを起動
claude
```

別のターミナルウィンドウで、元のプロジェクトでもClaudeを起動できます：

```bash
cd ~/my-practice-project
claude
```

**これで2つのClaudeが同時に動いています：**

```
ターミナル1                    ターミナル2
┌────────────────────┐       ┌────────────────────┐
│ my-practice-project│       │ my-practice-project-a│
│ (mainブランチ)      │       │ (work-aブランチ)     │
│                    │       │                    │
│ Claude セッション1  │       │ Claude セッション2  │
└────────────────────┘       └────────────────────┘
```

#### 4. 2つ目のworktreeを追加する

```bash
cd ~/my-practice-project
git worktree add ../my-practice-project-b -b work-b main
```

#### 5. 作業が終わったらworktreeを削除する

```bash
# 変更をコミット・プッシュしてから
cd ../my-practice-project-a
git add .
git commit -m "Complete feature A"
git push origin work-a

# メインプロジェクトに戻って削除
cd ~/my-practice-project
git worktree remove ../my-practice-project-a
```

**注意：** worktreeを削除してもブランチは残ります。ブランチも削除したい場合は `git branch -d work-a` を実行してください。

---

## 競合を防ぐ設計

並列作業で最も避けたいのは「競合（コンフリクト）」です。同じファイルの同じ場所を別々に編集すると、マージ（統合）時にGitが「どちらを採用すべきか分からない」状態になります。

### 競合を防ぐ3原則

#### 原則1：フォルダ単位で担当を分ける

```
                 良い分け方
┌─────────────────────────────────────────┐
│                                         │
│  worktree-a           worktree-b        │
│  ┌──────────┐         ┌──────────┐      │
│  │ frontend/│         │ backend/ │      │
│  │ を編集   │         │ を編集   │      │
│  └──────────┘         └──────────┘      │
│                                         │
│  → 別々のフォルダなので競合しない       │
└─────────────────────────────────────────┘

                 悪い分け方
┌─────────────────────────────────────────┐
│                                         │
│  worktree-a           worktree-b        │
│  ┌──────────┐         ┌──────────┐      │
│  │ App.tsx  │         │ App.tsx  │      │
│  │ を編集   │         │ を編集   │      │
│  └──────────┘         └──────────┘      │
│                                         │
│  → 同じファイル！マージ時に競合する     │
└─────────────────────────────────────────┘
```

Claudeに最初に伝えておくと、守ってくれます：

```bash
# feature worktreeで
claude
> このworktreeではfrontend/フォルダの作業だけ行います

# fix worktreeで
claude
> このworktreeではbackend/フォルダの作業だけ行います
```

#### 原則2：共通ファイルは1箇所でのみ編集する

以下のファイルは複数worktreeで編集すると競合しやすいです：

- `package.json`
- `README.md`
- 設定ファイル（`.env`, `config.ts` など）

これらは**メインのworktreeでのみ編集する**と決めておきましょう。

#### 原則3：観察用worktreeは編集禁止

「見るだけ」専用のworktreeを用意すると、安全にコードを確認できます：

```bash
# 観察専用worktreeを作成（ブランチは作らずmainを直接使用）
git worktree add ../project-watch main
```

```bash
cd ~/project-watch
claude
> このworktreeは読み取り専用です。コードの調査とログ確認だけ行い、
> ファイルの編集やコミットは絶対にしないでください。
```

### 推奨するworktree構成

最初は以下の3つ構成がおすすめです：

```bash
git worktree add ../project-feature -b feature main
git worktree add ../project-fix -b fix main
git worktree add ../project-watch main
```

| Worktree | 役割 | 何をするか |
|----------|------|-----------|
| project（元） | メイン | PRマージ後の確認、リリース作業 |
| project-feature | 機能開発 | 新しい機能を作る（時間がかかるタスク） |
| project-fix | 修正 | バグ修正、小さな改善（すぐ終わるタスク） |
| project-watch | 観察専用 | ログ確認、コード調査。**編集しない** |

### こまめにリベースする

長時間作業する前に、最新の状態を取り込みましょう：

```bash
git fetch origin
git rebase origin/main
```

これで「知らない間に他の人（または別のworktree）が同じファイルを変更していた」という事態を防げます。

---

## 実践例：Todoアプリを並列開発する

シンプルなTodoアプリを例に、実際の並列開発の流れを説明します。

### プロジェクト構成

```
todo-app/
├── frontend/          ← 画面（React）
│   ├── components/
│   └── pages/
├── backend/           ← API（Node.js）
│   ├── routes/
│   └── models/
├── docs/              ← ドキュメント
└── package.json
```

### worktreeのセットアップ

```bash
cd ~/todo-app

# 3つのworktreeを作成
git worktree add ../todo-app-ui -b feature/ui main
git worktree add ../todo-app-api -b feature/api main
git worktree add ../todo-app-watch main
```

### 各worktreeの役割

| Worktree | 担当フォルダ | Claudeへの指示 |
|----------|-------------|---------------|
| todo-app-ui | frontend/ | 「タスク一覧画面を作って」 |
| todo-app-api | backend/ | 「タスクを保存するAPIを作って」 |
| todo-app-watch | （編集しない） | 「コードの構造を説明して」 |

### 実際の作業の流れ

**ターミナル1（todo-app-ui）：**
```bash
cd ~/todo-app-ui
claude

あなた：タスク一覧を表示するコンポーネントを作ってください。
        frontend/components/ に作成してください。

Claude：（作業開始 → 自動で進む）
```

**ターミナル2（todo-app-api）：**
Claudeが画面を作っている間に、別のターミナルで：
```bash
cd ~/todo-app-api
claude

あなた：タスクを追加・取得するREST APIを作ってください。
        backend/routes/ に作成してください。

Claude：（作業開始 → 自動で進む）
```

**ターミナル3（todo-app-watch）：**
両方の進捗を確認したいとき：
```bash
cd ~/todo-app-watch
claude

あなた：frontend/とbackend/の現在の構造を説明してください。
        ファイルは編集しないでください。

Claude：（コードを読んで説明してくれる）
```

### 時間比較（目安）

| 作業 | 順次作業の場合 | 並列作業の場合 |
|------|----------------|----------------|
| フロントエンド実装 | 20分 | 20分 |
| バックエンド実装 | 15分 | （同時進行） |
| テスト作成 | 10分 | （同時進行） |
| **合計** | **45分** | **約25分** |

※ 並列作業の場合、最も長いタスク（20分）＋ 切り替え・確認の時間（5分）
※ worktree作成・環境初期化の時間（5〜10分）は含まず
※ 実際の時間はプロジェクトの複雑さにより異なります

### この例のポイント

1. **フォルダが分かれている** — frontend/ と backend/ で競合しない
2. **お互いに依存しない** — UIとAPIは独立して開発できる
3. **観察用がある** — 安全に全体を確認できる場所がある

---

## Claudeへの指示テンプレート集

各worktreeでClaudeを起動したら、最初に「役割」を伝えると効果的です。

### 機能開発用

```
このworktreeでは [機能名] の開発を行います。

作業範囲：
- [フォルダ名]/ 以下のファイルのみ編集してください
- 他のフォルダは触らないでください

目標：
- [具体的な目標を書く]

では、[最初のタスク] から始めてください。
```

**使用例：**
```
このworktreeでは認証機能の開発を行います。

作業範囲：
- src/features/auth/ 以下のファイルのみ編集してください
- 他のフォルダは触らないでください

目標：
- ログイン・ログアウト機能を実装する
- JWTトークンで認証する

では、ログイン画面のコンポーネント作成から始めてください。
```

### バグ修正用

```
このworktreeではバグ修正を行います。

対象のバグ：
- [バグの内容を説明]
- [再現手順があれば書く]

方針：
- まず原因を調査してください
- 修正は最小限の変更で行ってください
- 修正後、動作確認の方法を教えてください
```

**使用例：**
```
このworktreeではバグ修正を行います。

対象のバグ：
- タスクを削除しても画面に残り続ける
- 再読み込みすると消える

方針：
- まず原因を調査してください
- 修正は最小限の変更で行ってください
- 修正後、動作確認の方法を教えてください
```

### 観察・調査用（編集禁止）

```
このworktreeは読み取り専用です。

重要なルール：
- ファイルの編集は絶対にしないでください
- コミットも行わないでください
- 調査と説明のみ行ってください

調査してほしいこと：
- [調査内容を書く]
```

**使用例：**
```
このworktreeは読み取り専用です。

重要なルール：
- ファイルの編集は絶対にしないでください
- コミットも行わないでください
- 調査と説明のみ行ってください

調査してほしいこと：
- src/api/ のフォルダ構造を説明してください
- データの流れを図で示してください
- 改善できそうな箇所があれば教えてください
```

### 実験・プロトタイプ用

```
このworktreeは実験用です。

目的：
- [試したいことを書く]

ルール：
- 失敗してもOKです
- 動くものを素早く作ることを優先してください
- コードの品質は後で整えます

試したいこと：
- [具体的な実験内容]
```

**使用例：**
```
このworktreeは実験用です。

目的：
- 新しいアニメーションライブラリを試したい

ルール：
- 失敗してもOKです
- 動くものを素早く作ることを優先してください
- コードの品質は後で整えます

試したいこと：
- Framer Motionでタスクカードのアニメーションを作る
- 使い勝手が良ければ本番に採用する
```

### テンプレート使用のコツ

1. **最初に役割を宣言する** — Claudeが範囲外の作業をしにくくなる
2. **フォルダを明示する** — 競合防止に効果的
3. **目標を具体的に** — Claudeが的確に動いてくれる

---

## トラブルシューティング

### よくあるエラー

#### 「'work-a' is already checked out」

```
fatal: 'work-a' is already checked out at '/Users/yourname/my-practice-project-a'
```

**原因：** そのブランチは既に別のworktreeで使われています。

**解決策：** 別のブランチ名を使ってください。

```bash
git worktree add ../my-practice-project-c -b work-c main
```

#### 「not a git repository」

```
fatal: not a git repository
```

**原因：** Gitリポジトリではないフォルダで実行しています。

**解決策：** プロジェクトのフォルダに移動してから実行してください。

```bash
cd ~/my-practice-project
git worktree add ...
```

#### 「directory already exists」

```
fatal: '../my-practice-project-a' already exists
```

**原因：** そのフォルダが既に存在しています。

**解決策：**

```bash
# 方法1：別の名前を使う
git worktree add ../my-practice-project-a2 -b work-a2 main

# 方法2：既存のフォルダを削除してから再実行
rm -rf ../my-practice-project-a
git worktree prune
git worktree add ../my-practice-project-a -b work-a main
```

#### worktreeを削除できない

```
fatal: cannot remove: Directory not empty
```

**解決策：** 強制削除を使います。

```bash
git worktree remove --force ../my-practice-project-a
```

#### フォルダを直接削除してしまった場合

`rm -rf` でworktreeフォルダを消してしまった場合：

```bash
git worktree prune   # 管理情報をクリーンアップ
git worktree list    # 正しい状態に戻る
```

### 競合（コンフリクト）が起きてしまった

**症状：** `git merge` や `git rebase` で「CONFLICT」と表示された。ファイルに `<<<<<<<` や `>>>>>>>` という記号が入っている。

**これは何が起きているのか：**

```
同じファイルの同じ場所を、2つのブランチで別々に編集した結果です。

worktree-a で編集：「こんにちは」→「Hello」に変更
worktree-b で編集：「こんにちは」→「やあ」に変更

両方をmainにマージしようとすると...
Gitは「どっちを採用すればいいの？」と分からなくなります。
```

**復旧手順：**

```bash
# 1. 競合しているファイルを確認
git status
# → "both modified: src/greeting.ts" のように表示される

# 2. ファイルを開いて、競合箇所を確認
# 以下のような表示になっている：
```

```
<<<<<<< HEAD
const greeting = "Hello";
=======
const greeting = "やあ";
>>>>>>> feature-b
```

```bash
# 3. どちらを採用するか決めて、手動で修正
#    （<<<<<<<, =======, >>>>>>> の行も削除する）
```

```typescript
// 例：Helloを採用する場合
const greeting = "Hello";
```

```bash
# 4. 修正したファイルをステージング
git add src/greeting.ts

# 5. マージを完了
git commit -m "Resolve conflict in greeting.ts"
```

### 間違えてmainで作業してしまった

**症状：** mainブランチで直接コードを書いてしまった。コミットもしてしまった。

**復旧手順：**

```bash
# 1. 現在のブランチを確認
git branch
# → * main  ← mainにいる！

# 2. 作業内容を新しいブランチに移す
git branch new-feature  # 今の状態で新しいブランチを作成

# 3. mainを元の状態に戻す
git reset --hard origin/main
# ⚠️ 注意：これはmainの変更を消します

# 4. 新しいブランチに移動して作業を続ける
git checkout new-feature

# 5. 必要ならworktreeを作成
cd ..
git worktree add ./my-project-feature new-feature
```

**まだコミットしていない場合：**

```bash
# 変更を一時退避（stash = 一時的に横に置いておく機能）
git stash

# 新しいブランチを作成して移動
git checkout -b new-feature

# 変更を戻す
git stash pop
```

### 何をしたか分からなくなった

**最終手段：クリーンな状態からやり直す**

```bash
# 1. メインプロジェクトに移動
cd ~/my-practice-project

# 2. すべてのworktreeを強制削除
git worktree list | grep -v "my-practice-project " | awk '{print $1}' | xargs -I {} git worktree remove {} --force 2>/dev/null

# 3. 残骸をクリーンアップ
git worktree prune

# 4. 未コミットの変更を破棄（注意！）
git checkout .
git clean -fd

# 5. mainを最新に
git checkout main
git pull origin main

# 6. 確認
git status
# → "nothing to commit, working tree clean" なら成功

# 7. 必要なworktreeを作り直す
git worktree add ../my-practice-project-feature -b feature main
```

⚠️ **注意：** この手順は未コミットの変更をすべて消します。大事な変更がある場合は、先にバックアップを取ってください。

---

## チーム開発での活用

チームで開発する場合、「他の人も同じファイルを触る可能性がある」点に注意が必要です。

### 基本ルール

1. **作業開始前にチームに共有する**
   ```
   「今から認証機能（src/auth/）を触ります。2時間くらいかかる予定です。」
   ```

2. **こまめにプッシュする**
   ```bash
   git add .
   git commit -m "WIP: 認証機能の途中経過"  # WIP = Work In Progress（作業中）
   git push origin feature/auth
   ```

3. **PRを小さく分ける**
   - 悪い例：「認証機能全部作りました！」→ 変更ファイル50個
   - 良い例：「ログイン画面」「ログインAPI」「ログアウト」→ 各5個以下

### worktree命名規則の例

チームで統一しておくと分かりやすくなります：

```bash
# 名前ベース
todo-app-tanaka-feature
todo-app-suzuki-fix

# チケット番号ベース
todo-app-TASK-123
todo-app-BUG-456
```

---

## 補足：セッション管理とCLAUDE.md

### セッション管理

Claude Codeには、セッションを再開する機能があります：

```bash
# 直前のセッションを再開
claude --continue

# 特定のセッションを再開
claude --resume
```

worktreeごとにセッションが分かれるため、各worktreeで `--continue` を使えば、そのworktreeでの作業を継続できます。

### CLAUDE.mdについて

`CLAUDE.md` はプロジェクトのルートに置く設定ファイルで、Claudeにプロジェクト固有の情報を伝えられます。

**注意点：** `CLAUDE.local.md`（ローカル用の設定）はworktree間で共有されません。各worktreeで同じ設定が必要な場合は、`CLAUDE.md`（リポジトリにコミットする版）を使うか、各worktreeに個別に配置してください。

詳細は公式ドキュメントを参照してください。

---

## 参考リンク

### 公式ドキュメント

- [Claude Code 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code)
- [Git Worktree 公式ドキュメント](https://git-scm.com/docs/git-worktree)

### 代替手段・関連ツール

worktree以外にも並列開発を支援するツールがあります：

- [GitButler](https://gitbutler.com/) — ブランチの視覚的な管理ツール
- [ccmanager](https://github.com/kbwo/ccmanager) — Claude Codeセッション管理ツール
- [crystal](https://github.com/stravu/crystal) — 並列worktreeでClaude Codeを管理するデスクトップアプリ
