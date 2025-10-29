# 証明書オンライン申請システム 技術的検討事項

## 技術スタック

### フレームワーク: **Laravel 10.x**
### 管理画面: **Laravel-admin**
### インフラ: **さくらレンタルサーバ**

---

## 1. 公的個人認証サービス（JPKI）連携

### 1.1 JPKI概要
- マイナンバーカードのICチップに記録された電子証明書を利用
- 本人確認と電子署名の機能を提供
- 地方公共団体情報システム機構（J-LIS）が運営

### 1.2 認証フロー
```
1. ユーザーがLINEアプリから申請開始
2. LIFFアプリでマイナンバーカード読み取り画面表示
3. スマートフォンのNFC機能でカード読み取り
4. 利用者証明用電子証明書で本人確認
5. 署名用電子証明書で申請内容に電子署名
6. Laravelバックエンドで証明書検証・署名検証
7. 本人確認完了
```

### 1.3 Laravel実装方法
- **パッケージ**: JPKI検証用PHPライブラリの導入
- **サービスクラス**: `App\Services\JpkiService`
  - 証明書検証
  - 署名検証
  - CRL（証明書失効リスト）チェック

```php
// 実装例
class JpkiService {
    public function verifyCertificate($certificate) {
        // 証明書の有効性確認
    }

    public function verifySignature($data, $signature) {
        // 電子署名の検証
    }
}
```

### 1.4 技術要件
- **クライアント側**
  - NFC対応スマートフォン必須
  - LIFF SDKでのマイナンバーカード読み取り
  - JavaScript Crypto API

- **サーバー側（Laravel）**
  - OpenSSL拡張モジュール
  - JPKI検証ライブラリ
  - 証明書失効リスト（CRL）の定期取得（Cronジョブ）

### 1.5 実装上の課題
- [ ] NFCリーダーの対応状況確認
- [ ] iOS / Android での実装差異
- [ ] マイナポータルAPIの利用申請
- [ ] 証明書検証の処理速度最適化
- [ ] エラーハンドリング（カード読み取り失敗時など）

### 1.6 セキュリティ考慮事項
- 証明書の有効性確認
- 失効リストの定期更新（1日1回Cron実行）
- タイムスタンプの付与
- 申請データの改ざん防止（ハッシュ化）

---

## 2. LINE連携

### 2.1 LINE Messaging API
Laravelでの実装パターン

- **パッケージ**: `linecorp/line-bot-sdk` をComposerでインストール

```php
// config/services.php
'line' => [
    'channel_id' => env('LINE_CHANNEL_ID'),
    'channel_secret' => env('LINE_CHANNEL_SECRET'),
    'channel_token' => env('LINE_CHANNEL_ACCESS_TOKEN'),
],
```

- **Webhook処理**: `app/Http/Controllers/LineWebhookController.php`
  - 申請受付通知
  - 発送完了通知
  - リマインダー

### 2.2 LIFF（LINE Front-end Framework）
- **LIFFアプリ構成**
  - 証明書選択画面
  - JPKI認証画面
  - 申請内容入力画面
  - 決済画面
  - 完了画面

- **技術構成**
  - Blade Templates（Laravel標準）
  - Alpine.js / jQuery（軽量なJavaScript）
  - LIFF SDK

```javascript
// LIFF初期化
liff.init({ liffId: 'YOUR-LIFF-ID' })
  .then(() => {
    if (!liff.isLoggedIn()) {
      liff.login();
    }
  });
```

### 2.3 既存システムとの連携
- 既存の都留市公式LINEアカウントを活用
- 既存の予約システムとのデータベース統合（検討）
- ユーザーテーブルの共通化

### 2.4 Laravelルート設計
```php
// routes/web.php
Route::prefix('liff')->group(function () {
    Route::get('/certificates', [CertificateController::class, 'index']);
    Route::post('/applications', [ApplicationController::class, 'store']);
});

// routes/api.php
Route::post('/webhook/line', [LineWebhookController::class, 'handle']);
```

---

## 3. 決済システム連携

### 3.1 推奨決済サービス: **Stripe**
さくらレンタルサーバでの運用を考慮し、Stripeを推奨

| 理由 | 説明 |
|------|------|
| 導入が容易 | PHPライブラリが充実、Laravel統合が簡単 |
| 初期費用0円 | 月額費用も0円、決済手数料のみ |
| セキュリティ | PCI DSS準拠、カード情報を保持しない |
| ドキュメント | 日本語ドキュメント充実 |

### 3.2 Laravel実装
```bash
composer require stripe/stripe-php
```

```php
// app/Services/StripeService.php
use Stripe\StripeClient;

class StripeService {
    protected $stripe;

    public function __construct() {
        $this->stripe = new StripeClient(config('services.stripe.secret'));
    }

    public function createPaymentIntent($amount) {
        return $this->stripe->paymentIntents->create([
            'amount' => $amount,
            'currency' => 'jpy',
        ]);
    }
}
```

### 3.3 決済フロー
```
1. 申請内容確定
2. 手数料計算（証明書手数料 + 郵送料）
3. Stripe決済画面表示
4. カード情報入力
5. 決済実行（Stripe API）
6. 決済結果確認（Webhook受信）
7. 申請完了
```

### 3.4 手数料設定
| 証明書種類 | 手数料 |
|-----------|--------|
| 住民票の写し | 300円 |
| 印鑑登録証明書 | 300円 |
| 戸籍謄本 | 450円 |
| 戸籍抄本 | 450円 |
| 戸籍附票 | 300円 |
| 郵送料 | 84円〜 |

### 3.5 Stripe手数料
- 3.6%（1決済あたり）

---

## 4. システムアーキテクチャ

### 4.1 システム構成図

```
┌─────────────────┐
│   LINE App      │
│  (ユーザー)      │
└────────┬────────┘
         │
┌────────▼────────┐
│  LINE Platform  │
│ (Messaging API) │
└────────┬────────┘
         │
┌────────▼────────────────────────┐
│      LIFF Application           │
│  (Blade + Alpine.js)            │
│  - 証明書選択                    │
│  - JPKI認証                     │
│  - 申請入力                      │
│  - 決済（Stripe）                │
└────────┬────────────────────────┘
         │
┌────────▼────────────────────────┐
│   Laravel Application           │
│   (さくらレンタルサーバ)          │
│   - API Endpoints               │
│   - JPKI Verification           │
│   - Stripe Integration          │
│   - Laravel-admin (管理画面)    │
└────────┬────────────────────────┘
         │
    ┌────▼────┐
    │  MySQL  │
    │  8.0    │
    └─────────┘

External Services:
- LINE Messaging API
- JPKI Verification Service (J-LIS)
- Stripe (決済)
- SendGrid (メール配信)
```

### 4.2 さくらレンタルサーバ要件

#### 推奨プラン
**さくらのレンタルサーバ ビジネスプラン以上**

| 項目 | スペック |
|------|---------|
| ディスク容量 | 300GB |
| MySQL | 400GB |
| 転送量 | 無制限 |
| PHP | 8.1以上対応 |
| SSH | 利用可能 |
| Cron | 利用可能 |
| SSL | 無料SSL（Let's Encrypt） |
| 月額料金 | 2,619円（年間一括払い） |

#### Laravel動作要件
- PHP 8.1以上
- Composer（SSH経由でインストール）
- MySQL 8.0
- Cron（スケジューラー用）

### 4.3 デプロイ方法
1. GitHubにコードをpush
2. SSH経由でさくらサーバにログイン
3. `git pull`でコード取得
4. `composer install`
5. `php artisan migrate`
6. キャッシュクリア・最適化

---

## 5. Laravel-admin による管理画面

### 5.1 Laravel-admin とは
- Laravelベースの管理画面構築パッケージ
- CRUD機能を自動生成
- カスタマイズが容易

### 5.2 インストール
```bash
composer require encore/laravel-admin
php artisan admin:install
```

### 5.3 実装する管理機能
- 申請管理
  - 申請一覧・検索・詳細表示
  - ステータス管理（申請中/発行済/配送中/完了）
- 証明書発行管理
  - 発行指示
  - 発行記録
- 決済管理
  - 決済状況確認
  - 返金処理
- ユーザー管理
- 統計・レポート機能
  - 申請数推移（グラフ表示）
  - 証明書種類別集計
- システム設定
  - 証明書種類・手数料設定
  - お知らせ管理

### 5.4 カスタマイズ例
```php
// app/Admin/Controllers/ApplicationController.php
use App\Models\Application;
use Encore\Admin\Controllers\AdminController;
use Encore\Admin\Grid;
use Encore\Admin\Form;

class ApplicationController extends AdminController
{
    protected function grid()
    {
        $grid = new Grid(new Application());
        $grid->column('id', 'ID');
        $grid->column('application_number', '申請番号');
        $grid->column('certificate_type', '証明書種類');
        $grid->column('status', 'ステータス');
        return $grid;
    }
}
```

---

## 6. セキュリティ設計

### 6.1 セキュリティ要件
- [x] SSL/TLS通信（Let's Encrypt）
- [x] Laravelの標準的なセキュリティ機能
  - CSRF保護
  - XSS対策
  - SQLインジェクション対策（Eloquent ORM）
- [x] 個人情報の暗号化保存
- [x] アクセス制御・認証
  - Laravel Sanctum（API認証）
  - Laravel-admin（管理画面認証）
- [x] 監査ログ記録
- [x] 不正アクセス対策
  - ログイン試行回数制限（Laravel標準機能）

### 6.2 データ保護
- **個人情報の暗号化**
  - Laravelの`encrypt()`関数を使用
  - データベースレベルの暗号化

```php
// モデルでの実装例
protected $casts = [
    'name' => 'encrypted',
    'address' => 'encrypted',
];
```

- **バックアップ**
  - さくらサーバの自動バックアップ機能
  - 日次でデータベースダンプ（Cronジョブ）

### 6.3 監査ログ
Laravelパッケージ: `owen-it/laravel-auditing`

```bash
composer require owen-it/laravel-auditing
```

- ログ記録項目
  - アクセスログ
  - 認証ログ
  - 操作ログ（申請・決済）
  - エラーログ

---

## 7. データベース設計

### 7.1 主要テーブル（Laravel Migration）

#### users（ユーザー）
```php
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('line_user_id')->unique();
    $table->string('name'); // 暗号化
    $table->string('postal_code')->nullable();
    $table->text('address')->nullable(); // 暗号化
    $table->string('phone')->nullable(); // 暗号化
    $table->timestamps();
});
```

#### applications（申請）
```php
Schema::create('applications', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained();
    $table->string('application_number')->unique();
    $table->string('certificate_type');
    $table->integer('quantity')->default(1);
    $table->text('purpose')->nullable();
    $table->enum('delivery_method', ['mail', 'counter']);
    $table->text('delivery_address')->nullable(); // 暗号化
    $table->enum('status', ['pending', 'issued', 'shipping', 'completed']);
    $table->integer('amount');
    $table->enum('payment_status', ['unpaid', 'paid', 'refunded']);
    $table->text('jpki_signature')->nullable(); // 暗号化
    $table->timestamps();
});
```

#### payments（決済）
```php
Schema::create('payments', function (Blueprint $table) {
    $table->id();
    $table->foreignId('application_id')->constrained();
    $table->string('payment_method');
    $table->integer('amount');
    $table->string('stripe_payment_intent_id')->nullable();
    $table->enum('status', ['pending', 'completed', 'refunded']);
    $table->timestamps();
});
```

#### certificates（証明書マスタ）
```php
Schema::create('certificates', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->integer('fee');
    $table->text('description')->nullable();
    $table->boolean('is_active')->default(true);
    $table->timestamps();
});
```

#### audit_logs（監査ログ）
```php
Schema::create('audit_logs', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->nullable()->constrained();
    $table->string('action');
    $table->ipAddress('ip_address')->nullable();
    $table->text('user_agent')->nullable();
    $table->timestamps();
});
```

---

## 8. テスト計画

### 8.1 Laravelテストフレームワーク
- **PHPUnit**: Laravel標準のテストフレームワーク

### 8.2 テスト種類
- **単体テスト（Feature Test）**
  ```bash
  php artisan test
  ```

- **ブラウザテスト（Laravel Dusk）**
  ```bash
  php artisan dusk
  ```

### 8.3 テストケース例

#### JPKI認証テスト
```php
public function test_jpki_certificate_verification()
{
    $service = new JpkiService();
    $result = $service->verifyCertificate($testCertificate);
    $this->assertTrue($result);
}
```

#### 申請フローテスト
```php
public function test_application_submission()
{
    $response = $this->post('/api/applications', [
        'certificate_type' => 'resident_certificate',
        'quantity' => 1,
    ]);
    $response->assertStatus(201);
}
```

---

## 9. さくらレンタルサーバでの運用

### 9.1 初期セットアップ
1. さくらレンタルサーバ契約（ビジネスプラン）
2. SSH接続設定
3. Composer インストール
4. Laravel プロジェクトデプロイ
5. MySQL データベース作成
6. `.env` ファイル設定
7. `php artisan migrate`実行
8. SSL証明書設定（Let's Encrypt）

### 9.2 Cronジョブ設定
```bash
# Laravelスケジューラー実行（毎分）
* * * * * cd /home/username/laravel && php artisan schedule:run >> /dev/null 2>&1
```

Laravelスケジューラーで定義するタスク：
```php
// app/Console/Kernel.php
protected function schedule(Schedule $schedule)
{
    // CRL（証明書失効リスト）取得
    $schedule->command('jpki:update-crl')->daily();

    // データベースバックアップ
    $schedule->command('backup:run')->daily();
}
```

### 9.3 パフォーマンス最適化
- **OPcache有効化**
  ```php
  opcache.enable=1
  opcache.memory_consumption=128
  ```

- **Laravelキャッシュ**
  ```bash
  php artisan config:cache
  php artisan route:cache
  php artisan view:cache
  ```

- **データベースインデックス**
  ```php
  $table->index('line_user_id');
  $table->index('application_number');
  ```

### 9.4 監視
- **Laravel Telescope**（開発環境のみ）
  ```bash
  composer require laravel/telescope --dev
  ```

- **ログ監視**
  - `storage/logs/laravel.log`
  - エラーログをメール通知（Laravel標準機能）

---

## 10. コスト試算

### 10.1 開発工数見積もり（250万円）

| フェーズ | 工数（人日） | 単価（円/日） | 金額（円） |
|---------|------------|--------------|-----------|
| 要件定義・設計 | 10 | 60,000 | 600,000 |
| Laravel基盤構築 | 3 | 60,000 | 180,000 |
| LIFF開発 | 7 | 50,000 | 350,000 |
| API開発（JPKI・決済） | 10 | 60,000 | 600,000 |
| Laravel-admin構築 | 5 | 50,000 | 250,000 |
| テスト | 5 | 50,000 | 250,000 |
| インフラ構築・デプロイ | 3 | 50,000 | 150,000 |
| ドキュメント・研修 | 2 | 50,000 | 100,000 |
| **合計** | **45人日** | | **2,480,000** |

※若干のバッファ含めて**250万円**

### 10.2 インフラコスト（月額）

| 項目 | 料金（円/月） |
|------|-------------|
| さくらレンタルサーバ（ビジネスプラン） | 2,619 |
| SSL証明書（Let's Encrypt） | 0 |
| LINE Messaging API | 0〜5,000 |
| Stripe決済手数料 | 従量課金（3.6%） |
| SendGrid（メール配信） | 0〜3,000 |
| **合計** | **5,000〜10,000** |

### 10.3 保守運用費用（月額5万円）

| 項目 | 内容 |
|------|------|
| 定期保守 | 月次動作確認、ログ分析 |
| 問い合わせ対応 | メール・電話サポート（平日9-18時） |
| セキュリティパッチ適用 | Laravel・PHP・MySQLのアップデート |
| バックアップ管理 | データベースバックアップの確認 |
| 軽微な修正対応 | バグフィックス、小規模な機能改修 |

---

## 11. 開発スケジュール（約5ヶ月）

| フェーズ | 期間 |
|---------|------|
| 要件定義・設計 | 3週間 |
| 開発 | 8週間 |
| テスト | 3週間 |
| デプロイ・研修 | 2週間 |
| **合計** | **約4ヶ月** |

---

## 12. リスクと対策

| リスク | 影響度 | 対策 |
|-------|--------|------|
| さくらサーバの性能不足 | 中 | 事前に負荷テスト、必要に応じてプラン変更 |
| JPKI連携の技術的困難 | 高 | 事前検証POC、専門家への相談 |
| Stripe決済トラブル | 中 | テスト環境での入念な検証 |
| Laravel学習コスト | 低 | 既存の都留市システムでLaravel使用経験あり |

---

## 13. 参考リンク

- [Laravel 公式ドキュメント](https://laravel.com/docs)
- [Laravel-admin](http://laravel-admin.org/)
- [さくらレンタルサーバ](https://www.sakura.ne.jp/)
- [Stripe PHP SDK](https://stripe.com/docs/api)
- [公的個人認証サービス](https://www.jpki.go.jp/)
