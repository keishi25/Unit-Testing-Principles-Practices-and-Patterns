# 第8章：なぜ統合テストが必要か

## 8.1 統合テストとは何か

### 8.1.1 統合テストの役割

ユニットテストの3条件のうち1つでも満たさないテストは統合テスト。

- ユニットテスト → ドメインモデル・アルゴリズムを検証
- 統合テスト → コントローラー（外部依存との橋渡し）を検証

### 8.1.2 テストピラミッドの再確認

- ユニットテスト：エッジケースをできる限り網羅
- 統合テスト：ハッピーパス1件 ＋ ユニットテストで表現できないエッジケース
- 統合テストはユニットテストより遅く保守コストが高いが、回帰保護とリファクタリング耐性が高い

### 8.1.3 統合テスト vs Fail Fast

- **Fail Fastの原則**：予期しないエラーが発生したらすぐに処理を止める
- 誤った実行がすぐにアプリをクラッシュさせるエッジケースは、統合テストではなくFail Fastで対応できる
- 例：`CanChangeEmail()` のような事前条件チェック → 失敗時は即クラッシュ → 統合テスト不要

---

## 8.2 どの外部プロセス依存をそのままテストするか

### 8.2.1 外部プロセス依存の2種類

| 種類 | 定義 | 例 | 方針 |
|------|------|----|------|
| **管理依存** | 自アプリだけがアクセスする | アプリDB | 実インスタンスを使う |
| **非管理依存** | 外部から観測可能な通信 | SMTPサーバー、メッセージバス、他チームのAPI | モックに置き換える |

- 管理依存との通信は**実装の詳細**（外部から見えない）
- 非管理依存との通信は**観測可能な振る舞い**（外部システムが依存する）

### 8.2.2 両方の属性を持つ依存（SharedDB）

複数アプリが共有するDBは管理依存と非管理依存の両方の属性を持つ。

- 共有テーブル → 非管理依存的扱い（モック）
- 自アプリ専用テーブル → 管理依存的扱い（実インスタンス）

> 推奨：そもそもDBを複数アプリで共有する設計自体が問題。APIやメッセージバス経由が望ましい。

### 8.2.3 本番DBをテストで使えない場合

管理依存をモックにしてはいけない。モックにすると：
- リファクタリング耐性が下がる
- ユニットテストと同程度の価値しか生まない

→ **統合テスト自体を書かない**（ドメインモデルのユニットテストに集中）

---

## 8.3 統合テストの実例（CRMシステム）

### 8.3.1 テストするシナリオ

最長のハッピーパス：企業メールから非企業メールへの変更

副作用：
- DB：ユーザータイプとメール更新、会社の従業員数更新
- メッセージバス：メッセージ送信

### 8.3.2 DBとメッセージバスの分類

- **DB**（管理依存）→ 実インスタンスで最終状態を検証
- **メッセージバス**（非管理依存）→ モックで相互作用を検証

### 8.3.3 E2Eテストについて

E2Eテストとは：デプロイ済みの本番同等環境に対して、**モックなし**で実行するテスト。

統合テストを正しく設計すれば、E2Eテストに近い保護が得られる。
デプロイ後のサニティチェックとして1〜2件だけ書くのが現実的。

### 8.3.4 統合テストの実装例

```csharp
[Fact]
public void Changing_email_from_corporate_to_non_corporate()
{
    // Arrange
    var db = new Database(ConnectionString);
    User user = CreateUser("user@mycorp.com", UserType.Employee, db);
    CreateCompany("mycorp.com", 1, db);
    var messageBusMock = new Mock<IMessageBus>();
    var sut = new UserController(db, messageBusMock.Object);

    // Act
    string result = sut.ChangeEmail(user.UserId, "new@gmail.com");

    // Assert
    Assert.Equal("OK", result);
    object[] userData = db.GetUserById(user.UserId);
    User userFromDb = UserFactory.Create(userData);
    Assert.Equal("new@gmail.com", userFromDb.Email);
    Assert.Equal(UserType.Customer, userFromDb.Type);
    object[] companyData = db.GetCompany();
    Company companyFromDb = CompanyFactory.Create(companyData);
    Assert.Equal(0, companyFromDb.NumberOfEmployees);
    messageBusMock.Verify(
        x => x.SendEmailChangedMessage(user.UserId, "new@gmail.com"),
        Times.Once);
}
```

---

## 8.4 インターフェースで依存を抽象化する

### 8.4.1 インターフェースと疎結合（よくある誤解）

よくある誤解：
- 疎結合を達成するためにインターフェースを使う → **誤り**（実装が1つなら真の抽象ではない）
- 将来の拡張性のため → **誤り**（YAGNIに違反）

> 真の抽象は「発見されるもの」。2つ以上の実装が必要になったとき初めて導入する。

### 8.4.2 インターフェースを使う唯一の正当な理由

**モックを可能にするため。**

理由の順序が重要：

```
統合テストで非管理依存をモックしたい
  → モックするにはインターフェースが必要
  → だからインターフェースを導入する
```

```csharp
public class UserController
{
    private readonly Database _database;       // 管理依存 → 具体クラス（インターフェース不要）
    private readonly IMessageBus _messageBus;  // 非管理依存 → インターフェース（モックのため）
}
```

| 依存の種類 | インターフェース |
|-----------|----------------|
| 管理依存（自アプリのDB等） | **不要** |
| 非管理依存（メッセージバス・外部API等） | **必要** |

### 8.4.3 インプロセス依存へのインターフェースはレッドフラグ

`IUser` のような、ドメインクラスに対するインターフェース（実装が1つ）はレッドフラグ。
ドメインクラスの相互作用をモックしようとしている = 実装の詳細に結合したテストを作っている。

---

## 8.5 統合テストのベストプラクティス

### 8.5.1 ドメインモデルの境界を明示する

ドメインモデルを別アセンブリまたは名前空間に分離する。
→ ユニットテスト（ドメインモデル）と統合テスト（コントローラー）の区別が明確になる。

### 8.5.2 レイヤー数を最小化する

推奨の3層構造：

1. **ドメインモデル**
2. **アプリケーションサービス層**（コントローラー）
3. **インフラ層**（DBリポジトリ、ORM、SMTPゲートウェイ）

過剰な抽象レイヤーはコードのナビゲーションを困難にし、テストを損なう。

### 8.5.3 循環依存を排除する

循環依存：2つ以上のクラスが直接・間接的にお互いに依存し合っている状態。

```
❌ コールバックパターン（悪い例）
CheckOutService → ReportGenerationService
       ↑________________________↓  （循環！）

✅ 戻り値を使う（良い例）
CheckOutService → ReportGenerationService（一方向のみ）
```

```csharp
// ❌ 悪い例：callbackで循環依存
public class ReportGenerationService {
    public void GenerateReport(int orderId, CheckOutService checkOutService) { ... }
}

// ✅ 良い例：戻り値で解消
public class ReportGenerationService {
    public Report GenerateReport(int orderId) { ... }
}
```

インターフェースを導入してもランタイムの循環は残る → 根本解決にならない。

### 8.5.4 テスト内の複数のActセクション

複数のActセクションはコードスメル（複数の振る舞いを同時に検証しているサイン）。

| テストの種類 | 複数Actセクション |
|-------------|----------------|
| ユニットテスト | **絶対にNG** |
| 統合テスト | 原則NG、外部依存のコストが高い場合のみ例外 |
| E2Eテスト | ほぼここに分類される |

> 複数のActセクションを持つテストはほぼ常にE2Eテストに分類される。

---

## 8.6 ロギングのテスト

### 8.6.1 ロギングをテストすべきか

| 種類 | 対象 | 分類 | テストすべきか |
|------|------|------|--------------|
| **サポートロギング** | 運用担当者・管理者向け | 観測可能な振る舞い | **Yes** |
| **診断ロギング** | 開発者向けデバッグ | 実装の詳細 | **No** |

### 8.6.2 ロギングのテスト方法

`ILogger` を直接モックしない。`DomainLogger` ラッパーを作る。

```csharp
public class DomainLogger : IDomainLogger
{
    private readonly ILogger _logger;
    public DomainLogger(ILogger logger) { _logger = logger; }

    public void UserTypeHasChanged(int userId, UserType oldType, UserType newType)
    {
        _logger.Info($"User {userId} changed type from {oldType} to {newType}");
    }
}
```

ドメインクラスにサポートログが必要な場合は**ドメインイベント**経由で実現（関心の分離）：

```csharp
// Userクラス内
AddDomainEvent(new UserTypeChangedEvent(UserId, Type, newType));

// Controllerがイベントを処理して外部通信に変換
_eventDispatcher.Dispatch(user.DomainEvents);
// EmailChangedEvent → _messageBus.SendEmailChangedMessage()
// UserTypeChangedEvent → _domainLogger.UserTypeHasChanged()
```

### 8.6.3 ロギングの量

診断ロギングを過剰に使わない理由：
- コードが煩雑になる（特にドメインモデル）
- ログのS/N比が下がる

推奨：**未処理の例外に対してのみ診断ロギングを使う**

### 8.6.4 ロガーインスタンスの渡し方

```csharp
// ❌ アンビエントコンテキスト（staticフィールド）→ アンチパターン
private static readonly ILogger _logger = LogManager.GetLogger(typeof(User));
// 依存が隠蔽され、テストが困難になる

// ✅ コンストラクタ注入または引数で明示的に渡す
public void ChangeEmail(string newEmail, Company company, ILogger logger) { ... }
```

---

## 8.7 まとめ

- 統合テスト = ユニットテストの条件を1つ以上満たさないテスト
- 統合テストはコントローラーを検証し、ユニットテストはドメインモデルを検証する
- **管理依存**（自アプリ専用）→ 実インスタンスを使う
- **非管理依存**（外部から観測可能）→ モックに置き換える
- インターフェースはモックのためだけに導入する（非管理依存にのみ）
- 実装が1つのインターフェースは真の抽象ではなくYAGNI違反
- ドメインモデルの境界を明示し、レイヤーを最小化し、循環依存を排除する
- サポートロギングは観測可能な振る舞い → `DomainLogger` でテスト
- 診断ロギングは実装の詳細 → テスト不要
- ロガーはアンビエントコンテキストではなく明示的に注入する
