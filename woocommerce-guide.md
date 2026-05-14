# WooCommerce 全面使用指南与实战案例

> 从安装配置到定制开发，涵盖产品管理、订单流程、支付/配送、REST API、Hook 系统与完整实战

---

## 目录

1. [WooCommerce 概述与安装](#1-woocommerce-概述与安装)
2. [产品管理全解](#2-产品管理全解)
3. [订单与结账流程](#3-订单与结账流程)
4. [支付网关](#4-支付网关)
5. [配送与物流](#5-配送与物流)
6. [税费配置](#6-税费配置)
7. [优惠券系统](#7-优惠券系统)
8. [模板结构与覆盖](#8-模板结构与覆盖)
9. [Hook 系统与核心钩子](#9-hook-系统与核心钩子)
10. [REST API](#10-rest-api)
11. [常见定制开发场景](#11-常见定制开发场景)
12. [完整实战项目](#12-完整实战项目)

---

## 1. WooCommerce 概述与安装

### 1.1 什么是 WooCommerce

WooCommerce 是 WordPress 上最流行的电商插件，全球市占率约 25%+。它将 WordPress 转变为一个全功能在线商店。

**核心能力：**
- 实体商品 / 虚拟商品 / 可下载商品
- 多种产品类型（简单、变体、组合、外部关联）
- 完整的订单管理与结账流程
- 支付网关集成（Stripe、PayPal、支付宝、微信支付等）
- 配送区域与运费计算
- 税费自动计算
- 优惠券与折扣
- 库存管理
- REST API 支持

### 1.2 安装方式

```bash
# 方式 1：WordPress 后台 → 插件 → 安装插件 → 搜索 WooCommerce → 安装并激活

# 方式 2：WP-CLI
wp plugin install woocommerce --activate

# 方式 3：手动上传
# 下载 https://wordpress.org/plugins/woocommerce/ → 上传到 wp-content/plugins/
```

### 1.3 安装向导快速配置

激活后 WooCommerce 会启动 Setup Wizard，关键步骤：

```
1. 商店详情 → 地址、国家、货币（例如 CNY ¥，保留 2 位小数）
2. 行业类型 → 选择所属行业
3. 产品类型 → 勾选实体/可下载（按需）
4. 业务详情 → 收入规模等
5. 主题选择 → 推荐使用 Storefront 或支持 WooCommerce 的主题
```

### 1.4 WooCommerce 数据表

激活插件后会创建以下数据库表：

```sql
wp_woocommerce_sessions            -- 购物车会话
wp_woocommerce_api_keys            -- REST API 密钥
wp_woocommerce_attribute_taxonomies -- 属性分类
wp_woocommerce_downloadable_product_permissions -- 可下载商品权限
wp_woocommerce_order_items         -- 订单行项目
wp_woocommerce_order_itemmeta      -- 订单行项目元数据
wp_woocommerce_tax_rates           -- 税率
wp_woocommerce_tax_rate_locations  -- 税率地区
wp_woocommerce_shipping_zones      -- 配送区域
wp_woocommerce_shipping_zone_locations -- 配送区域位置
wp_woocommerce_shipping_zone_methods   -- 配送方式
wp_woocommerce_payment_tokens      -- 支付令牌
wp_woocommerce_payment_tokenmeta   -- 支付令牌元数据
```

### 1.5 系统状态调试

后台 → WooCommerce → 状态（System Status）：

```php
// 编程方式获取系统状态
$system_status = WC()->api->get_endpoint_data('/wc/v3/system_status');
// 或直接访问
// GET /wp-json/wc/v3/system_status（需认证）
```

常见状态检查项：
- PHP 版本 ≥ 7.4
- MySQL ≥ 5.6
- WordPress 内存限制 ≥ 256M
- `wp_remote_get()` / cURL 可用
- HTTPS 已启用

---

## 2. 产品管理全解

### 2.1 产品类型

| 类型 | 说明 | 典型场景 |
|------|------|----------|
| Simple（简单） | 无选项的单一产品 | 一本书、一件T恤 |
| Variable（变体） | 有颜色/尺寸等选项 | 服装（多颜色多尺码） |
| Grouped（组合） | 多个简单产品的集合 | 套装家具 |
| External/Affiliate（外部） | 跳转到外部链接购买 | 联盟营销 |
| Virtual（虚拟） | 无实体配送 | 服务、会员 |
| Downloadable（可下载） | 购买后可下载文件 | 电子书、软件、音乐 |

### 2.2 编程创建产品

```php
// 创建简单产品
$product = new WC_Product_Simple();
$product->set_name('我的商品');
$product->set_regular_price('99.00');
$product->set_sale_price('79.00');
$product->set_description('<p>产品描述</p>');
$product->set_short_description('简短描述');
$product->set_sku('SKU-001');
$product->set_stock_quantity(100);
$product->set_manage_stock(true);
$product->set_stock_status('instock');
$product->set_category_ids([12, 15]);
$product->set_tag_ids([8]);
$product->set_image_id(42); // 特色图片附件 ID
$product->set_gallery_image_ids([43, 44]);
$product->set_status('publish');
$product->save();

echo $product->get_id(); // 新产品 ID

// 创建变体产品
$parent = new WC_Product_Variable();
$parent->set_name('T恤 - 多色多码');
$parent->set_attributes([
    'color' => ['name' => '颜色', 'options' => ['红', '蓝', '黑'], 'visible' => true, 'variation' => true],
    'size'  => ['name' => '尺码', 'options' => ['S', 'M', 'L'], 'visible' => true, 'variation' => true],
]);
$parent->save();

// 创建变体
$variation = new WC_Product_Variation();
$variation->set_parent_id($parent->get_id());
$variation->set_attributes(['color' => '红', 'size' => 'M']);
$variation->set_regular_price('129.00');
$variation->set_stock_quantity(50);
$variation->set_manage_stock(true);
$variation->save();
```

### 2.3 批量导入产品（CSV）

后台 → 产品 → 导入 → 选择 CSV 文件。关键列：

| 列名 | 说明 | 示例 |
|------|------|------|
| `ID` / `SKU` | 产品标识 | `SKU-001` |
| `Name` | 产品名称 | `羊毛围巾` |
| `Regular price` | 原价 | `199` |
| `Sale price` | 促销价 | `149` |
| `Stock` | 库存 | `100` |
| `Categories` | 分类（用 `>` 分隔层级） | `服装 > 配饰` |
| `Images` | 图片 URL（逗号分隔） | `https://...` |

### 2.4 产品数据元字段

所有产品信息都存储在 `wp_postmeta` 表中，常见 `meta_key`：

```php
'_price'              // 当前价格（自动计算）
'_regular_price'      // 原价
'_sale_price'         // 促销价
'_sku'                // SKU
'_stock'              // 库存数量
'_stock_status'       // 库存状态：instock/outofstock/onbackorder
'_manage_stock'       // 是否管理库存：yes/no
'_virtual'            // 虚拟商品：yes/no
'_downloadable'       // 可下载：yes/no
'_weight'             // 重量
'_length' / '_width' / '_height' // 尺寸
'_sale_price_dates_from' / '_sale_price_dates_to' // 促销时间
'_product_attributes' // 属性序列化数组
'_purchase_note'      // 购买备注
'_featured'           // 推荐产品：yes/no
_total_sales'         // 累计销售量
```

### 2.5 产品查询

```php
// 标准查询
$products = wc_get_products([
    'limit'   => 10,
    'status'  => 'publish',
    'category'=> ['clothing', 'accessories'], // 分类别名
    'tag'     => ['new-arrival'],
    'featured'=> true,
    'orderby' => 'date',
    'order'   => 'DESC',
]);

// 按属性查询
$products = wc_get_products([
    'attribute' => 'color',
    'attribute_term' => 'red',
]);

// 按价格区间
$products = wc_get_products([
    'meta_key'   => '_price',
    'orderby'    => 'meta_value_num',
    'meta_query' => [[
        'key'     => '_price',
        'value'   => [50, 200],
        'compare' => 'BETWEEN',
        'type'    => 'NUMERIC',
    ]],
]);
```

---

## 3. 订单与结账流程

### 3.1 订单生命周期

```
创建 (pending/cart)
  → 待付款 (pending payment)
    → 正在处理 (processing) ← 付款成功
      → 已完成 (completed)
      → 已退款 (refunded)
    → 失败 (failed)
    → 已取消 (cancelled)
  → 保留 (on-hold) ← 付款待确认
```

### 3.2 编程创建订单

```php
$order = wc_create_order();

// 添加产品
$order->add_product(wc_get_product(123), 2); // 产品ID=123, 数量=2

// 设置地址
$address = [
    'first_name' => '张三',
    'last_name'  => '',
    'email'      => 'zhangsan@example.com',
    'phone'      => '13800138000',
    'address_1'  => '朝阳区某某路 100 号',
    'city'       => '北京',
    'state'      => 'BJ',
    'postcode'   => '100000',
    'country'    => 'CN',
];
$order->set_address($address, 'billing');
$order->set_address($address, 'shipping');

// 添加优惠券
$order->apply_coupon('SUMMER2024');

// 添加费用
$item_fee = new WC_Order_Item_Fee();
$item_fee->set_name('包装费');
$item_fee->set_amount(5);
$item_fee->set_total(5);
$order->add_item($item_fee);

// 计算总计
$order->calculate_totals();

// 更新状态
$order->update_status('completed', '订单已完成，备注信息');

// 添加订单备注（客户可见）
$order->add_order_note('您的订单已发货，快递单号：SF1234567890', 1); // 1=客户可见
$order->add_order_note('仓库已拣货'); // 私有备注

$order->save();
```

### 3.3 查询订单

```php
// 获取单个订单
$order = wc_get_order(1001);
echo $order->get_status();           // 订单状态
echo $order->get_total();            // 总金额
echo $order->get_billing_email();    // 账单邮箱
echo $order->get_payment_method();   // 支付方式

// 遍历订单商品
foreach ($order->get_items() as $item_id => $item) {
    $product = $item->get_product(); // WC_Product 对象
    echo $item->get_name();          // 商品名
    echo $item->get_quantity();      // 数量
    echo $item->get_total();         // 小计
}

// 批量查询订单
$orders = wc_get_orders([
    'limit'        => 20,
    'status'       => ['processing', 'on-hold'],
    'date_created' => '2024-01-01...2024-12-31',
    'customer'     => 'zhangsan@example.com',
    'payment_method' => 'stripe',
]);

// 财务统计
$total_sales = wc_get_orders([
    'status'  => ['completed', 'processing'],
    'return'  => 'total', // 只返回总数
]);
```

### 3.4 结账流程钩子

```php
// 自定义结账字段
add_filter('woocommerce_checkout_fields', function($fields) {
    // 添加自定义字段
    $fields['billing']['billing_id_number'] = [
        'label'       => '身份证号',
        'required'    => true,
        'class'       => ['form-row-wide'],
        'priority'    => 25,
    ];
    // 移除字段
    unset($fields['billing']['billing_company']);
    return $fields;
});

// 验证自定义字段
add_action('woocommerce_checkout_process', function() {
    if (empty($_POST['billing_id_number'])) {
        wc_add_notice('请输入身份证号', 'error');
    }
});

// 保存自定义字段到订单
add_action('woocommerce_checkout_update_order_meta', function($order_id) {
    if (!empty($_POST['billing_id_number'])) {
        update_post_meta($order_id, '_billing_id_number',
            sanitize_text_field($_POST['billing_id_number']));
    }
});

// 在后台订单详情中显示自定义字段
add_action('woocommerce_admin_order_data_after_billing_address', function($order) {
    $id_number = get_post_meta($order->get_id(), '_billing_id_number', true);
    if ($id_number) {
        echo '<p><strong>身份证号：</strong> ' . esc_html($id_number) . '</p>';
    }
});
```

### 3.5 订单状态自定义

```php
// 注册自定义订单状态
add_action('init', function() {
    register_post_status('wc-awaiting-shipment', [
        'label'                     => '待发货',
        'public'                    => true,
        'show_in_admin_status_list' => true,
        'label_count'               => _n_noop(
            '待发货 <span class="count">(%s)</span>',
            '待发货 <span class="count">(%s)</span>'
        ),
    ]);
});

// 添加到 WooCommerce 订单状态列表
add_filter('wc_order_statuses', function($statuses) {
    $statuses['wc-awaiting-shipment'] = '待发货';
    return $statuses;
});

// 支付完成后自动转为"待发货"
add_action('woocommerce_order_status_processing', function($order_id) {
    $order = wc_get_order($order_id);
    $order->update_status('wc-awaiting-shipment', '自动转为待发货');
});
```

---

## 4. 支付网关

### 4.1 内置支付方式

WooCommerce 默认提供：
- **Direct Bank Transfer (BACS)** — 银行转账
- **Check Payments** — 支票付款
- **Cash on Delivery (COD)** — 货到付款
- **PayPal Standard** — PayPal（需在设置中配置邮箱）

### 4.2 配置 Stripe 支付

官方插件：WooCommerce Stripe Payment Gateway

```bash
wp plugin install woocommerce-gateway-stripe --activate
```

后台配置路径：WooCommerce → 设置 → 付款 → Stripe

关键配置项：
```
- Test/Live Publishable Key / Secret Key
- Webhook Secret（接收异步通知）
- 启用 Apple Pay / Google Pay
- 3D Secure 强制（推荐）
```

### 4.3 开发自定义支付网关

```php
// 放在插件中
add_filter('woocommerce_payment_gateways', function($gateways) {
    $gateways[] = 'WC_Gateway_My_Custom';
    return $gateways;
});

class WC_Gateway_My_Custom extends WC_Payment_Gateway {

    public function __construct() {
        $this->id           = 'my_custom';
        $this->title        = '我的自定义支付';
        $this->description  = '通过自定义网关支付';
        $this->method_title = '自定义支付网关';

        $this->init_form_fields();
        $this->init_settings();

        add_action('woocommerce_update_options_payment_gateways_' . $this->id,
            [$this, 'process_admin_options']);
    }

    // 后台设置表单
    public function init_form_fields() {
        $this->form_fields = [
            'enabled' => [
                'title'   => '启用',
                'type'    => 'checkbox',
                'default' => 'no',
            ],
            'api_key' => [
                'title' => 'API Key',
                'type'  => 'text',
            ],
        ];
    }

    // 处理支付
    public function process_payment($order_id) {
        $order = wc_get_order($order_id);

        // TODO: 调用第三方支付 API

        // 支付成功
        $order->payment_complete();
        $order->add_order_note('自定义支付成功');

        // 清空购物车
        WC()->cart->empty_cart();

        return [
            'result'   => 'success',
            'redirect' => $this->get_return_url($order),
        ];
    }
}
```

### 4.4 支付状态回调（Webhook）

```php
// 注册 REST 端点接收支付回调
add_action('rest_api_init', function() {
    register_rest_route('my-payment/v1', '/callback', [
        'methods'             => 'POST',
        'callback'            => 'handle_payment_callback',
        'permission_callback' => '__return_true',
    ]);
});

function handle_payment_callback($request) {
    $params    = $request->get_params();
    $order_id  = $params['order_id'];
    $signature = $params['sign'];

    // 验签
    $expected = hash_hmac('sha256', $order_id, MY_API_SECRET);
    if (!hash_equals($expected, $signature)) {
        return new WP_REST_Response(['error' => 'Invalid signature'], 403);
    }

    $order = wc_get_order($order_id);
    if (!$order) {
        return new WP_REST_Response(['error' => 'Order not found'], 404);
    }

    $order->payment_complete();
    return new WP_REST_Response(['success' => true]);
}
```

---

## 5. 配送与物流

### 5.1 配送区域结构

```
配送区域（Shipping Zone）
  ├── 区域范围（国家/州/邮编）
  └── 配送方式（Shipping Methods）
      ├── Flat Rate（统一费率）
      ├── Free Shipping（免运费）
      └── Local Pickup（自提）
```

### 5.2 编程创建配送区域

```php
// 创建配送区域
$zone = new WC_Shipping_Zone();
$zone->set_zone_name('中国大陆');
$zone->add_location('CN', 'country');
$zone->save();

// 添加统一费率
$flat_rate = $zone->add_shipping_method('flat_rate');
$flat_rate->set_instance_option('cost', '10'); // 基础运费 ¥10

// 添加免运费（满 99 包邮）
$free = $zone->add_shipping_method('free_shipping');
$free->set_instance_option('min_amount', '99');
$free->set_instance_option('requires', 'min_amount');
```

### 5.3 自定义配送计算

```php
// 基于购物车总重量的阶梯运费
add_filter('woocommerce_package_rates', function($rates, $package) {
    $total_weight = 0;
    foreach (WC()->cart->get_cart() as $cart_item) {
        $product = $cart_item['data'];
        $weight  = $product->get_weight();
        $total_weight += $weight * $cart_item['quantity'];
    }

    foreach ($rates as $rate_id => $rate) {
        if ('flat_rate' === $rate->method_id) {
            if ($total_weight <= 1) {
                $rates[$rate_id]->cost = 10; // 1kg 以内 ¥10
            } elseif ($total_weight <= 5) {
                $rates[$rate_id]->cost = 15; // 1-5kg ¥15
            } else {
                $rates[$rate_id]->cost = 15 + ceil($total_weight - 5) * 2;
            }
        }
    }
    return $rates;
}, 10, 2);
```

---

## 6. 税费配置

### 6.1 税设置

WooCommerce → 设置 → 税务

关键选项：
- **启用税费** → 是
- **价格含税** → 根据目标国家决定
- **基于配送地址计税** → 推荐
- **税率类别** → Standard Rate / Reduced Rate / Zero Rate

### 6.2 编程添加税率

```php
// 插入税率为 13%（中国增值税）
WC_Tax::_insert_tax_rate([
    'tax_rate_country'  => 'CN',
    'tax_rate_state'    => '',
    'tax_rate'          => 13.0000,
    'tax_rate_name'     => 'VAT',
    'tax_rate_priority' => 1,
    'tax_rate_compound' => 0,
    'tax_rate_shipping' => 1,
    'tax_rate_order'    => 1,
    'tax_rate_class'    => '', // '' = 标准税率
]);
```

---

## 7. 优惠券系统

### 7.1 优惠券类型

| 类型 | 说明 |
|------|------|
| 百分比折扣 | 按比例减免，如 8 折 |
| 固定购物车折扣 | 整个订单减固定金额 |
| 固定产品折扣 | 单品减固定金额 |

### 7.2 编程创建优惠券

```php
$coupon = new WC_Coupon();

$coupon->set_code('SUMMER2024');
$coupon->set_discount_type('percent'); // percent / fixed_cart / fixed_product
$coupon->set_amount(15);               // 15% 折扣

// 使用限制
$coupon->set_minimum_amount(100);      // 最低消费 ¥100
$coupon->set_maximum_amount(500);      // 最高折扣 ¥500
$coupon->set_individual_use(true);     // 不可与其他优惠叠加
$coupon->set_usage_limit(100);         // 总使用次数限制
$coupon->set_usage_limit_per_user(1);  // 每人限用 1 次

// 时间限制
$coupon->set_date_expires('2024-08-31');

// 适用产品
$coupon->set_product_ids([10, 20]);    // 仅适用指定产品
$coupon->set_excluded_product_ids([99]);
$coupon->set_product_categories(['clothing']);

// 邮件限制
$coupon->set_email_restrictions(['@company.com']);

$coupon->save();
```

### 7.3 编程应用优惠券

```php
// 前端应用
WC()->cart->apply_coupon('SUMMER2024');

// 检查有效性
$valid = WC()->cart->has_discount('SUMMER2024');
$messages = wc_get_notices('error');

// 移除优惠券
WC()->cart->remove_coupon('SUMMER2024');
```

---

## 8. 模板结构与覆盖

### 8.1 模板文件位置

```
wp-content/plugins/woocommerce/templates/
├── archive-product.php         # 产品列表页
├── single-product.php          # 产品详情页
├── content-product.php         # 产品列表中的单个产品块
├── content-single-product.php  # 产品详情内容
├── cart/                       # 购物车模板
│   ├── cart.php
│   ├── cart-totals.php
│   └── mini-cart.php
├── checkout/                   # 结账模板
│   ├── form-checkout.php
│   ├── form-billing.php
│   ├── form-shipping.php
│   ├── form-payment.php
│   └── thankyou.php
├── myaccount/                  # 我的账户
├── emails/                     # 邮件模板
│   ├── admin-new-order.php
│   ├── customer-processing-order.php
│   └── customer-completed-order.php
├── loop/                       # 循环部件
│   ├── add-to-cart.php
│   ├── price.php
│   ├── rating.php
│   └── sale-flash.php
└── global/                     # 全局部件
    ├── breadcrumb.php
    └── sidebar.php
```

### 8.2 模板覆盖（Override）

将需要修改的模板复制到主题中：

```
your-theme/woocommerce/
├── archive-product.php
├── single-product.php
├── content-product.php
├── cart/cart.php
└── emails/customer-processing-order.php
```

**规则：** 主题中的 `woocommerce/` 目录会完全覆盖插件模板。

### 8.3 关键模板函数

```php
// 判断函数
is_woocommerce()          // 任何 WooCommerce 页面
is_shop()                 // 产品列表页
is_product()              // 产品详情页
is_cart()                 // 购物车
is_checkout()             // 结账页
is_account_page()         // 我的账户
woocommerce_page_id()     // 获取页面对应的 ID

// 在产品循环中使用的全局函数
global $product;
$product->get_id();
$product->get_title();
$product->get_price_html();
$product->get_average_rating();
$product->get_review_count();
$product->is_on_sale();
$product->is_in_stock();
$product->is_featured();
$product->get_image();
$product->get_gallery_image_ids();
woocommerce_template_loop_add_to_cart(); // 加入购物车按钮
```

---

## 9. Hook 系统与核心钩子

### 9.1 产品相关钩子

```php
// 单个产品页：在"加入购物车"按钮前添加内容
add_action('woocommerce_before_add_to_cart_button', function() {
    echo '<p class="shipping-note">🚚 满 99 包邮</p>';
});

// 单个产品页：在标题后添加副标题
add_action('woocommerce_single_product_summary', function() {
    echo '<p class="subtitle">限时优惠中</p>';
}, 6); // 在标题(5)之后，价格(10)之前

// 产品选项卡（Tab）
add_filter('woocommerce_product_tabs', function($tabs) {
    $tabs['delivery'] = [
        'title'    => '配送说明',
        'priority' => 30,
        'callback' => function() {
            echo '<p>全国包邮，24 小时内发货。</p>';
        },
    ];
    // 移除评论选项卡
    unset($tabs['reviews']);
    return $tabs;
});

// 修改价格显示格式
add_filter('woocommerce_get_price_html', function($price, $product) {
    if ($product->is_on_sale()) {
        $regular = wc_price($product->get_regular_price());
        $sale    = wc_price($product->get_sale_price());
        $saved   = wc_price($product->get_regular_price() - $product->get_sale_price());
        return "<del>{$regular}</del> <ins>{$sale}</ins> <span class='saved'>省 {$saved}</span>";
    }
    return $price;
}, 10, 2);
```

### 9.2 购物车相关钩子

```php
// 购物车页面添加提示
add_action('woocommerce_before_cart', function() {
    if (WC()->cart->subtotal < 99) {
        $need = 99 - WC()->cart->subtotal;
        wc_print_notice("再买 ¥{$need} 即可享受免运费！", 'notice');
    }
});

// 修改"继续购物"按钮链接
add_filter('woocommerce_return_to_shop_redirect', function($url) {
    return home_url('/products/new-arrivals');
});

// 购物车自动更新数量
add_action('woocommerce_add_to_cart', function($cart_item_key) {
    // 首次加入购物车后的操作
}, 10, 1);

// 购物车中移除商品时的操作
add_action('woocommerce_cart_item_removed', function($cart_item_key, $cart) {
    // 记录移除行为
}, 10, 2);
```

### 9.3 订单状态变更钩子

```php
// 订单完成后发送自定义通知
add_action('woocommerce_order_status_completed', function($order_id) {
    $order = wc_get_order($order_id);
    $customer_email = $order->get_billing_email();

    wp_mail($customer_email, '订单完成',
        "您的订单 #{$order_id} 已完成，感谢购买！");

    // 增加用户积分
    $user_id = $order->get_user_id();
    $points  = $order->get_total() * 10; // 每消费 1 元 = 10 积分
    update_user_meta($user_id, 'loyalty_points',
        (int)get_user_meta($user_id, 'loyalty_points', true) + $points);
});

// 订单状态变更通用钩子
add_action('woocommerce_order_status_changed', function($order_id, $old_status, $new_status, $order) {
    // 状态从 A 变为 B 时触发
}, 10, 4);

// 阻止取消特定订单
add_filter('woocommerce_cancel_unpaid_order', function($cancel, $order) {
    if ($order->get_meta('_vip_order') === 'yes') {
        return false; // VIP 订单不自动取消
    }
    return $cancel;
}, 10, 2);
```

### 9.4 邮件钩子

```php
// 修改邮件标题
add_filter('woocommerce_email_subject_customer_processing_order', function($subject, $order) {
    return "您的订单 #{$order->get_id()} 正在处理中";
}, 10, 2);

// 在邮件中追加内容
add_action('woocommerce_email_after_order_table', function($order, $sent_to_admin, $plain_text, $email) {
    if ($email->id === 'customer_completed_order') {
        echo '<p>感谢您的购买！下次购物使用优惠码 <strong>THANKYOU</strong> 享 9 折。</p>';
    }
}, 10, 4);

// 修改发件人
add_filter('woocommerce_email_from_address', function($email) {
    return 'shop@example.com';
});
```

### 9.5 核心钩子速查表

**产品页：**

| Hook | 位置 |
|------|------|
| `woocommerce_before_single_product` | 产品详情页最顶部 |
| `woocommerce_before_single_product_summary` | 产品概要前（图片区域） |
| `woocommerce_single_product_summary` | 产品概要区 |
| `woocommerce_after_single_product_summary` | 产品概要后 |
| `woocommerce_after_single_product` | 产品详情页最底部 |
| `woocommerce_before_add_to_cart_button` | 加入购物车前 |
| `woocommerce_after_add_to_cart_button` | 加入购物车后 |

**购物车页：**

| Hook | 位置 |
|------|------|
| `woocommerce_before_cart` | 购物车顶部 |
| `woocommerce_cart_contents` | 购物车商品列表 |
| `woocommerce_cart_collaterals` | 购物车汇总区 |
| `woocommerce_after_cart` | 购物车底部 |
| `woocommerce_before_cart_table` | 表格前 |
| `woocommerce_after_cart_table` | 表格后 |

**结账页：**

| Hook | 位置 |
|------|------|
| `woocommerce_before_checkout_form` | 结账表单前 |
| `woocommerce_checkout_before_customer_details` | 客户详情前 |
| `woocommerce_checkout_billing` | 账单区域 |
| `woocommerce_checkout_shipping` | 配送区域 |
| `woocommerce_checkout_before_order_review` | 订单摘要前 |
| `woocommerce_checkout_order_review` | 订单摘要 |
| `woocommerce_after_checkout_form` | 结账表单后 |
| `woocommerce_before_thankyou` | 感谢页顶部 |

---

## 10. REST API

### 10.1 API 认证

后台 → WooCommerce → 设置 → 高级 → REST API → 创建密钥

```bash
# Consumer Key 和 Consumer Secret
# 权限：Read/Write
```

### 10.2 常用 API 端点

```bash
# === 产品 ===
GET  /wp-json/wc/v3/products
GET  /wp-json/wc/v3/products/{id}
GET  /wp-json/wc/v3/products?category=20&per_page=50
POST /wp-json/wc/v3/products        # 创建产品
PUT  /wp-json/wc/v3/products/{id}   # 更新产品
DELETE /wp-json/wc/v3/products/{id} # 删除产品

# === 产品变体 ===
GET  /wp-json/wc/v3/products/{product_id}/variations
POST /wp-json/wc/v3/products/{product_id}/variations

# === 订单 ===
GET  /wp-json/wc/v3/orders
GET  /wp-json/wc/v3/orders?status=processing&after=2024-01-01T00:00:00
POST /wp-json/wc/v3/orders          # 创建订单
PUT  /wp-json/wc/v3/orders/{id}     # 更新订单

# === 订单备注 ===
POST /wp-json/wc/v3/orders/{order_id}/notes

# === 优惠券 ===
GET  /wp-json/wc/v3/coupons
POST /wp-json/wc/v3/coupons

# === 客户 ===
GET  /wp-json/wc/v3/customers

# === 报告 ===
GET  /wp-json/wc/v3/reports/sales
GET  /wp-json/wc/v3/reports/products/totals

# === 设置 ===
GET  /wp-json/wc/v3/settings/general
```

### 10.3 PHP 中调用 REST API

```php
use Automattic\WooCommerce\Client;

$woocommerce = new Client(
    'https://example.com',
    'ck_xxx',  // Consumer Key
    'cs_xxx',  // Consumer Secret
    ['version' => 'wc/v3']
);

// 获取产品列表
$products = $woocommerce->get('products', [
    'per_page' => 20,
    'page'     => 1,
]);

// 创建订单
$order = $woocommerce->post('orders', [
    'payment_method' => 'bacs',
    'billing' => [
        'first_name' => '张三',
        'email'      => 'zhangsan@example.com',
    ],
    'line_items' => [[
        'product_id' => 123,
        'quantity'   => 2,
    ]],
]);
```

---

## 11. 常见定制开发场景

### 11.1 自定义产品类型

```php
// 注册新产品类型
add_action('init', function() {
    class WC_Product_Subscription extends WC_Product {
        public function get_type() {
            return 'subscription';
        }
        public function is_virtual() {
            return true;
        }
    }
});

// 添加到产品类型选择器
add_filter('product_type_selector', function($types) {
    $types['subscription'] = '订阅产品';
    return $types;
});

// 添加自定义数据选项卡
add_filter('woocommerce_product_data_tabs', function($tabs) {
    $tabs['subscription'] = [
        'label'    => '订阅设置',
        'target'   => 'subscription_product_data',
        'class'    => ['show_if_subscription'],
        'priority' => 31,
    ];
    return $tabs;
});

// 选项卡面板内容
add_action('woocommerce_product_data_panels', function() {
    ?>
    <div id="subscription_product_data" class="panel woocommerce_options_panel">
        <?php
        woocommerce_wp_text_input([
            'id'          => '_subscription_period',
            'label'       => '订阅周期（天）',
            'type'        => 'number',
            'description' => '例如 30 表示每月订阅',
        ]);
        ?>
    </div>
    <?php
});

// 保存自定义字段
add_action('woocommerce_process_product_meta', function($post_id) {
    $product = wc_get_product($post_id);
    if (isset($_POST['_subscription_period'])) {
        $product->update_meta_data('_subscription_period',
            absint($_POST['_subscription_period']));
        $product->save();
    }
});
```

### 11.2 批量操作（后台产品列表）

```php
// 添加自定义批量操作
add_filter('bulk_actions-edit-product', function($actions) {
    $actions['mark_featured']  = '设为推荐';
    $actions['unmark_featured'] = '取消推荐';
    return $actions;
});

// 处理批量操作
add_filter('handle_bulk_actions-edit-product', function($redirect, $action, $ids) {
    if ($action === 'mark_featured') {
        foreach ($ids as $id) {
            $product = wc_get_product($id);
            $product->set_featured(true);
            $product->save();
        }
        $redirect = add_query_arg('featured_count', count($ids), $redirect);
    }
    return $redirect;
}, 10, 3);
```

### 11.3 会员功能集成

```php
// 会员专属价格
add_filter('woocommerce_get_price_html', function($price, $product) {
    if (is_user_logged_in() && current_user_can('vip_member')) {
        $vip_price = $product->get_regular_price() * 0.8;
        $original  = wc_price($product->get_regular_price());
        $vip_html  = wc_price($vip_price);
        return "<del>{$original}</del> <ins>{$vip_html}</ins> <span class='vip-badge'>VIP</span>";
    }
    return $price;
}, 10, 2);

// 会员价在购物车中生效
add_action('woocommerce_before_calculate_totals', function($cart) {
    if (!is_user_logged_in() || !current_user_can('vip_member')) return;
    if (is_admin() && !defined('DOING_AJAX')) return;

    foreach ($cart->get_cart() as $cart_item) {
        $product = $cart_item['data'];
        $vip_price = $product->get_regular_price() * 0.8;
        $cart_item['data']->set_price($vip_price);
    }
});
```

### 11.4 最低订单金额

```php
// 低于最低消费不允许结账
add_action('woocommerce_check_cart_items', function() {
    $minimum = 50;
    if (WC()->cart->subtotal < $minimum) {
        wc_add_notice(
            sprintf('最低消费金额为 ¥%s，当前购物车金额为 ¥%s。',
                $minimum, WC()->cart->subtotal),
            'error'
        );
    }
});
```

### 11.5 自定义商品标签/徽章

```php
// 在商品图片上添加自定义徽章
add_action('woocommerce_before_shop_loop_item_title', function() {
    global $product;

    // 新品标签
    $created = strtotime($product->get_date_created());
    if ($created > strtotime('-7 days')) {
        echo '<span class="badge badge-new">NEW</span>';
    }

    // 限时折扣标签
    if ($product->is_on_sale() && $product->get_sale_price() < $product->get_regular_price() * 0.7) {
        echo '<span class="badge badge-hot">🔥 热卖</span>';
    }
}, 9);
```

### 11.6 禁用/简化功能

```php
// 禁用 WooCommerce CSS
add_filter('woocommerce_enqueue_styles', '__return_empty_array');

// 只保留 minimal 需要的样式，在主题自己写

// 简化结账字段
add_filter('woocommerce_checkout_fields', function($fields) {
    // 只保留姓名、电话、地址、城市
    $keep = ['billing_first_name', 'billing_phone', 'billing_address_1', 'billing_city'];
    foreach ($fields['billing'] as $key => $value) {
        if (!in_array($key, $keep)) {
            unset($fields['billing'][$key]);
        }
    }
    return $fields;
});

// 禁用评论
add_filter('woocommerce_product_tabs', function($tabs) {
    unset($tabs['reviews']);
    return $tabs;
});

// 移除关联产品
remove_action('woocommerce_after_single_product_summary', 'woocommerce_output_related_products', 20);
```

### 11.7 多语言

```php
// 使用 WPML 或 Polylang + WooCommerce Multilingual
// 推荐组合：Polylang + Hyyan WooCommerce Polylang Integration

// 手工翻译产品（不依赖插件）
// 为每个语言版本创建独立产品 + 用 postmeta 关联
$product->update_meta_data('_translation_en_id', $english_product_id);
```

---

## 12. 完整实战项目

### 12.1 自动化「满减促销」插件

自动在购物车中应用满减规则：

```php
<?php
/**
 * Plugin Name: Auto Tiered Discount
 * Description: 购物车自动满减：满 200 减 20，满 500 减 80
 * Version:     1.0.0
 * Author:      khz
 */

defined('ABSPATH') || exit;

class Auto_Tiered_Discount {

    public function __construct() {
        // 在购物车计算前应用折扣
        add_action('woocommerce_cart_calculate_fees', [$this, 'apply_discount']);
        // 在购物车表格中显示提示
        add_action('woocommerce_before_cart', [$this, 'show_promotion_notice']);
        add_action('woocommerce_before_checkout_form', [$this, 'show_promotion_notice']);
    }

    /**
     * 满减规则（可配置）
     */
    public function get_tiers() {
        return [
            500 => 80,
            200 => 20,
        ];
    }

    public function apply_discount($cart) {
        if (is_admin() && !defined('DOING_AJAX')) return;

        $subtotal = $cart->subtotal;

        foreach ($this->get_tiers() as $threshold => $discount) {
            if ($subtotal >= $threshold) {
                $discount_amount = -$discount;
                $label = sprintf('满 %d 减 %d', $threshold, $discount);
                $cart->add_fee($label, $discount_amount);
                break; // 只应用最高档
            }
        }
    }

    public function show_promotion_notice() {
        $subtotal = WC()->cart->subtotal;
        $tiers    = $this->get_tiers();
        ksort($tiers); // 按门槛升序

        $next = null;
        foreach ($tiers as $threshold => $discount) {
            if ($subtotal < $threshold) {
                $next = [$threshold, $discount];
                break;
            }
        }

        if ($next) {
            $need = $next[0] - $subtotal;
            wc_print_notice(
                sprintf('再买 ¥%s 即可享受满 %d 减 %d 优惠！', $need, $next[0], $next[1]),
                'notice'
            );
        } elseif ($subtotal > 0) {
            wc_print_notice('✅ 已享受最高档满减优惠！', 'success');
        }
    }
}

new Auto_Tiered_Discount();
```

### 12.2 自定义「订单导出 CSV」功能

在后台添加一键导出订单为 CSV 的功能：

```php
add_action('admin_menu', function() {
    add_submenu_page(
        'woocommerce',
        '导出订单',
        '导出订单',
        'manage_woocommerce',
        'wc-orders-export',
        'render_orders_export_page'
    );
});

function render_orders_export_page() {
    // 处理导出
    if (isset($_POST['export_orders'])) {
        check_admin_referer('wc_export_orders');

        $orders = wc_get_orders([
            'limit'        => -1,
            'status'       => ['completed', 'processing'],
            'date_created' => date('Y-m-01') . '...' . date('Y-m-t'),
        ]);

        header('Content-Type: text/csv; charset=utf-8');
        header('Content-Disposition: attachment; filename="orders-' . date('Y-m-d') . '.csv"');

        $output = fopen('php://output', 'w');
        // BOM for Excel
        fputs($output, "\xEF\xBB\xBF");

        fputcsv($output, ['订单号', '日期', '客户', '商品', '数量', '金额', '支付方式', '状态']);

        foreach ($orders as $order) {
            $items = [];
            foreach ($order->get_items() as $item) {
                $items[] = $item->get_name() . '×' . $item->get_quantity();
            }

            fputcsv($output, [
                $order->get_order_number(),
                $order->get_date_created()->date('Y-m-d H:i'),
                $order->get_billing_first_name() . ' ' . $order->get_billing_last_name(),
                implode('; ', $items),
                $order->get_item_count(),
                $order->get_total(),
                $order->get_payment_method_title(),
                wc_get_order_status_name($order->get_status()),
            ]);
        }
        fclose($output);
        exit;
    }
    ?>
    <div class="wrap">
        <h1>导出订单</h1>
        <form method="post">
            <?php wp_nonce_field('wc_export_orders'); ?>
            <p>导出本月的已完成和处理中订单</p>
            <?php submit_button('导出 CSV', 'primary', 'export_orders'); ?>
        </form>
    </div>
    <?php
}
```

### 12.3 产品「售罄倒计时」插件

在产品页显示基于销售速度的倒计时：

```php
// 在产品页显示"热销倒计时"
add_action('woocommerce_single_product_summary', function() {
    global $product;

    // 检查是否低库存
    $stock = $product->get_stock_quantity();
    if (!$stock || $stock > 20) return;

    $total_sales = (int) $product->get_meta('_total_sales') ?: 1;

    ?>
    <div class="stock-warning" style="background:#fff3cd; padding:10px; border-radius:4px; margin:10px 0;">
        🔥 仅剩 <strong><?php echo $stock; ?></strong> 件！
        已有 <?php echo $total_sales; ?> 人购买
        <span id="urgency-countdown" class="countdown">
            预计<?php echo ceil($stock / max($total_sales, 1) * 7); ?>天内售罄
        </span>
    </div>
    <?php
}, 25);
```

---

## 附录：常用代码片断速查

### 获取当前用户购物车

```php
$cart = WC()->cart;
$items = $cart->get_cart();
$subtotal = $cart->subtotal;
$total = $cart->total;
$count = $cart->get_cart_contents_count();
```

### 获取产品对象

```php
$product = wc_get_product($product_id);
$product = $item->get_product(); // 从订单 item 获取
global $product;  // 在 WooCommerce 循环中
```

### 显示价格

```php
echo wc_price(99.50); // ¥99.50
echo $product->get_price_html();
echo wc_format_sale_price($regular, $sale);
```

### 库存操作

```php
// 增加库存
wc_update_product_stock($product_id, 50, 'increase');
// 减少库存
wc_update_product_stock($product_id, 10, 'decrease');
// 设置库存
wc_update_product_stock($product_id, 100, 'set');
```

### 发送邮件

```php
// 触发订单邮件
$mailer = WC()->mailer();
$mailer->emails['WC_Email_Customer_Processing_Order']->trigger($order_id);
```

---

> **文档版本**: 1.0  
> **更新日期**: 2026-05-14  
> **适用版本**: WooCommerce 8.x+ / WordPress 6.x+ / PHP 7.4+
