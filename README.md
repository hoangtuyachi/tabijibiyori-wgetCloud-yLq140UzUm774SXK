# Laravel 新项目避坑指南10 大基础设置让代码半年不崩

有没有遇到过这种 Laravel 项目：刚上线那会儿干干净净，过三个月就变成无法收拾的灾难？Controller 动不动就 500 多行、慢得要命的数据库查询随处可见，甚至有人把 `.env` 推上 GitHub，所有密钥一夜之间全线暴露。

别以为只有你栽过这种坑。来自开发者论坛和 Stack Overflow 的统计显示：Laravel 应用里的性能问题有 70% 可以在一开始就避免，接近 50% 的安全事故也都是初始配置不到位造成的。

[原文链接 Laravel 新项目避坑指南：10 大基础设置让代码半年不崩](https://github.com)

## 别只会复制 `.env`——先搞懂它！

### 常见问题

很多新手只会把 `.env.example` 复制成 `.env`，填上数据库账号就直接 `php artisan serve`。问题是，这个文件里藏着许多直接影响线上环境的配置，稍不留神就会出事。

真实案例：有开发者上线后忘了把 `APP_DEBUG=true` 改成 `false`，结果所有错误信息包括 SQL 查询、文件路径、敏感数据全都暴露，黑客看到简直像打开盲盒。

### 正确做法

```
# 线上环境一定要设成 false！
APP_DEBUG=false
APP_ENV=production

# 生成独一无二的 key，别用默认值
APP_KEY=base64:xxxxx

# 线上别写 127.0.0.1
APP_URL=https://yourdomain.com

# 数据库凭据千万别提交到 Git！
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=your_database_name
DB_USERNAME=database_user
DB_PASSWORD=strong_password_please

# 会话与缓存驱动
SESSION_DRIVER=redis
CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
```

### 实用提醒

* **生成强随机 APP\_KEY**：每个新项目都要跑一遍 `php artisan key:generate`，它可是用来加密 Session 等敏感数据的。
* **维护完整的 `.env.example`**：把所有必须的变量都列出来，但不要写真实值，方便团队协作。
* **确认 `.env` 没进 Git**：默认 `.gitignore` 会忽略它，但还是要确认。如果不小心提交了，赶紧这么做：

```
git rm --cached .env
git commit -m "Remove .env from version control"
```

## 从第一天就定下命名规范

### 常见问题

```
// 糊里糊涂的 Controller
class Users extends Controller {
    public function GetUserData() {
        $All_Users = User::all();
        return view('user-list', ['data' => $All_Users]);
    }
}

// 奇怪的 migration
Schema::create('User', function (Blueprint $table) {
    $table->id();
    $table->string('UserName');
    $table->string('user-email');
});
```

如果有新人接手这种代码，第一反应肯定是“这是边吃饭边写的吧？”。

### 正确做法

Laravel 社区已经约定俗成了一套命名规范，照着走就省事。

#### 模型

```
// ✅ 使用单数、PascalCase
class Article extends Model {
    // hasMany 关系 → 复数
    public function comments() {
        return $this->hasMany(Comment::class);
    }
    
    // belongsTo 关系 → 单数
    public function author() {
        return $this->belongsTo(User::class);
    }
}

// ❌ 错误示范
class Articles extends Model { }
class article extends Model { }
```

#### 控制器

```
// ✅ 单数、PascalCase，并以 Controller 结尾
class ArticleController extends Controller {
    // 方法用 camelCase
    public function showLatest() {
        // 变量也用 camelCase
        $latestArticles = Article::latest()
            ->take(5)
            ->get();
            
        return view('articles.latest', [
            'articles' => $latestArticles
        ]);
    }
}

// ❌ 错误示范
class ArticlesController { }  // 复数
class articleController { }   // 小写开头
class Article_Controller { }  // 下划线
```

#### 迁移和数据库

```
// ✅ 复数、snake_case
Schema::create('blog_posts', function (Blueprint $table) {
    $table->id();
    $table->string('post_title');
    $table->text('post_content');
    $table->foreignId('category_id')
          ->constrained('categories')
          ->onDelete('cascade');
    $table->timestamps();
});

// ❌ 单数或 PascalCase
Schema::create('BlogPost', function (Blueprint $table) {
    $table->string('PostTitle');
});
```

#### 路由

```
// ✅ 复数、kebab-case
Route::get('/blog-posts', [BlogPostController::class, 'index'])
    ->name('blog-posts.index');

// ❌ 错误示范
Route::get('/BlogPosts', [BlogPostController::class, 'index']);
Route::get('/blog_posts', [BlogPostController::class, 'index']);
```

### 为什么要这么做？

想象你和 5 个开发一起合作，每个人有一套命名习惯，整个代码库就像水果捞一样乱七八糟。统一命名带来的好处：

* 新人上手快，不用猜。
* 排查 Bug 更轻松，一眼就知道谁是模型谁是控制器。
* IDE 自动补全更精准，PHPStorm、VS Code 都会更懂你。

## 外键约束必须上，别全靠代码兜底！

### 常见问题

```
// ❌ 敷衍的建表
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->unsignedBigInteger('user_id'); // 只是普通字段
    $table->string('title');
    $table->timestamps();
});
```

看起来没毛病？等等，这里有三个隐患：

* **孤儿数据**：用户删了，帖子还在，`$post->user` 直接报错 `Trying to get property of null`。
* **数据不一致**：别人可以插入不存在的 `user_id`，数据库全是垃圾数据。
* **调试噩梦**：出问题时只能一个一个查数据。

有人说“我们可以靠 Laravel Observer 或 Events 处理逻辑”。确实，但这不是 100% 保险：

* 原生 SQL 会绕过 Eloquent。
* 批量导入直接写数据库。
* 任何一个忘记触发事件的 Bug。

数据照样会烂掉。

### 正确做法

```
// ✅ 建表时加外键约束
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')
          ->constrained('users')
          ->onDelete('cascade');  // 也可以是 restrict、set null 等
    $table->string('title');
    $table->text('content');
    $table->timestamps();
});

// 更复杂的关系
Schema::create('comments', function (Blueprint $table) {
    $table->id();
    $table->foreignId('post_id')
          ->constrained()  // 自动推断 posts 表
          ->onDelete('cascade');
    $table->foreignId('user_id')
          ->constrained()
          ->onDelete('restrict');  // 有评论的用户禁止删除
    $table->text('comment_body');
    $table->timestamps();
});
```

### 你需要理解的选项

```
// 1. CASCADE - 删除父记录时一起删
->onDelete('cascade')

// 2. RESTRICT - 阻止删除父记录
->onDelete('restrict')

// 3. SET NULL - 设置为 NULL（记得列要 nullable）
->onDelete('set null')
->nullable()

// 4. NO ACTION - MySQL 默认行为
->onDelete('no action')
```

### 什么时候可以不加？

* **分片数据库**：比如 PlanetScale 不支持外键。
* **一人维护的小项目**：你能保证数据一致性。
* **极端性能场景**：外键会增加一点点开销（其实很少见）。

对 99% 的项目来说，加外键一定是最佳选择。

## 起步就躲开 N+1 查询

### 常见问题

这是 Laravel 性能问题 Top 1。最危险的是，你上线前根本察觉不到。

```
// ❌ 典型的 N+1 查询
public function showAllPosts() {
    $posts = Post::all();
    
    return view('posts.index', compact('posts'));
}

{{-- Blade 视图 --}}
@foreach($posts as $post)
    <div class="post">
        <h2>{{ $post->title }}</h2>
        <p>By: {{ $post->user->name }}</p>  <!-- 危险！ -->
        <p>{{ $post->comments->count() }} comments</p>  <!-- 也危险！ -->
    </div>
@endforeach
```

如果有 100 篇帖子？

* 1 次取 posts。
* 100 次取用户。
* 100 次统计 comments。

总共 201 条查询，服务器直接崩溃。

### 正确做法

```
// ✅ 预加载
public function showAllPosts() {
    $posts = Post::with(['user', 'comments'])
        ->latest()
        ->get();
    
    return view('posts.index', compact('posts'));
}

// ✅ 只需要数量时用 withCount
public function showAllPosts() {
    $posts = Post::with('user')
        ->withCount('comments')
        ->latest()
        ->get();
    
    return view('posts.index', compact('posts'));
}
```

只需要 3 条查询，性能翻倍都不止。

### 从第一天就装 Laravel Debugbar

在开发环境安装它：

```
composer require barryvdh/laravel-debugbar --dev
```

Debugbar 会实时显示：

* 当前页面跑了多少条查询。
* 每条查询耗时。
* 是否有重复查询。

黄金法则：单页查询超过 20 条就要警觉。

### 小技巧：Lazy Eager Loading

```
// 即使你已经拿到了 $posts
$posts = Post::all();

// 仍然可以临时加载关联
$posts->load(['user', 'comments']);
```

## 建 Service 层，别把所有逻辑塞进 Controller！

### 常见问题

```
// ❌ 典型的胖 Controller
class OrderController extends Controller 
{
    public function checkout(Request $request) 
    {
        $validated = $request->validate([
            'items' => 'required|array',
            'payment_method' => 'required|string',
            'shipping_address' => 'required|string'
        ]);
        
        DB::beginTransaction();
        try {
            // 50 行计算逻辑
            $total = 0;
            foreach ($validated['items'] as $item) {
                $product = Product::find($item['id']);
                $total += $product->price * $item['quantity'];
                $product->stock -= $item['quantity'];
                $product->save();
            }
            
            // 30 行支付逻辑
            if ($validated['payment_method'] === 'credit_card') {
                // 处理信用卡
            } elseif ($validated['payment_method'] === 'bank_transfer') {
                // 处理银行转账
            }
            
            // 40 行物流逻辑
            $shippingCost = $this->calculateShipping(...);
            
            // 20 行创建订单逻辑
            $order = Order::create([...]);
            
            // 15 行发邮件逻辑
            Mail::to($user)->send(new OrderConfirmation($order));
            
            // 10 行发通知逻辑
            
            DB::commit();
            return redirect()->route('orders.success');
        } catch (\Exception $e) {
            DB::rollback();
            return back()->withErrors('Order failed');
        }
    }
}
```

这种 Controller 动辄 150+ 行，维护成本爆炸，逻辑也没法复用。

### 正确做法：拆到 Service 层

```
// ✅ 精简后的 Controller
class OrderController extends Controller 
{
    public function __construct(
        private OrderService $orderService
    ) {}
    
    public function checkout(CheckoutRequest $request) 
    {
        try {
            $order = $this->orderService->processCheckout(
                $request->validated()
            );
            
            return redirect()
                ->route('orders.success', $order->id)
                ->with('success', '订单处理成功！');
                
        } catch (InsufficientStockException $e) {
            return back()->withErrors('商品库存不足');
        } catch (PaymentFailedException $e) {
            return back()->withErrors('支付失败');
        }
    }
}
```

看到了吗？Controller 只做它该做的：接请求、调用服务、给出响应。

```
// app/Services/OrderService.php
class OrderService 
{
    public function __construct(
        private CartService $cartService,
        private PaymentService $paymentService,
        private ShippingService $shippingService,
        private NotificationService $notificationService
    ) {}
    
    public function processCheckout(array $data): Order 
    {
        return DB::transaction(function () use ($data) {
            // 校验库存
            $this->cartService->validateStock($data['items']);
            
            // 计算总价
            $orderTotal = $this->cartService->calculateTotal($data['items']);
            
            // 计算运费
            $shippingCost = $this->shippingService->calculate(
                $data['shipping_address']
            );
            
            // 处理支付
            $payment = $this->paymentService->charge(
                $data['payment_method'],
                $orderTotal + $shippingCost
            );
            
            // 创建订单
            $order = $this->createOrder($data, $payment);
            
            // 减库存
            $this->cartService->reduceStock($data['items']);
            
            // 发通知
            $this->notificationService->sendOrderConfirmation($order);
            
            return $order;
        });
    }
    
    private function createOrder(array $data, Payment $payment): Order 
    {
        // 专注处理创建订单的细节
        return Order::create([
            'user_id' => auth()->id(),
            'payment_id' => $payment->id,
            'total_amount' => $payment->amount,
            'status' => OrderStatus::PENDING,
            // ... 其他字段
        ]);
    }
}
```

### 推荐目录结构

```
app/
├── Http/
│   ├── Controllers/
│   │   └── OrderController.php
│   └── Requests/
│       └── CheckoutRequest.php
├── Services/
│   ├── OrderService.php
│   ├── CartService.php
│   ├── PaymentService.php
│   └── ShippingService.php
├── Models/
│   ├── Order.php
│   ├── Product.php
│   └── Payment.php
└── Exceptions/
    ├── InsufficientStockException.php
    └── PaymentFailedException.php
```

### 好处一箩筐

* **可测试性**：每个 Service 都能单独写测试。
* **可复用**：逻辑可以给 API、命令、队列复用。
* **可维护**：Bug 和新需求都能快速定位。
* **团队协作**：大家分模块开发，冲突更少。

## 请求验证交给 Form Request，别堆在 Controller！

### 常见问题

```
// ❌ Controller 里塞满验证
public function store(Request $request) 
{
    $validated = $request->validate([
        'title' => 'required|max:255|unique:posts',
        'slug' => 'required|unique:posts',
        'content' => 'required|min:100',
        'category_id' => 'required|exists:categories,id',
        'tags' => 'required|array',
        'tags.*' => 'exists:tags,id',
        'featured_image' => 'required|image|max:2048',
        'meta_title' => 'required|max:60',
        'meta_description' => 'required|max:160',
        // ... 还有一堆
    ]);
    
    // 后面才是创建逻辑
}
```

问题在于：

* Controller 变得臃肿。
* 验证规则无法复用。
* 自定义提示语很难管理。
* 复杂逻辑不好写。

### 正确做法：使用 Form Request

```
php artisan make:request StorePostRequest
```

```
// app/Http/Requests/StorePostRequest.php
class StorePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        // 判断当前用户是否能创建文章
        return auth()->user()->can('create-post');
    }

    public function rules(): array
    {
        return [            
            'title' => ['required', 'max:255', 'unique:posts,title'],
            'slug' => ['required', 'unique:posts,slug', 'regex:/^[a-z0-9-]+$/'],
            'content' => 'required|min:100',
            'category_id' => 'required|exists:categories,id',
            'tags' => 'required|array|min:1|max:5',
            'tags.*' => 'exists:tags,id',
            'featured_image' => 'required|image|mimes:jpg,png,webp|max:2048',
            'meta_title' => 'nullable|max:60',
            'meta_description' => 'nullable|max:160',
            'publish_at' => 'nullable|date|after:now',
        ];
    }
    
    public function messages(): array
    {
        return [
            'title.required' => '文章标题是必填项！',
            'title.unique' => '标题已存在，换个更有创意的吧！',
            'slug.regex' => 'Slug 只能包含小写字母、数字和连字符',
            'tags.min' => '至少选择 1 个标签方便分类',
            'tags.max' => '最多 5 个标签，别太贪心！',
            'featured_image.max' => '图片太大了，最大 2MB',
        ];
    }
    
    public function attributes(): array
    {
        return [            
          'category_id' => '分类',            
          'featured_image' => '封面图',            
          'publish_at' => '发布时间',        
        ];
    }
    
    // 自定义验证逻辑
    protected function prepareForValidation(): void
    {
        // 自动根据标题生成 slug
        if (!$this->slug) {
            $this->merge([
                'slug' => Str::slug($this->title)
            ]);
        }
    }
}
```

```
// Controller 变得超级干净！
class PostController extends Controller 
{
    public function store(StorePostRequest $request) 
    {
        // 数据自动验证、自动授权
        $post = Post::create($request->validated());
        
        // 关联标签
        $post->tags()->attach($request->tags);
        
        return redirect()
            ->route('posts.show', $post)
            ->with('success', '文章发布成功！');
    }
}
```

## `.gitignore` 要写完整

### 常见问题

总有人把不该进仓库的文件提交进去：

```
# ❌ 常见事故
.env                  # 数据库密码全暴露！
/vendor              # 100MB+ 的依赖
node_modules/        # 几千个文件
.DS_Store           # macOS 垃圾文件
Thumbs.db           # Windows 垃圾文件
*.log               # 巨大的日志
/storage/*.key      # SSL 密钥
```

真实事件：有个创业团队把 `.env` 推上了 GitHub，公开仓库 2 小时内服务器就被人拿去挖矿，AWS 账单直接涨到 15,000 美元。

### 完整的 `.gitignore` 参考

```
# Laravel
/node_modules
/public/hot
/public/storage
/storage/*.key
/vendor

# 环境文件
.env
.env.backup
.env.production
.env.testing
.env.*.php
.phpunit.result.cache

# IDE & 编辑器
.idea/
.vscode/
*.sublime-project
*.sublime-workspace
.phpstorm.meta.php
_ide_helper.php
_ide_helper_models.php

# 操作系统
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db
desktop.ini

# 日志 & 数据库
*.log
*.sql
*.sqlite
*.sqlite-journal

# 构建产物
/public/build
/public/mix-manifest.json
/public/js/app.js
/public/css/app.css
npm-debug.log
yarn-error.log

# 部署
.deploy/
.rocketeer/

# 测试
/coverage
/phpunit.xml

# 安全 & 密钥
*.pem
*.key
.cert
```

### 如果 `.env` 已经被提交

别慌，但动作要快：

```
# 1. 从 Git 历史里移除
git rm --cached .env

# 2. 提交变更
git commit -m "Remove .env from version control"

# 3. 推上远端
git push origin main

# 4. 重点！所有泄露的密钥都要立刻更换
# - 改数据库密码
# - 重生成 API Key
# - 轮换所有密钥
```

如果仓库是公开的，密钥在 Git 历史里就算曝光了，爬虫早晚会找到。

## 制定清晰的日志策略

### 常见问题

不少人靠 `dd()`、`var_dump()` 调试，线上出错却没任何记录；反过来也有人把所有东西都往日志里塞，结果 log 文件动辄几个 G。

```
// ❌ 粗糙的日志
public function processPayment($orderId) 
{
    try {
        // 处理支付
        echo "Processing order: " . $orderId;  // 不专业
        var_dump($paymentData);                // 生产环境危险
        
        // 支付逻辑
        
    } catch (\Exception $e) {
        // 错误直接消失
        return false;
    }
}
```

### 正确做法

```
// app/Services/PaymentService.php
use Illuminate\Support\Facades\Log;

class PaymentService 
{
    public function processPayment(Order $order): bool 
    {
        Log::info('开始处理支付', [
            'order_id' => $order->id,
            'amount' => $order->total,
            'user_id' => $order->user_id,
            'timestamp' => now()
        ]);
        
        try {
            $payment = $this->chargeCustomer($order);
            
            Log::info('支付成功', [
                'order_id' => $order->id,
                'payment_id' => $payment->id,
                'gateway' => $payment->gateway
            ]);
            
            return true;
            
        } catch (PaymentGatewayException $e) {
            Log::error('支付网关失败', [
                'order_id' => $order->id,
                'error' => $e->getMessage(),
                'gateway' => $e->getGateway(),
                'trace' => $e->getTraceAsString()
            ]);
            
            throw $e;
            
        } catch (\Exception $e) {
            Log::critical('支付发生未知异常', [
                'order_id' => $order->id,
                'error' => $e->getMessage(),
                'file' => $e->getFile(),
                'line' => $e->getLine()
            ]);
            
            throw $e;
        }
    }
}
```

### 在 `config/logging.php` 里配置多通道

```
'channels' => [
    'stack' => [
        'driver' => 'stack',
        'channels' => ['daily', 'slack'],
        'ignore_exceptions' => false,
    ],

    'daily' => [
        'driver' => 'daily',
        'path' => storage_path('logs/laravel.log'),
        'level' => env('LOG_LEVEL', 'debug'),
        'days' => 14,  // 14 天自动清理
    ],

    'payment' => [
        'driver' => 'daily',
        'path' => storage_path('logs/payment.log'),
        'level' => 'info',
        'days' => 90,  // 支付日志保留 3 个月
    ],

    'security' => [
        'driver' => 'daily',
        'path' => storage_path('logs/security.log'),
        'level' => 'warning',
        'days' => 365,  // 安全日志留一年
    ],

    'slack' => [
        'driver' => 'slack',
        'url' => env('LOG_SLACK_WEBHOOK_URL'),
        'username' => 'Laravel Log',
        'emoji' => ':boom:',
        'level' => 'critical',  // 只有严重问题才打到 Slack
    ],
],
```

## 别等出 Bug 再想起测试

### 常见问题

“测试？项目跑起来再说吧……”

这通常意味着：

* Bug 是线上用户帮你发现的。
* 每次发版都提心吊胆。
* 修一个 Bug 引入三个新 Bug。
* 团队信心值直接跌到负数。

### 搭好测试环境

```
# 安装 Pest（更现代的测试框架）
composer require pestphp/pest --dev --with-all-dependencies
php artisan pest:install

# 或者坚持用 PHPUnit
dcomposer require phpunit/phpunit --dev
```

`.env.testing` 示例：

```
APP_ENV=testing
APP_KEY=base64:testing-key-here
DB_CONNECTION=sqlite
DB_DATABASE=:memory:
CACHE_DRIVER=array
SESSION_DRIVER=array
QUEUE_CONNECTION=sync
MAIL_MAILER=array
```

### 写点像样的测试

```
// tests/Feature/OrderTest.php
php</span

use App\Models\User;
use App\Models\Product;
use App\Models\Order;

test('用户可以在库存充足时创建订单', function () {
    // Arrange
    $user = User::factory()->create();
    $product = Product::factory()->create([
        'price' => 100000,
        'stock' => 10
    ]);
    
    // Act
    $response = $this->actingAs($user)
        ->post('/orders', [
            'product_id' => $product->id,
            'quantity' => 2
        ]);
    
    // Assert
    $response->assertStatus(201);
    $response->assertJsonStructure([
        'order_id',
        'total_amount',
        'status'
    ]);
    
    $this->assertDatabaseHas('orders', [
        'user_id' => $user->id,
        'total_amount' => 200000
    ]);
    
    // 库存减少
    expect($product->fresh()->stock)->toBe(8);
});

test('库存为 0 时无法下单', function () {
    $user = User::factory()->create();
    $product = Product::factory()->create(['stock' => 0]);
    
    $response = $this->actingAs($user)
        ->post('/orders', [
            'product_id' => $product->id,
            'quantity' => 1
        ]);
    
    $response->assertStatus(422);
    $response->assertJsonValidationErrors(['product_id']);
});

test('未登录用户无法下单', function () {
    $product = Product::factory()->create();
    
    $response = $this->post('/orders', [
        'product_id' => $product->id,
        'quantity' => 1
    ]);
    
    $response->assertStatus(401);
});
```

### 常用测试命令

```
# 跑全部测试
php artisan test

# Pest 可执行文件
./vendor/bin/pest

# 跑指定测试
php artisan test --filter OrderTest

# 生成覆盖率
php artisan test --coverage
```

## 用 Vite 管理前端资源

### Laravel 9+ 默认的 Vite 设置

如果项目里还没有，先安装依赖：

```
npm install
```

`vite.config.js`：

```
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            input: [
                'resources/css/app.css',
                'resources/js/app.js',
            ],
            refresh: true,
        }),
    ],

    server: {
        host: '0.0.0.0',
        hmr: {
            host: 'localhost',
        },
    },
});
```

Blade 模板里引入：

```
DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>{{ config('app.name') }}title>
    
    {{-- ✅ 就这句最简单 --}}
    @vite(['resources/css/app.css', 'resources/js/app.js'])
head>
<body>
    
body>
html>
```

### 必备命令

```
# 开发调试（热更新）
npm run dev

# 构建生产包（压缩优化）
npm run build

# 预览生产包
npm run preview
```

## 总结：最好的投资，就是起步时的扎实准备

项目初期多花点时间打基础，看似慢，其实能省掉后面一堆返工。

好代码不是一次成型，而是不断打磨出来的。没人一上来就完美无缺，真正重要的是：

* **先把地基打牢**：项目初始设置要到位。
* **持续迭代**：定期重构、优化。
* **从错误里学习**：每个 Bug 都是提醒。
* **保持更新**：框架一直在进化。

现在就打开你的 Laravel 项目，逐条打勾看看这 10 项设置落实了多少。如果还一项没做，先从最关键的两个开始：**外键约束** 和 **Service 层架构**。

本博客参考[flyint](https://tishengbao.com)。转载请注明出处！
