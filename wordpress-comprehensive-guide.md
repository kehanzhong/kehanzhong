# WordPress 全面开发指南

> 从基础架构到高级实战，涵盖钩子、插件、主题、REST API、性能优化与安全

---

## 目录

1. [核心架构与执行流程](#1-核心架构与执行流程)
2. [数据库结构](#2-数据库结构)
3. [安装与 wp-config.php 详解](#3-安装与-wp-configphp-详解)
4. [钩子系统 (Hooks)](#4-钩子系统-hooks)
5. [插件开发实战](#5-插件开发实战)
6. [主题开发实战](#6-主题开发实战)
7. [REST API](#7-rest-api)
8. [WP-CLI 命令行工具](#8-wp-cli-命令行工具)
9. [性能优化与缓存](#9-性能优化与缓存)
10. [安全性加固](#10-安全性加固)
11. [多站点 (Multisite)](#11-多站点-multisite)
12. [完整实战项目](#12-完整实战项目)

---

## 1. 核心架构与执行流程

### 1.1 请求生命周期

```
HTTP 请求
  → index.php
    → wp-blog-header.php
      → wp-load.php (加载 wp-config.php, wp-settings.php)
      → wp() (解析 URL, 构建查询)
      → template-loader.php (选择模板)
        → 对应模板文件 (single.php, page.php 等)
```

### 1.2 核心文件职责

| 文件 | 作用 |
|------|------|
| `index.php` | 入口文件，加载 WordPress 环境 |
| `wp-config.php` | 数据库连接、密钥、调试模式等配置 |
| `wp-settings.php` | 加载核心文件、插件、主题 |
| `wp-load.php` | 加载 wp-config.php 并初始化 |
| `wp-includes/` | 核心函数库（functions.php, post.php, user.php 等） |
| `wp-admin/` | 后台管理界面 |
| `wp-content/` | 用户内容：插件、主题、上传文件 |

### 1.3 加载顺序

```
1. wp-config.php
2. wp-settings.php
3. 加载必须使用的插件 (mu-plugins)
4. 加载激活的插件
5. 加载主题 functions.php
6. 触发 'init' action
7. 解析请求 URL
8. 触发 'wp' action
9. 加载模板文件
```

---

## 2. 数据库结构

### 2.1 默认数据表（共 12 张）

```sql
-- 核心内容
wp_posts          -- 文章、页面、附件、自定义文章类型
wp_postmeta       -- 文章元数据
wp_comments       -- 评论
wp_commentmeta    -- 评论元数据

-- 分类系统
wp_terms          -- 术语（分类名、标签名）
wp_termmeta       -- 术语元数据
wp_term_taxonomy  -- 术语分类法（category, post_tag, 自定义分类法）
wp_term_relationships -- 文章与术语的关联

-- 用户系统
wp_users          -- 用户
wp_usermeta       -- 用户元数据

-- 选项与链接
wp_options        -- 全局设置、插件配置、主题选项
wp_links          -- 友情链接（已弃用但保留）
```

### 2.2 `$wpdb` 全局数据库操作

```php
global $wpdb;

// 查询
$results = $wpdb->get_results("SELECT * FROM {$wpdb->posts} WHERE post_status = 'publish'");

// 单行
$post = $wpdb->get_row("SELECT * FROM {$wpdb->posts} WHERE ID = 1");

// 单值
$count = $wpdb->get_var("SELECT COUNT(*) FROM {$wpdb->users}");

// 插入
$wpdb->insert(
    $wpdb->postmeta,
    ['post_id' => 1, 'meta_key' => 'views', 'meta_value' => 100],
    ['%d', '%s', '%d']
);

// 更新
$wpdb->update(
    $wpdb->postmeta,
    ['meta_value' => 200],
    ['post_id' => 1, 'meta_key' => 'views'],
    ['%d'],
    ['%d', '%s']
);

// 删除
$wpdb->delete($wpdb->postmeta, ['post_id' => 1, 'meta_key' => 'views']);

// 查看最后执行的 SQL
echo $wpdb->last_query;
```

---

## 3. 安装与 wp-config.php 详解

### 3.1 环境要求

- PHP 7.4+ （推荐 PHP 8.1+）
- MySQL 5.7+ / MariaDB 10.3+
- Nginx / Apache
- HTTPS 支持

### 3.2 wp-config.php 关键配置

```php
<?php
// ===== 数据库配置 =====
define('DB_NAME',     'wordpress');
define('DB_USER',     'root');
define('DB_PASSWORD', 'password');
define('DB_HOST',     'localhost');
define('DB_CHARSET',  'utf8mb4');
define('DB_COLLATE',  'utf8mb4_unicode_ci');
$table_prefix = 'wp_';

// ===== 安全密钥（从 https://api.wordpress.org/secret-key/1.1/salt/ 生成）=====
define('AUTH_KEY',         'put your unique phrase here');
define('SECURE_AUTH_KEY',  'put your unique phrase here');
define('LOGGED_IN_KEY',    'put your unique phrase here');
define('NONCE_KEY',        'put your unique phrase here');
define('AUTH_SALT',        'put your unique phrase here');
define('SECURE_AUTH_SALT', 'put your unique phrase here');
define('LOGGED_IN_SALT',   'put your unique phrase here');
define('NONCE_SALT',       'put your unique phrase here');

// ===== 调试模式 =====
define('WP_DEBUG', true);
define('WP_DEBUG_LOG', true);   // 日志写入 /wp-content/debug.log
define('WP_DEBUG_DISPLAY', false); // 不在页面显示错误
define('SCRIPT_DEBUG', true);    // 使用未压缩的 JS/CSS

// ===== 性能与安全 =====
define('WP_POST_REVISIONS', 10);     // 限制文章修订版本数
define('AUTOSAVE_INTERVAL', 120);    // 自动保存间隔（秒）
define('DISALLOW_FILE_EDIT', true);  // 禁用后台文件编辑
define('FORCE_SSL_ADMIN', true);     // 强制后台 SSL
define('WP_AUTO_UPDATE_CORE', false);// 禁用核心自动更新

// ===== 内存与文件 =====
define('WP_MEMORY_LIMIT', '256M');
define('WP_MAX_MEMORY_LIMIT', '512M');
define('UPLOAD_MAX_FILESIZE', '64M');

// ===== 多站点 =====
// define('WP_ALLOW_MULTISITE', true);

// ===== 自定义目录 =====
// define('WP_CONTENT_DIR', dirname(__FILE__) . '/custom-content');
// define('WP_CONTENT_URL', 'https://example.com/custom-content');
```

---

## 4. 钩子系统 (Hooks)

### 4.1 Action Hooks（动作钩子）

在特定时间点**执行**自定义逻辑，不返回值。

```php
// === 注册 Action ===
add_action('hook_name', 'callback_function', $priority, $accepted_args);

// 示例 1：在 <head> 中插入 Google Analytics
add_action('wp_head', function() {
    echo "<!-- Google Analytics Code -->";
});

// 示例 2：文章发布后发送通知
add_action('publish_post', 'notify_on_publish', 10, 2);
function notify_on_publish($post_id, $post) {
    $admin_email = get_option('admin_email');
    wp_mail($admin_email, '新文章发布', "文章「{$post->post_title}」已发布。");
}

// 示例 3：在 init 时注册自定义文章类型
add_action('init', function() {
    register_post_type('book', [
        'public'  => true,
        'label'   => '图书',
        'supports' => ['title', 'editor', 'thumbnail'],
        'has_archive' => true,
    ]);
});
```

### 4.2 Filter Hooks（过滤钩子）

接收一个值，修改后**返回**。

```php
// === 注册 Filter ===
add_filter('hook_name', 'callback_function', $priority, $accepted_args);

// 示例 1：修改文章内容，自动追加版权信息
add_filter('the_content', function($content) {
    if (is_single()) {
        $content .= '<p class="copyright">© ' . date('Y') . ' 版权所有</p>';
    }
    return $content;
});

// 示例 2：自定义摘要长度
add_filter('excerpt_length', function($length) {
    return 30; // 改为 30 个词
});

// 示例 3：修改上传文件类型白名单
add_filter('upload_mimes', function($mimes) {
    $mimes['svg'] = 'image/svg+xml';
    return $mimes;
});
```

### 4.3 Action 和 Filter 的区别

| | Action | Filter |
|---|---|---|
| 目的 | 在特定时机执行操作 | 修改数据并返回 |
| 返回值 | 不需要返回 | 必须返回修改后的值 |
| 典型场景 | 发送邮件、注册 CPT、入队列脚本 | 修改内容、更改查询参数、过滤输出 |

### 4.4 常用 Hooks 速查表

#### 系统级 Action Hooks

| Hook | 触发时机 | 典型用途 |
|------|----------|----------|
| `init` | WordPress 初始化完成后 | 注册 CPT、分类法、短代码 |
| `admin_init` | 后台初始化 | 注册后台设置 |
| `wp_loaded` | WordPress 完全加载后 | 表单处理、重定向 |
| `wp` | 主查询设置完成后 | 修改主查询 |
| `wp_enqueue_scripts` | 前台脚本入队列 | 加载 CSS/JS |
| `admin_enqueue_scripts` | 后台脚本入队列 | 加载后台 CSS/JS |
| `save_post` | 文章保存时 | 同步自定义字段、清理缓存 |

#### 模板级 Action Hooks

| Hook | 触发时机 |
|------|----------|
| `wp_head` | `<head>` 标签内 |
| `wp_body_open` | `<body>` 标签后 |
| `wp_footer` | `</body>` 前 |
| `loop_start` | 主循环开始 |
| `loop_end` | 主循环结束 |

#### 常用 Filter Hooks

| Hook | 过滤目标 | 参数 |
|------|----------|------|
| `the_content` | 文章内容 | `$content` |
| `the_title` | 文章标题 | `$title, $post_id` |
| `the_excerpt` | 文章摘要 | `$excerpt` |
| `excerpt_length` | 摘要长度 | `$length` |
| `excerpt_more` | 摘要后缀 "..." | `$more` |
| `body_class` | body CSS 类 | `$classes` |
| `post_class` | 文章容器 CSS 类 | `$classes` |
| `wp_nav_menu_items` | 导航菜单项 | `$items, $args` |
| `query_vars` | 查询变量 | `$vars` |
| `posts_where` | WHERE 子句 | `$where` |
| `posts_orderby` | ORDER BY 子句 | `$orderby` |
| `wp_mail_from` | 发件人邮箱 | `$email` |
| `upload_mimes` | 允许上传的文件类型 | `$mimes` |
| `the_password_form` | 密码保护表单 HTML | `$output` |

### 4.5 优先级与执行顺序

```php
// 优先级：数字越小越先执行，默认 10
add_action('wp_footer', 'second', 20);  // 后执行
add_action('wp_footer', 'first', 5);    // 先执行
add_action('wp_footer', 'middle');      // 默认 10，中间执行
```

### 4.6 移除 Hook

```php
// 移除指定 hook
remove_action('wp_head', 'wp_generator');
remove_filter('the_content', 'wpautop');

// 移除所有同类型 hook
remove_all_actions('wp_head');
remove_all_filters('the_content', 99); // 仅移除优先级 ≤99 的
```

---

## 5. 插件开发实战

### 5.1 最小插件结构

```
my-plugin/
├── my-plugin.php      # 主插件文件（必须）
├── includes/          # 功能模块
│   ├── class-admin.php
│   └── class-frontend.php
├── assets/
│   ├── css/
│   └── js/
└── languages/         # 国际化
    └── my-plugin-zh_CN.po
```

### 5.2 插件文件头

```php
<?php
/**
 * Plugin Name: My Awesome Plugin
 * Plugin URI:  https://example.com/my-plugin
 * Description: 插件描述——展示 WordPress 插件开发最佳实践
 * Version:     1.0.0
 * Author:      khz
 * Author URI:  https://kehanzhong.github.io
 * License:     GPL-2.0+
 * Text Domain: my-plugin
 * Domain Path: /languages
 * Requires PHP: 7.4
 * Requires at least: 5.8
 */

// 防止直接访问
defined('ABSPATH') || exit;

// 定义插件常量
define('MY_PLUGIN_VERSION', '1.0.0');
define('MY_PLUGIN_PATH', plugin_dir_path(__FILE__));
define('MY_PLUGIN_URL', plugin_dir_url(__FILE__));
```

### 5.3 插件激活与卸载

```php
// 激活钩子
register_activation_hook(__FILE__, 'my_plugin_activate');
function my_plugin_activate() {
    // 创建数据库表
    // 设置默认选项
    add_option('my_plugin_settings', [
        'enable_feature_x' => true,
        'api_key'          => '',
    ]);
    // 设置重写规则后刷新
    flush_rewrite_rules();
}

// 卸载钩子
register_deactivation_hook(__FILE__, 'my_plugin_deactivate');
function my_plugin_deactivate() {
    // 清理定时任务
    wp_clear_scheduled_hook('my_plugin_cron');
    flush_rewrite_rules();
}

// 完整卸载（通过 uninstall.php 或回调）
register_uninstall_hook(__FILE__, 'my_plugin_uninstall');
function my_plugin_uninstall() {
    // 删除所有插件数据
    delete_option('my_plugin_settings');
    global $wpdb;
    $wpdb->query("DROP TABLE IF EXISTS {$wpdb->prefix}my_plugin_data");
}
```

### 5.4 后台菜单与设置页

```php
// 添加顶级菜单
add_action('admin_menu', function() {
    // 顶级菜单
    add_menu_page(
        'My Plugin',           // 页面标题
        'My Plugin',           // 菜单标题
        'manage_options',      // 权限
        'my-plugin',           // 菜单别名 (slug)
        'my_plugin_admin_page',// 回调函数
        'dashicons-admin-generic',
        30
    );

    // 子菜单
    add_submenu_page(
        'my-plugin',
        '设置',
        '设置',
        'manage_options',
        'my-plugin-settings',
        'my_plugin_settings_page'
    );
});

function my_plugin_admin_page() {
    ?>
    <div class="wrap">
        <h1>My Plugin Dashboard</h1>
        <p>欢迎使用 My Plugin。</p>
    </div>
    <?php
}

function my_plugin_settings_page() {
    // 检查权限
    if (!current_user_can('manage_options')) {
        wp_die('无权访问');
    }

    // 处理表单提交
    if (isset($_POST['submit'])) {
        check_admin_referer('my_plugin_settings');
        update_option('my_plugin_settings', [
            'api_key' => sanitize_text_field($_POST['api_key']),
            'enable_feature' => isset($_POST['enable_feature']),
        ]);
        echo '<div class="notice notice-success"><p>设置已保存</p></div>';
    }

    $settings = get_option('my_plugin_settings', []);
    ?>
    <div class="wrap">
        <h1>My Plugin 设置</h1>
        <form method="post">
            <?php wp_nonce_field('my_plugin_settings'); ?>
            <table class="form-table">
                <tr>
                    <th><label for="api_key">API Key</label></th>
                    <td>
                        <input type="text" id="api_key" name="api_key"
                               value="<?php echo esc_attr($settings['api_key'] ?? ''); ?>"
                               class="regular-text">
                    </td>
                </tr>
                <tr>
                    <th>启用功能</th>
                    <td>
                        <label>
                            <input type="checkbox" name="enable_feature"
                                <?php checked($settings['enable_feature'] ?? false); ?>>
                            启用 XXX 功能
                        </label>
                    </td>
                </tr>
            </table>
            <?php submit_button(); ?>
        </form>
    </div>
    <?php
}
```

### 5.5 使用 Settings API（推荐方式）

```php
add_action('admin_init', function() {
    // 注册设置
    register_setting('my_plugin_options_group', 'my_plugin_settings', [
        'type'    => 'array',
        'sanitize_callback' => 'my_plugin_sanitize_settings',
    ]);

    // 添加设置节
    add_settings_section(
        'my_plugin_main_section',
        '主要设置',
        function() { echo '<p>配置插件的主要参数。</p>'; },
        'my-plugin-settings'
    );

    // 添加设置字段
    add_settings_field(
        'api_key',
        'API Key',
        function() {
            $options = get_option('my_plugin_settings');
            echo '<input type="text" name="my_plugin_settings[api_key]" value="'
                 . esc_attr($options['api_key'] ?? '')
                 . '" class="regular-text">';
        },
        'my-plugin-settings',
        'my_plugin_main_section'
    );
});

function my_plugin_sanitize_settings($input) {
    $input['api_key'] = sanitize_text_field($input['api_key'] ?? '');
    $input['enable_feature'] = !empty($input['enable_feature']);
    return $input;
}
```

### 5.6 短代码 (Shortcode)

```php
// 基础短代码
add_shortcode('greeting', function($atts, $content = null) {
    $atts = shortcode_atts([
        'name' => 'World',
    ], $atts, 'greeting');

    return sprintf('<p class="greeting">Hello, %s!</p>', esc_html($atts['name']));
});
// 使用：[greeting name="khz"]

// 包裹内容短代码
add_shortcode('box', function($atts, $content = null) {
    $atts = shortcode_atts([
        'color' => 'blue',
    ], $atts);
    return sprintf(
        '<div class="box box-%s">%s</div>',
        esc_attr($atts['color']),
        do_shortcode($content) // 递归解析嵌套短代码
    );
});
// 使用：[box color="red"]内容[/box]

// 在 PHP 中调用短代码
echo do_shortcode('[greeting name="程序员"]');
```

### 5.7 自定义文章类型 (CPT) 完整示例

```php
add_action('init', function() {
    $labels = [
        'name'               => '产品',
        'singular_name'      => '产品',
        'add_new'            => '添加产品',
        'add_new_item'       => '添加新产品',
        'edit_item'          => '编辑产品',
        'search_items'       => '搜索产品',
        'not_found'          => '没有找到产品',
        'all_items'          => '所有产品',
        'menu_name'          => '产品中心',
        'name_admin_bar'     => '产品',
    ];

    register_post_type('product', [
        'labels'       => $labels,
        'public'       => true,
        'has_archive'  => true,
        'rewrite'      => ['slug' => 'products'],
        'supports'     => ['title', 'editor', 'thumbnail', 'excerpt', 'custom-fields'],
        'menu_icon'    => 'dashicons-cart',
        'show_in_rest' => true,  // 启用古腾堡编辑器
        'taxonomies'   => ['category', 'post_tag'],
    ]);

    // 注册自定义分类法
    register_taxonomy('product_category', 'product', [
        'labels'       => ['name' => '产品分类'],
        'hierarchical' => true,  // 类似分类目录
        'show_in_rest' => true,
        'rewrite'      => ['slug' => 'product-category'],
    ]);

    register_taxonomy('product_tag', 'product', [
        'labels'       => ['name' => '产品标签'],
        'hierarchical' => false, // 类似标签
        'show_in_rest' => true,
        'rewrite'      => ['slug' => 'product-tag'],
    ]);
});
```

### 5.8 自定义元数据框 (Meta Box)

```php
// 添加元数据框
add_action('add_meta_boxes', function() {
    add_meta_box(
        'product_details',
        '产品详情',
        'render_product_meta_box',
        'product',
        'normal',    // normal / side / advanced
        'high'       // high / core / default / low
    );
});

function render_product_meta_box($post) {
    // 添加 nonce 验证
    wp_nonce_field('product_details_nonce', 'product_details_nonce_field');

    $price = get_post_meta($post->ID, '_product_price', true);
    $sku   = get_post_meta($post->ID, '_product_sku', true);
    ?>
    <table class="form-table">
        <tr>
            <th><label for="product_price">价格 (¥)</label></th>
            <td>
                <input type="number" id="product_price" name="product_price"
                       value="<?php echo esc_attr($price); ?>" step="0.01" min="0">
            </td>
        </tr>
        <tr>
            <th><label for="product_sku">SKU</label></th>
            <td>
                <input type="text" id="product_sku" name="product_sku"
                       value="<?php echo esc_attr($sku); ?>" class="regular-text">
            </td>
        </tr>
    </table>
    <?php
}

// 保存元数据
add_action('save_post', function($post_id) {
    // 安全检查
    if (defined('DOING_AUTOSAVE') && DOING_AUTOSAVE) return;
    if (!current_user_can('edit_post', $post_id)) return;
    if (!isset($_POST['product_details_nonce_field'])) return;
    if (!wp_verify_nonce($_POST['product_details_nonce_field'], 'product_details_nonce')) return;

    if (isset($_POST['product_price'])) {
        update_post_meta($post_id, '_product_price', floatval($_POST['product_price']));
    }
    if (isset($_POST['product_sku'])) {
        update_post_meta($post_id, '_product_sku', sanitize_text_field($_POST['product_sku']));
    }
});
```

### 5.9 AJAX 处理

```php
// === 前端 JS ===
add_action('wp_enqueue_scripts', function() {
    wp_enqueue_script('my-ajax', MY_PLUGIN_URL . 'assets/js/ajax.js', ['jquery'], '1.0', true);
    wp_localize_script('my-ajax', 'MyAjax', [
        'ajax_url' => admin_url('admin-ajax.php'),
        'nonce'    => wp_create_nonce('my_ajax_nonce'),
    ]);
});

// === JS 文件 (ajax.js) ===
/*
jQuery(document).ready(function($) {
    $('#load-more').on('click', function() {
        $.ajax({
            url: MyAjax.ajax_url,
            type: 'POST',
            data: {
                action: 'load_more_posts',
                nonce: MyAjax.nonce,
                page: 2
            },
            success: function(response) {
                if (response.success) {
                    $('#post-list').append(response.data.html);
                }
            }
        });
    });
});
*/

// === PHP 处理（针对已登录用户）===
add_action('wp_ajax_load_more_posts', 'handle_load_more_posts');

// PHP 处理（针对未登录用户）
add_action('wp_ajax_nopriv_load_more_posts', 'handle_load_more_posts');

function handle_load_more_posts() {
    // 验证 nonce
    check_ajax_referer('my_ajax_nonce', 'nonce');

    $page = intval($_POST['page'] ?? 1);

    $query = new WP_Query([
        'post_type'      => 'post',
        'posts_per_page' => 5,
        'paged'          => $page,
    ]);

    ob_start();
    while ($query->have_posts()) {
        $query->the_post();
        echo '<article><h3>' . get_the_title() . '</h3></article>';
    }
    wp_reset_postdata();
    $html = ob_get_clean();

    wp_send_json_success([
        'html'      => $html,
        'has_more'  => $page < $query->max_num_pages,
    ]);
}
```

---

## 6. 主题开发实战

### 6.1 经典主题文件结构

```
my-theme/
├── style.css            # 主题样式表 + 主题头信息（必须）
├── index.php            # 主模板（必须）
├── functions.php        # 主题函数
├── screenshot.png       # 主题缩略图 (1200×900)
│
├── header.php           # 页头
├── footer.php           # 页脚
├── sidebar.php          # 侧边栏
│
├── single.php           # 单篇文章
├── page.php             # 静态页面
├── archive.php          # 归档页
├── search.php           # 搜索结果
├── 404.php              # 404 页面
├── home.php             # 博客首页
├── front-page.php       # 网站首页
│
├── category.php         # 分类归档
├── tag.php              # 标签归档
├── author.php           # 作者归档
├── date.php             # 日期归档
│
├── attachment.php       # 附件页
├── image.php            # 图片附件
│
├── comments.php         # 评论模板
├── searchform.php       # 搜索表单
│
├── assets/
│   ├── css/
│   ├── js/
│   └── images/
│
├── template-parts/      # 模板部件
│   ├── content.php
│   ├── content-single.php
│   └── content-none.php
│
└── inc/                 # 功能模块
    ├── customizer.php
    ├── hooks.php
    └── widgets.php
```

### 6.2 style.css 主题头

```css
/*
Theme Name: My Custom Theme
Theme URI: https://example.com
Author: khz
Author URI: https://kehanzhong.github.io
Description: 一个自定义 WordPress 主题
Version: 1.0.0
Requires at least: 5.8
Tested up to: 6.4
Requires PHP: 7.4
License: GPL-2.0+
Text Domain: my-theme
*/
```

### 6.3 模板层级 (Template Hierarchy)

WordPress 根据 URL 自动选择模板文件，优先级从高到低：

```
首页：
  front-page.php → home.php → index.php

单篇文章：
  single-{post_type}.php → single.php → singular.php → index.php

静态页面：
  {slug}.php → page-{id}.php → page.php → singular.php → index.php

分类归档：
  category-{slug}.php → category-{id}.php → category.php → archive.php → index.php

标签归档：
  tag-{slug}.php → tag-{id}.php → tag.php → archive.php → index.php

自定义分类法：
  taxonomy-{taxonomy}-{term}.php → taxonomy-{taxonomy}.php → taxonomy.php → archive.php → index.php

自定义文章类型归档：
  archive-{post_type}.php → archive.php → index.php

作者归档：
  author-{nicename}.php → author-{id}.php → author.php → archive.php → index.php

404：
  404.php → index.php

搜索结果：
  search.php → index.php
```

### 6.4 functions.php 核心配置

```php
<?php
/**
 * 主题设置
 */
function my_theme_setup() {
    // === 主题支持 ===
    add_theme_support('title-tag');                        // 自动生成 <title>
    add_theme_support('post-thumbnails');                  // 特色图片
    add_theme_support('custom-logo', [
        'height' => 100,
        'width'  => 400,
    ]);
    add_theme_support('html5', [
        'comment-list', 'comment-form',
        'search-form', 'gallery', 'caption',
    ]);
    add_theme_support('responsive-embeds');                // 响应式嵌入
    add_theme_support('align-wide');                       // 宽对齐
    add_theme_support('customize-selective-refresh-widgets');

    // === 注册导航菜单 ===
    register_nav_menus([
        'primary' => '主导航',
        'footer'  => '页脚导航',
        'mobile'  => '移动端导航',
    ]);

    // === 设置图片尺寸 ===
    update_option('thumbnail_size_w', 300);
    update_option('thumbnail_size_h', 200);
    add_image_size('hero', 1200, 400, true); // 硬裁剪

    // === 国际化 ===
    load_theme_textdomain('my-theme', get_template_directory() . '/languages');
}
add_action('after_setup_theme', 'my_theme_setup');

// === 注册侧边栏 ===
add_action('widgets_init', function() {
    register_sidebar([
        'name'          => '主侧边栏',
        'id'            => 'sidebar-main',
        'before_widget' => '<div class="widget %2$s">',
        'after_widget'  => '</div>',
        'before_title'  => '<h3 class="widget-title">',
        'after_title'   => '</h3>',
    ]);
    register_sidebar([
        'name' => '页脚 1',
        'id'   => 'footer-1',
    ]);
});

// === 加载资源 ===
add_action('wp_enqueue_scripts', function() {
    $version = wp_get_theme()->get('Version');

    wp_enqueue_style('my-theme', get_stylesheet_uri(), [], $version);
    wp_enqueue_style('my-theme-main', get_template_directory_uri() . '/assets/css/main.css', [], $version);

    wp_enqueue_script('my-theme-main', get_template_directory_uri() . '/assets/js/main.js', [], $version, true);

    // 仅单页加载评论回复脚本
    if (is_singular() && comments_open() && get_option('thread_comments')) {
        wp_enqueue_script('comment-reply');
    }

    // 传递数据到 JS
    wp_localize_script('my-theme-main', 'ThemeData', [
        'ajax_url' => admin_url('admin-ajax.php'),
        'theme_url'=> get_template_directory_uri(),
    ]);
});
```

### 6.5 主循环 (The Loop)

```php
<!-- index.php 典范结构 -->
<?php get_header(); ?>

<main id="main-content">
    <?php if (have_posts()) : ?>
        <?php while (have_posts()) : the_post(); ?>
            <article id="post-<?php the_ID(); ?>" <?php post_class(); ?>>
                <h2 class="entry-title">
                    <a href="<?php the_permalink(); ?>"><?php the_title(); ?></a>
                </h2>
                <div class="entry-meta">
                    <span><?php the_time('Y-m-d'); ?></span>
                    <span><?php the_author(); ?></span>
                    <span><?php comments_number('0 评论', '1 评论', '% 评论'); ?></span>
                </div>
                <div class="entry-content">
                    <?php the_excerpt(); ?>
                </div>
            </article>
        <?php endwhile; ?>

        <div class="pagination">
            <?php
            the_posts_pagination([
                'prev_text' => '← 上一页',
                'next_text' => '下一页 →',
            ]);
            ?>
        </div>
    <?php else : ?>
        <p>没有找到内容。</p>
    <?php endif; ?>
</main>

<?php get_sidebar(); ?>
<?php get_footer(); ?>
```

### 6.6 自定义查询与 WP_Query

```php
// 基本查询参数
$query = new WP_Query([
    'post_type'      => 'post',
    'posts_per_page' => 6,
    'orderby'        => 'date',
    'order'          => 'DESC',
]);

// 分类查询
$query = new WP_Query([
    'category_name' => 'php',  // 别名
    // 'cat'       => 5,       // 分类 ID
    'posts_per_page' => 10,
]);

// 标签查询
$query = new WP_Query([
    'tag'            => 'wordpress',
    'posts_per_page' => 5,
]);

// 元数据查询
$query = new WP_Query([
    'post_type'  => 'product',
    'meta_key'   => '_product_price',
    'meta_value' => 100,
    'meta_compare' => '>=',
    'meta_type'  => 'NUMERIC',
    'orderby'    => 'meta_value_num',
    'order'      => 'ASC',
]);

// 日期查询
$query = new WP_Query([
    'date_query' => [
        [
            'after'     => '2024-01-01',
            'before'    => '2024-12-31',
            'inclusive' => true,
        ],
    ],
]);

// 多条件元数据查询
$query = new WP_Query([
    'meta_query' => [
        'relation' => 'AND',
        [
            'key'   => '_product_price',
            'value' => 50,
            'compare' => '>=',
            'type'  => 'NUMERIC',
        ],
        [
            'key'   => '_product_stock',
            'value' => 0,
            'compare' => '>',
            'type'  => 'NUMERIC',
        ],
    ],
]);

// 分类法查询
$query = new WP_Query([
    'tax_query' => [
        'relation' => 'AND',
        [
            'taxonomy' => 'product_category',
            'field'    => 'slug',
            'terms'    => ['electronics', 'books'],
        ],
    ],
]);

// 使用查询
if ($query->have_posts()) {
    while ($query->have_posts()) {
        $query->the_post();
        // 输出...
    }
    wp_reset_postdata(); // 重要！重置全局 $post
}
```

### 6.7 自定义 Widget

```php
class Latest_Posts_Widget extends WP_Widget {

    public function __construct() {
        parent::__construct(
            'latest_posts_widget',
            '最新文章',
            ['description' => '显示最新的 N 篇文章']
        );
    }

    // 前端输出
    public function widget($args, $instance) {
        echo $args['before_widget'];

        if (!empty($instance['title'])) {
            echo $args['before_title'] . esc_html($instance['title']) . $args['after_title'];
        }

        $query = new WP_Query([
            'posts_per_page' => $instance['count'] ?? 5,
        ]);

        echo '<ul>';
        while ($query->have_posts()) {
            $query->the_post();
            printf('<li><a href="%s">%s</a></li>', get_permalink(), get_the_title());
        }
        echo '</ul>';
        wp_reset_postdata();

        echo $args['after_widget'];
    }

    // 后台表单
    public function form($instance) {
        $title = $instance['title'] ?? '最新文章';
        $count = $instance['count'] ?? 5;
        ?>
        <p>
            <label for="<?php echo $this->get_field_id('title'); ?>">标题：</label>
            <input class="widefat" id="<?php echo $this->get_field_id('title'); ?>"
                   name="<?php echo $this->get_field_name('title'); ?>"
                   type="text" value="<?php echo esc_attr($title); ?>">
        </p>
        <p>
            <label for="<?php echo $this->get_field_id('count'); ?>">显示数量：</label>
            <input id="<?php echo $this->get_field_id('count'); ?>"
                   name="<?php echo $this->get_field_name('count'); ?>"
                   type="number" value="<?php echo esc_attr($count); ?>" min="1" max="20">
        </p>
        <?php
    }

    // 保存
    public function update($new_instance, $old_instance) {
        return [
            'title' => sanitize_text_field($new_instance['title']),
            'count' => absint($new_instance['count']),
        ];
    }
}

// 注册 Widget
add_action('widgets_init', function() {
    register_widget('Latest_Posts_Widget');
});
```

### 6.8 子主题 (Child Theme)

```css
/* child-theme/style.css */
/*
Theme Name:   My Child Theme
Template:     my-theme
Version:      1.0.0
*/
```

```php
<?php
// child-theme/functions.php
// 子主题的 functions.php 会在父主题之前加载

// 覆盖父主题函数
function my_child_theme_setup() {
    // 你自己的设置
}
add_action('after_setup_theme', 'my_child_theme_setup', 11); // 优先级 > 10

// 正确加载父主题样式
add_action('wp_enqueue_scripts', function() {
    wp_enqueue_style(
        'parent-style',
        get_template_directory_uri() . '/style.css'
    );
    wp_enqueue_style(
        'child-style',
        get_stylesheet_directory_uri() . '/style.css',
        ['parent-style']
    );
});
```

---

## 7. REST API

### 7.1 基础请求

```bash
# 获取文章列表
GET /wp-json/wp/v2/posts

# 获取单篇文章
GET /wp-json/wp/v2/posts/1

# 分页
GET /wp-json/wp/v2/posts?per_page=10&page=1

# 排序
GET /wp-json/wp/v2/posts?orderby=date&order=desc

# 搜索
GET /wp-json/wp/v2/posts?search=keyword

# 按分类过滤
GET /wp-json/wp/v2/posts?categories=5

# 嵌入关联数据
GET /wp-json/wp/v2/posts?_embed=author,wp:featuredmedia
```

### 7.2 自定义 REST API 端点

```php
add_action('rest_api_init', function() {
    // 注册自定义路由
    register_rest_route('my-plugin/v1', '/latest-posts', [
        'methods'             => 'GET',
        'callback'            => 'get_latest_posts_api',
        'permission_callback' => '__return_true', // 公开端点
        'args'                => [
            'count' => [
                'default'           => 10,
                'sanitize_callback' => 'absint',
            ],
            'category' => [
                'sanitize_callback' => 'sanitize_text_field',
            ],
        ],
    ]);

    // 需要认证的端点
    register_rest_route('my-plugin/v1', '/settings', [
        'methods'             => 'POST',
        'callback'            => 'update_plugin_settings_api',
        'permission_callback' => function() {
            return current_user_can('manage_options');
        },
    ]);
});

// 回调函数
function get_latest_posts_api($request) {
    $count    = $request->get_param('count');
    $category = $request->get_param('category');

    $args = [
        'posts_per_page' => $count,
        'post_status'    => 'publish',
    ];

    if ($category) {
        $args['category_name'] = $category;
    }

    $query = new WP_Query($args);
    $posts = [];

    while ($query->have_posts()) {
        $query->the_post();
        $posts[] = [
            'id'       => get_the_ID(),
            'title'    => get_the_title(),
            'excerpt'  => get_the_excerpt(),
            'url'      => get_permalink(),
            'date'     => get_the_date('c'),
            'author'   => get_the_author(),
            'thumbnail'=> get_the_post_thumbnail_url(null, 'medium'),
        ];
    }
    wp_reset_postdata();

    return new WP_REST_Response([
        'success' => true,
        'data'    => $posts,
        'total'   => $query->found_posts,
    ], 200);
}
```

### 7.3 修改已有端点返回值

```php
// 在文章 REST 响应中追加自定义字段
add_action('rest_api_init', function() {
    register_rest_field('post', 'custom_fields', [
        'get_callback' => function($post) {
            return [
                'views'   => get_post_meta($post['id'], '_post_views', true),
                'likes'   => get_post_meta($post['id'], '_post_likes', true),
            ];
        },
    ]);
});
```

---

## 8. WP-CLI 命令行工具

### 8.1 常用命令速查

```bash
# === 核心管理 ===
wp core download                          # 下载 WordPress
wp core install --url=example.com --title="My Site" --admin_user=admin --admin_password=pass --admin_email=admin@example.com
wp core update                            # 更新核心
wp core version                           # 查看版本

# === 插件 ===
wp plugin list                            # 列出插件
wp plugin install akismet --activate      # 安装并激活
wp plugin activate my-plugin
wp plugin deactivate my-plugin
wp plugin delete my-plugin
wp plugin update --all                    # 更新所有插件

# === 主题 ===
wp theme list
wp theme install twentytwentyfour --activate
wp theme activate my-theme

# === 用户 ===
wp user list
wp user create editor-user editor@example.com --role=editor
wp user update 2 --user_pass=new_password
wp user delete 2 --reassign=1

# === 文章 ===
wp post list --post_type=post --posts_per_page=5
wp post create --post_title="Hello" --post_content="Content" --post_status=publish
wp post update 1 --post_title="Updated Title"
wp post delete 1 --force

# === 选项 ===
wp option get blogname
wp option update blogname "New Site Name"
wp option get siteurl

# === 数据库 ===
wp db export backup.sql                   # 导出数据库
wp db import backup.sql                   # 导入数据库
wp db optimize                            # 优化数据库
wp db repair                              # 修复数据库

# === 重写规则 ===
wp rewrite flush                          # 刷新 URL 重写规则
wp rewrite list                           # 查看重写规则

# === 缓存 ===
wp cache flush                            # 刷新对象缓存

# === 搜索替换（迁移用）===
wp search-replace 'http://old.com' 'https://new.com' --dry-run  # 预览
wp search-replace 'http://old.com' 'https://new.com'             # 执行

# === 维护模式 ===
wp maintenance-mode activate
wp maintenance-mode deactivate

# === 定时任务 ===
wp cron event list
wp cron event run wp_scheduled_delete
wp cron event delete wp_scheduled_delete
```

### 8.2 自定义 WP-CLI 命令

```php
// 在插件/主题中注册
if (defined('WP_CLI') && WP_CLI) {
    WP_CLI::add_command('my-custom-command', function($args, $assoc_args) {
        $count = $assoc_args['count'] ?? 10;
        $posts = get_posts(['numberposts' => $count]);

        WP_CLI::success(sprintf('Found %d posts', count($posts)));

        foreach ($posts as $post) {
            WP_CLI::line("- {$post->post_title}");
        }
    }, [
        'shortdesc' => '自定义命令描述',
        'synopsis'  => [
            [
                'type'     => 'assoc',
                'name'     => 'count',
                'optional' => true,
                'default'  => 10,
            ],
        ],
    ]);
}
// 使用：wp my-custom-command --count=5
```

---

## 9. 性能优化与缓存

### 9.1 Transients API（临时缓存）

```php
// 缓存查询结果 1 小时
$posts = get_transient('homepage_featured_posts');
if ($posts === false) {
    $posts = new WP_Query([
        'posts_per_page' => 5,
        'meta_key'       => '_is_featured',
        'meta_value'     => '1',
    ]);
    set_transient('homepage_featured_posts', $posts, HOUR_IN_SECONDS);
}

// 站点级缓存（多站点）
// set_site_transient() / get_site_transient()

// 删除缓存
delete_transient('homepage_featured_posts');
```

### 9.2 对象缓存 (Object Cache)

```php
// WordPress 内置缓存 API（支持 Redis/Memcached 持久化插件）
$data = wp_cache_get('my_key', 'my_group');
if ($data === false) {
    $data = expensive_query();
    wp_cache_set('my_key', $data, 'my_group', 300); // 缓存 5 分钟
}

// 删除
wp_cache_delete('my_key', 'my_group');
wp_cache_flush(); // 清除所有缓存
```

### 9.3 查询优化

```php
// ❌ 不要每页查两次
$posts_to_show = new WP_Query(['posts_per_page' => 10]);

// ✅ 使用 'no_found_rows' 优化不需要分页的查询
$posts_to_show = new WP_Query([
    'posts_per_page' => 10,
    'no_found_rows'  => true,        // 跳过 SQL_CALC_FOUND_ROWS
    'update_post_term_cache' => false, // 不需要分类信息时
    'update_post_meta_cache' => false, // 不需要元数据时
]);

// ✅ 只查需要的字段
$posts_to_show = new WP_Query([
    'fields' => 'ids',  // 只返回 ID 数组，极大减少内存
]);
```

### 9.4 数据库优化 SQL

```sql
-- 清理文章修订版本
DELETE FROM wp_posts WHERE post_type = 'revision';

-- 清理垃圾评论
DELETE FROM wp_comments WHERE comment_approved = 'spam';

-- 清理孤立元数据
DELETE pm FROM wp_postmeta pm
LEFT JOIN wp_posts wp ON wp.ID = pm.post_id
WHERE wp.ID IS NULL;

-- 优化表
OPTIMIZE TABLE wp_posts, wp_postmeta, wp_comments, wp_options;
```

### 9.5 懒加载与分页

```php
// 图片懒加载（WordPress 5.5+ 默认）
// 所有图片自动添加 loading="lazy"

// AJAX 无限滚动
add_action('wp_ajax_infinite_scroll', 'handle_infinite_scroll');
add_action('wp_ajax_nopriv_infinite_scroll', 'handle_infinite_scroll');

function handle_infinite_scroll() {
    $page = intval($_POST['page']);
    $query = new WP_Query([
        'posts_per_page' => 6,
        'paged'          => $page,
    ]);

    if ($query->have_posts()) {
        while ($query->have_posts()) {
            $query->the_post();
            get_template_part('template-parts/content');
        }
    }
    wp_reset_postdata();
    wp_die();
}
```

---

## 10. 安全性加固

### 10.1 数据验证与清理

```php
// === 输入验证 ===
$email = sanitize_email($_POST['email']);
$text  = sanitize_text_field($_POST['name']);      // 纯文本
$html  = wp_kses_post($_POST['content']);           // 允许安全的 HTML
$url   = esc_url_raw($_POST['url']);                // URL 清理
$int   = intval($_POST['id']);                      // 整数
$key   = sanitize_key($_POST['key']);               // 仅保留字母数字下划线

// === 输出转义 ===
echo esc_html($title);                // HTML 文本
echo esc_url($link);                  // URL
echo esc_attr($attr);                 // HTML 属性
echo esc_js($js);                     // JavaScript
echo wp_kses($html, ['a' => ['href']]); // 白名单 HTML 标签

// === Nonce 验证 ===
// 创建
$nonce = wp_create_nonce('my_action');
// 输出隐藏字段
wp_nonce_field('my_action', 'my_nonce_field');
// 验证
if (!wp_verify_nonce($_POST['my_nonce_field'], 'my_action')) {
    wp_die('安全验证失败');
}
// AJAX 中验证
check_ajax_referer('my_action', 'nonce');

// === 权限检查 ===
if (!current_user_can('edit_posts')) {
    wp_die('无权执行此操作');
}
```

### 10.2 SQL 注入防护

```php
// ❌ 危险——永远不要拼接 SQL
$wpdb->query("SELECT * FROM {$wpdb->posts} WHERE post_title = '{$_GET['title']}'");

// ✅ 使用 prepare
$query = $wpdb->prepare(
    "SELECT * FROM {$wpdb->posts} WHERE post_title = %s",
    $_GET['title']
);
$results = $wpdb->get_results($query);

// ✅ 使用占位符
$wpdb->prepare("SELECT * FROM {$wpdb->posts} WHERE ID = %d", $id);        // %d 整数
$wpdb->prepare("SELECT * FROM {$wpdb->posts} WHERE post_title = %s", $s); // %s 字符串
$wpdb->prepare("SELECT * FROM {$wpdb->posts} WHERE post_title LIKE %s", '%' . $wpdb->esc_like($s) . '%'); // LIKE
```

### 10.3 常见安全加固措施

```php
// wp-config.php 加固
define('DISALLOW_FILE_EDIT', true);        // 禁用后台文件编辑
define('FORCE_SSL_ADMIN', true);           // 强制后台 HTTPS
define('DISALLOW_FILE_MODS', true);        // 禁用插件/主题安装更新（生产环境）

// 隐藏 WordPress 版本号
remove_action('wp_head', 'wp_generator');
add_filter('the_generator', '__return_empty_string');

// 限制登录尝试（需要插件或自定义代码）
add_filter('authenticate', function($user, $username) {
    if ($username) {
        $attempts = get_transient('login_attempts_' . $username) ?: 0;
        if ($attempts >= 5) {
            return new WP_Error('too_many_attempts', '登录尝试次数过多，请稍后再试。');
        }
    }
    return $user;
}, 30, 2);

add_action('wp_login_failed', function($username) {
    $attempts = get_transient('login_attempts_' . $username) ?: 0;
    set_transient('login_attempts_' . $username, $attempts + 1, 15 * MINUTE_IN_SECONDS);
});

// 禁用 XML-RPC（如果不需要）
add_filter('xmlrpc_enabled', '__return_false');

// 移除 REST API 用户端点（防止用户枚举）
add_filter('rest_endpoints', function($endpoints) {
    if (isset($endpoints['/wp/v2/users'])) {
        unset($endpoints['/wp/v2/users']);
    }
    if (isset($endpoints['/wp/v2/users/(?P<id>[\\d]+)'])) {
        unset($endpoints['/wp/v2/users/(?P<id>[\\d]+)']);
    }
    return $endpoints;
});

// 设置安全的 HTTP 头
add_action('send_headers', function() {
    header('X-Content-Type-Options: nosniff');
    header('X-Frame-Options: SAMEORIGIN');
    header('X-XSS-Protection: 1; mode=block');
    header('Referrer-Policy: strict-origin-when-cross-origin');
});
```

---

## 11. 多站点 (Multisite)

### 11.1 启用多站点

```php
// wp-config.php 中添加
define('WP_ALLOW_MULTISITE', true);
```

然后在后台 → 工具 → 网络设置中配置。WordPress 会生成以下配置代码：

```php
// wp-config.php
define('MULTISITE', true);
define('SUBDOMAIN_INSTALL', false); // false = 子目录, true = 子域名
define('DOMAIN_CURRENT_SITE', 'example.com');
define('PATH_CURRENT_SITE', '/');
define('SITE_ID_CURRENT_SITE', 1);
define('BLOG_ID_CURRENT_SITE', 1);
```

### 11.2 多站点常用函数

```php
// 获取当前站点 ID
$blog_id = get_current_blog_id();

// 切换站点
switch_to_blog(2);
// 在站点 2 的上下文中操作
restore_current_blog();

// 获取所有站点
$sites = get_sites(['number' => 100]);

// 创建新站点
$site_id = wpmu_create_blog('new-site.example.com', '/', 'New Site', 1);

// 为所有站点激活插件
// 将插件放入 wp-content/mu-plugins/ 目录

// 网络级选项
$option = get_site_option('site_option_name');
update_site_option('site_option_name', $value);
```

---

## 12. 完整实战项目

### 12.1 自定义「站点访问统计」插件

完整的插件实例——记录页面访问量并在后台显示统计：

```php
<?php
/**
 * Plugin Name: Site Stats Tracker
 * Description: 记录网站页面访问量并显示统计
 * Version:     1.0.0
 * Author:      khz
 * Text Domain: site-stats
 */

defined('ABSPATH') || exit;

class Site_Stats_Tracker {

    private static $instance = null;

    public static function get_instance() {
        if (null === self::$instance) {
            self::$instance = new self();
        }
        return self::$instance;
    }

    private function __construct() {
        add_action('wp_head', [$this, 'track_page_view']);
        add_action('admin_menu', [$this, 'add_admin_menu']);
        add_action('admin_enqueue_scripts', [$this, 'enqueue_chart_js']);
        add_action('rest_api_init', [$this, 'register_api_routes']);
    }

    // === 记录页面访问 ===
    public function track_page_view() {
        if (is_admin() || is_user_logged_in()) return;

        global $post;
        $post_id = $post->ID ?? 0;

        $today = date('Y-m-d');
        $key = "stats_{$today}";

        $daily_stats = get_option($key, []);
        $daily_stats[$post_id] = ($daily_stats[$post_id] ?? 0) + 1;
        update_option($key, $daily_stats);

        // 文章总访问量
        $total_views = (int) get_post_meta($post_id, '_total_views', true);
        update_post_meta($post_id, '_total_views', $total_views + 1);
    }

    // === 后台菜单 ===
    public function add_admin_menu() {
        add_menu_page(
            '站点统计',
            '站点统计',
            'manage_options',
            'site-stats',
            [$this, 'render_admin_page'],
            'dashicons-chart-area',
            25
        );
    }

    // === 加载 Chart.js ===
    public function enqueue_chart_js($hook) {
        if ($hook !== 'toplevel_page_site-stats') return;
        wp_enqueue_script(
            'chartjs',
            'https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js',
            [],
            '4.4.0',
            true
        );
    }

    // === 后台页面 ===
    public function render_admin_page() {
        $stats = $this->get_weekly_stats();
        ?>
        <div class="wrap">
            <h1>站点访问统计</h1>

            <div class="stats-cards" style="display:flex; gap:20px; margin:20px 0;">
                <div class="card" style="background:#fff; padding:20px; border:1px solid #ccd0d4; flex:1;">
                    <h3>今日访问</h3>
                    <p style="font-size:32px; font-weight:bold; color:#2271b1;">
                        <?php echo $stats['today']; ?>
                    </p>
                </div>
                <div class="card" style="background:#fff; padding:20px; border:1px solid #ccd0d4; flex:1;">
                    <h3>本周访问</h3>
                    <p style="font-size:32px; font-weight:bold; color:#2271b1;">
                        <?php echo $stats['week']; ?>
                    </p>
                </div>
                <div class="card" style="background:#fff; padding:20px; border:1px solid #ccd0d4; flex:1;">
                    <h3>热门文章</h3>
                    <p style="font-size:16px;">
                        <?php echo esc_html($stats['top_post']); ?>
                    </p>
                </div>
            </div>

            <div class="chart-container" style="background:#fff; padding:20px; border:1px solid #ccd0d4;">
                <h2>最近 7 天趋势</h2>
                <canvas id="statsChart" width="400" height="200"></canvas>
            </div>
        </div>

        <script>
        const ctx = document.getElementById('statsChart').getContext('2d');
        new Chart(ctx, {
            type: 'line',
            data: {
                labels: <?php echo json_encode($stats['labels']); ?>,
                datasets: [{
                    label: '日访问量',
                    data: <?php echo json_encode($stats['values']); ?>,
                    borderColor: '#2271b1',
                    backgroundColor: 'rgba(34, 113, 177, 0.1)',
                    fill: true,
                }]
            },
            options: {
                responsive: true,
                plugins: {
                    legend: { display: false }
                }
            }
        });
        </script>
        <?php
    }

    // === 获取统计数据 ===
    private function get_weekly_stats() {
        $today      = 0;
        $week       = 0;
        $labels     = [];
        $values     = [];
        $top_post   = '';
        $max_views  = 0;

        for ($i = 6; $i >= 0; $i--) {
            $date  = date('Y-m-d', strtotime("-{$i} days"));
            $key   = "stats_{$date}";
            $data  = get_option($key, []);
            $total = array_sum($data);

            $labels[] = date('m/d', strtotime($date));
            $values[] = $total;
            $week    += $total;

            if ($i === 0) $today = $total;

            // 找当天热门文章
            if ($total > $max_views) {
                $max_views = $total;
                if (!empty($data)) {
                    $post_id  = array_key_first($data);
                    $top_post = get_the_title($post_id) ?: '—';
                }
            }
        }

        return compact('today', 'week', 'labels', 'values', 'top_post');
    }

    // === REST API ===
    public function register_api_routes() {
        register_rest_route('site-stats/v1', '/today', [
            'methods'             => 'GET',
            'callback'            => [$this, 'api_get_today_stats'],
            'permission_callback' => function() {
                return current_user_can('manage_options');
            },
        ]);
    }

    public function api_get_today_stats() {
        $data = get_option('stats_' . date('Y-m-d'), []);
        return new WP_REST_Response([
            'total'  => array_sum($data),
            'by_post' => $data,
        ]);
    }
}

// 初始化
Site_Stats_Tracker::get_instance();
```

---

## 附录：常用函数速查

### 获取内容

| 函数 | 说明 |
|------|------|
| `get_the_title($id)` | 获取文章标题 |
| `get_the_content()` | 获取文章内容 |
| `get_the_excerpt()` | 获取摘要 |
| `get_permalink($id)` | 获取文章链接 |
| `get_the_post_thumbnail_url($id, $size)` | 获取特色图片 URL |
| `get_the_terms($id, $taxonomy)` | 获取文章分类/标签 |
| `get_post_meta($id, $key, $single)` | 获取自定义字段 |
| `get_option($name)` | 获取选项值 |
| `get_theme_mod($name)` | 获取主题自定义设置 |
| `wp_get_attachment_url($id)` | 获取附件 URL |

### 判断函数

| 函数 | 说明 |
|------|------|
| `is_single()` | 是单篇文章 |
| `is_page()` | 是静态页面 |
| `is_home()` | 是博客首页 |
| `is_front_page()` | 是网站首页 |
| `is_archive()` | 是归档页 |
| `is_category()` | 是分类页 |
| `is_tag()` | 是标签页 |
| `is_search()` | 是搜索页 |
| `is_404()` | 是 404 页面 |
| `is_admin()` | 是后台 |
| `is_user_logged_in()` | 用户已登录 |

---

> **文档版本**: 1.0  
> **更新日期**: 2026-05-14  
> **适用版本**: WordPress 5.8+ / PHP 7.4+
