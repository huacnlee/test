# Rails 国际化 API

Rails（Rails 2.2 及以上版本）自带的 Ruby I18n（internationalization 的简写）gem，提供了易用、可扩展的框架，用于把应用翻译成英语之外的语言，或为应用提供多语言支持。

“国际化”（internationalization）过程通常是指，把所有字符串及本地化相关信息（例如日期或货币格式）从应用中抽取出来。“本地化”（localization）过程通常是指，翻译这些字符串并提供相关信息的本地格式。

因此，在国际化 Rails 应用的过程中，我们需要：

*   确保 Rails 提供了 I18n 支持；
*   把区域设置字典（locale dictionary）的位置告诉 Rails；
*   告诉 Rails 如何设置、保存和切换区域（locale）。

在本地化 Rails 应用的过程中，我们可能需要完成下面三项工作：

*   替换或补充 Rails 的默认区域设置，例如日期和时间格式、月份名称、Active Record 模型名等；
*   从应用中抽取字符串，并放入字典，例如视图中的闪现信息（flash message）、静态文本等；
*   把生成的字典储存在某个地方。

本文介绍 Rails I18n API，并提供国际化 Rails 应用的入门教程。

读完本文后，您将学到：

*   Rails 中 I18n 的工作原理；
*   在 REST 式应用中正确使用 I18n 的几种方式；
*   如何使用 I18n 翻译 Active Record 错误或 Action Mailer 电子邮件主题；
*   用于进一步翻译应用的其他工具。

-----------------------------------------------------------------------------

NOTE: Ruby I18n 框架提供了 Rails 应用国际化/本地化所需的全部必要支持。我们还可以使用各种 gem 来添加附加功能或特性。更多介绍请参阅 [rails-18n gem](https://github.com/svenfuchs/rails-i18n)。

<a class="anchor" id="how-i18n-in-ruby-on-rails-works"></a>

## Rails 中 I18n 的工作原理

国际化是一个复杂的问题。自然语言在很多方面（例如复数规则）有所不同，要想一次性提供解决所有问题的工具很难。因此，Rails I18n API 专注于：

*   支持英语及类似语言
*   易于定制和扩展，以支持其他语言

作为这个解决方案的一部分，Rails 框架中的每个静态字符串（例如，Active Record 数据验证信息、时间和日期格式）都已国际化。Rails 应用的本地化意味着把这些静态字符串翻译为所需语言。

<a class="anchor" id="the-overall-architecture-of-the-library"></a>

### I18n 库的总体架构

因此，Ruby I18n gem 分为两部分：

*   I18n 框架的公开 API——包含公开方法的 Ruby 模块，定义 I18n 库的工作方式
*   实现这些方法的默认后端（称为简单后端）

作为用户，我们应该始终只访问 I18n 模块的公开方法，但了解后端的功能也很有帮助。

NOTE: 我们可以把默认的简单后端替换为其他功能更强的后端，这时翻译数据可能会储存在关系数据库、GetText 字典或类似解决方案中。更多介绍请参阅 [使用不同的后端](#using-different-backends)。

<a class="anchor" id="the-public-i18n-api"></a>

### I18n 公开 API

I18n API 中最重要的两个方法是：

```ruby
translate # 查找文本翻译
localize  # 把日期和时间对象转换为本地格式（本地化）
```

这两个方法的别名分别为 `#t` 和 `#l`，用法如下：

```ruby
I18n.t 'store.title'
I18n.l Time.now
```

对于下列属性，I18n API 还提供了属性读值方法和设值方法：

```ruby
load_path                 # 自定义翻译文件的路径
locale                    # 获取或设置当前区域
default_locale            # 获取或设置默认区域
available_locales         # 应用可用的区域设置白名单
enforce_available_locales # 强制使用白名单（true 或 false）
exception_handler         # 使用其他异常处理程序
backend                   # 使用其他后端
```

现在，我们已经掌握了 Rails I18n API 的基本用法，从下一节开始，我们将从头开始国际化一个简单的 Rails 应用。

<a class="anchor" id="setup-the-rails-application-for-internationalization"></a>

## Rails 应用的国际化设置

本节介绍为 Rails 应用提供 I18n 支持的几个步骤。

<a class="anchor" id="configure-the-i18n-module"></a>

### 配置 I18n 模块

根据“多约定，少配置”原则，Rails I18n 库提供了默认翻译字符串。如果需要不同的翻译字符串，可以直接覆盖默认值。

Rails 会把 `config/locales` 文件夹中的 `.rb` 和 `.yml` 文件自动添加到翻译文件加载路径中。

这个文件夹中的 `en.yml` 区域设置文件包含了一个翻译字符串示例：

```yml
en:
  hello: "Hello world"
```

上面的代码表示，在 `:en` 区域设置中，键 `hello` 会映射到 `Hello world` 字符串上。在 Rails 中，字符串都以这种方式进行国际化，例如，Active Model 的数据验证信息位于 [activemodel/lib/active_model/locale/en.yml](https://github.com/rails/rails/blob/master/activemodel/lib/active_model/locale/en.yml) 文件中，时间和日期格式位于 [activesupport/lib/active_support/locale/en.yml](https://github.com/rails/rails/blob/master/activesupport/lib/active_support/locale/en.yml) 文件中。我们可以使用 YAML 或标准 Ruby 散列，把翻译信息储存在默认的简单后端中。

I18n 库使用英语作为默认的区域设置，例如，如果未设置为其他区域，那就使用 `:en` 区域来查找翻译。

NOTE: 经过[讨论](http://groups.google.com/group/rails-i18n/browse_thread/thread/14dede2c7dbe9470/80eec34395f64f3c?hl=en)，I18n 库在选取区域设置的键时最终采取了务实的方式，也就是仅包含语言部分，例如 `:en`、`:pl`，而不是传统上使用的语言和区域两部分，例如 `:en-US` 、 `:en-GB`。很多国际化的应用都是这样做的，例如把 `:cs`、`:th` 和 `:es` 分别用于捷克语、泰语和西班牙语。尽管如此，在同一语系中也可能存在重要的区域差异，例如，`:en-US` 使用 `$` 作为货币符号，而 `:en-GB` 使用 `£` 作为货币符号。因此，如果需要，我们也可以使用传统方式，例如，在 `:en-GB` 字典中提供完整的 `"English - United Kingdom"` 区域。像 [Globalize3](https://github.com/globalize/globalize) 这样的 gem 可以实现这一功能。

Rails 会自动加载翻译文件加载路径（`I18n.load_path`），这是一个保存有翻译文件路径的数组。通过配置翻译文件加载路径，我们可以自定义翻译文件的目录结构和文件命名规则。

NOTE: I18n 库的后端采用了延迟加载技术，相关翻译信息仅在第一次查找时加载。我们可以根据需要，随时替换默认后端。

默认的区域设置和翻译的加载路径可以在 `config/application.rb` 文件中配置，如下所示：

```ruby
config.i18n.load_path += Dir[Rails.root.join('my', 'locales', '*.{rb,yml}').to_s]
config.i18n.default_locale = :de
```

在查找翻译文件之前，必须先指定翻译文件加载路径。应该通过初始化脚本修改默认区域设置，而不是 `config/application.rb` 文件：

```ruby
# config/initializers/locale.rb

# 指定 I18n 库搜索翻译文件的路径
I18n.load_path += Dir[Rails.root.join('lib', 'locale', '*.{rb,yml}')]

# 应用可用的区域设置白名单
I18n.available_locales = [:en, :pt]

# 修改默认区域设置（默认是 :en）
I18n.default_locale = :pt
```

<a class="anchor" id="managing-the-locale-across-requests"></a>

### 跨请求管理区域设置

除非显式设置了 `I18n.locale`，默认区域设置将会应用于所有翻译文件。

本地化应用有时需要支持多区域设置。此时，需要在每个请求之前设置区域，这样在请求的整个生命周期中，都会根据指定区域，对所有字符串进行翻译。

我们可以在 `ApplicationController` 中使用 `before_action` 方法设置区域：

```ruby
before_action :set_locale

def set_locale
  I18n.locale = params[:locale] || I18n.default_locale
end
```

上面的例子说明了如何使用 URL 查询参数来设置区域。例如，对于 http://example.com/books?locale=pt 会使用葡萄牙语进行本地化，对于 http://localhost:3000?locale=de 会使用德语进行本地化。

接下来介绍区域设置的几种不同方式。

<a class="anchor" id="setting-the-locale-from-the-domain-name"></a>

#### 根据域名设置区域

第一种方式是，根据应用的域名设置区域。例如，通过 `www.example.com` 加载英语（或默认）区域设置，通过 `www.example.es` 加载西班牙语区域设置。也就是根据顶级域名设置区域。这种方式有下列优点：

*   区域设置成为 URL 地址显而易见的一部分
*   用户可以直观地判断出页面所使用的语言
*   在 Rails 中非常容易实现
*   搜索引擎偏爱这种把不同语言内容放在不同域名上的做法

在 `ApplicationController` 中，我们可以进行如下配置：

```ruby
before_action :set_locale

def set_locale
  I18n.locale = extract_locale_from_tld || I18n.default_locale
end

# 从顶级域名中获取区域设置，如果获取失败会返回 nil
# 需要在 /etc/hosts 文件中添加如下设置：
#   127.0.0.1 application.com
#   127.0.0.1 application.it
#   127.0.0.1 application.pl
def extract_locale_from_tld
  parsed_locale = request.host.split('.').last
  I18n.available_locales.map(&:to_s).include?(parsed_locale) ? parsed_locale : nil
end
```

我们还可以通过类似方式，根据子域名设置区域：

```ruby
# 从子域名中获取区域设置（例如 http://it.application.local:3000）
# 需要在 /etc/hosts 文件中添加如下设置：
#   127.0.0.1 gr.application.local
def extract_locale_from_subdomain
  parsed_locale = request.subdomains.first
  I18n.available_locales.map(&:to_s).include?(parsed_locale) ? parsed_locale : nil
end
```

要想为应用添加区域设置切换菜单，可以使用如下代码：

```ruby
link_to("Deutsch", "#{APP_CONFIG[:deutsch_website_url]}#{request.env['PATH_INFO']}")
```

其中 `APP_CONFIG[:deutsch_website_url]` 的值类似 `http://www.application.de`。

尽管这个解决方案具有上面提到的各种优点，但通过不同域名来提供不同的本地化版本（“语言版本”）有时并非我们的首选。在其他各种可选方案中，在 URL 参数（或请求路径）中包含区域设置是最常见的。

<a class="anchor" id="setting-the-locale-from-url-params"></a>

#### 根据 URL 参数设置区域

区域设置（和传递）的最常见方式，是将其包含在 URL 参数中，例如，在前文第一个示例中，`before_action` 方法调用中的 `I18n.locale = params[:locale]`。此时，我们会使用 `www.example.com/books?locale=ja` 或 `www.example.com/ja/books` 这样的网址。

和根据域名设置区域类似，这种方式具有不少优点，尤其是 REST 式的命名风格，顺应了当前的互联网潮流。不过采用这种方式所需的工作量要大一些。

从 URL 参数获取并设置区域并不难，只要把区域设置包含在 URL 中并通过请求传递即可。当然，没有人愿意在生成每个 URL 地址时显式添加区域设置，例如 `link_to(books_url(locale: I18n.locale))`。

Rails 的 `ApplicationController#default_url_options` 方法提供的“集中修改 URL 动态生成规则”的功能，正好可以解决这个问题：我们可以设置 `url_for` 及相关辅助方法的默认行为（通过覆盖 `default_url_options` 方法）。

我们可以在 `ApplicationController` 中添加下面的代码：

```ruby
# app/controllers/application_controller.rb
def default_url_options
  { locale: I18n.locale }
end
```

这样，所有依赖于 `url_for` 的辅助方法（例如，具名路由辅助方法 `root_path` 和 `root_url`，资源路由辅助方法 `books_path` 和 `books_url` 等等）都会自动在查询字符串中添加区域设置，例如：`http://localhost:3001/?locale=ja`。

至此，我们也许已经很满意了。但是，在应用的每个 URL 地址的末尾添加区域设置，会影响 URL 地址的可读性。此外，从架构的角度看，区域设置的层级应该高于 URL 地址中除域名之外的其他组成部分，这一点也应该通过 URL 地址自身体现出来。

要想使用 `http://www.example.com/en/books`（加载英语区域设置）和 `http://www.example.com/nl/books`（加载荷兰语区域设置）这样的 URL 地址，我们可以使用前文提到的覆盖 `default_url_options` 方法的方式，通过 `scope` 方法设置路由：

```ruby
# config/routes.rb
scope "/:locale" do
  resources :books
end
```

现在，当我们调用 `books_path` 方法时，就会得到 `"/en/books"`（对于默认区域设置）。像 `http://localhost:3001/nl/books` 这样的 URL 地址会加载荷兰语区域设置，之后调用 `books_path` 方法时会返回 `"/nl/books"`（因为区域设置发生了变化）。

WARNING: 由于 `default_url_options` 方法的返回值是根据请求分别缓存的，因此无法通过循环调用辅助方法来生成 URL 地址中的区域设置，
也就是说，无法在每次迭代中设置相应的 `I18n.locale`。正确的做法是，保持 `I18n.locale` 不变，向辅助方法显式传递 `:locale` 选项，或者编辑 `request.original_fullpath`。

如果不想在路由中强制使用区域设置，我们可以使用可选的路径作用域（用括号表示），就像下面这样：

```ruby
# config/routes.rb
scope "(:locale)", locale: /en|nl/ do
  resources :books
end
```

通过这种方式，访问不带区域设置的 `http://localhost:3001/books` URL 地址时就不会抛出 `Routing Error` 错误了。这样，我们就可以在不指定区域设置时，使用默认的区域设置。

当然，我们需要特别注意应用的根地址﹝通常是“主页（homepage）”或“仪表盘（dashboard）”﹞。像 `root to: "books#index"` 这样的不考虑区域设置的路由声明，会导致 `http://localhost:3001/nl` 无法正常访问。（尽管“只有一个根地址”看起来并没有错）

因此，我们可以像下面这样映射 URL 地址：

```ruby
# config/routes.rb
get '/:locale' => 'dashboard#index'
```

需要特别注意路由的声明顺序，以避免这条路由覆盖其他路由。（我们可以把这条路由添加到 `root :to` 路由声明之前）

NOTE: 有一些 gem 可以简化路由设置，如 [routing_filter](https://github.com/svenfuchs/routing-filter/tree/master)、[rails-translate-routes](https://github.com/francesc/rails-translate-routes) 和 [route_translator](https://github.com/enriclluelles/route_translator)。

<a class="anchor" id="setting-the-locale-from-user-preferences"></a>

#### 根据用户偏好设置进行区域设置

支持用户身份验证的应用，可能会允许用户在界面中选择区域偏好设置。通过这种方式，用户选择的区域偏好设置会储存在数据库中，并用于处理该用户发起的请求。

```ruby
def set_locale
  I18n.locale = current_user.try(:locale) || I18n.default_locale
end
```

<a class="anchor" id="choosing-an-implied-locale"></a>

#### 使用隐式区域设置

如果没有显式地为请求设置区域（例如，通过上面提到的各种方式），应用就会尝试推断出所需区域。

<a class="anchor" id="inferring-locale-from-the-language-header"></a>

##### 根据 HTTP 首部推断区域设置

`Accept-Language` HTTP 首部指明响应请求时使用的首选语言。浏览器[根据用户的语言偏好设置设定这个 HTTP 首部](http://www.w3.org/International/questions/qa-lang-priorities)，这是推断区域设置的首选方案。

下面是使用 `Accept-Language` HTTP 首部的一个简单实现：

```ruby
def set_locale
  logger.debug "* Accept-Language: #{request.env['HTTP_ACCEPT_LANGUAGE']}"
  I18n.locale = extract_locale_from_accept_language_header
  logger.debug "* Locale set to '#{I18n.locale}'"
end

private
  def extract_locale_from_accept_language_header
    request.env['HTTP_ACCEPT_LANGUAGE'].scan(/^[a-z]{2}/).first
  end
```

实际上，我们通常会使用更可靠的代码。Iain Hecker 开发的 [http_accept_language](https://github.com/iain/http_accept_language/tree/master) 或 Ryan Tomayko 开发的 [locale](https://github.com/rack/rack-contrib/blob/master/lib/rack/contrib/locale.rb) Rack 中间件就提供了更好的解决方案。

<a class="anchor" id="inferring-the-locale-from-ip-geolocation"></a>

##### 根据 IP 地理位置推断区域设置

我们可以通过客户端请求的 IP 地址来推断客户端所处的地理位置，进而推断其区域设置。[GeoIP Lite Country](http://www.maxmind.com/app/geolitecountry) 这样的服务或 [geocoder](https://github.com/alexreisner/geocoder) 这样的 gem 就可以实现这一功能。

一般来说，这种方式远不如使用 HTTP 首部可靠，因此并不适用于大多数 Web 应用。

<a class="anchor" id="storing-the-locale-from-the-session-or-cookies"></a>

#### 在会话或 Cookie 中储存区域设置

WARNING: 我们可能会认为，可以把区域设置储存在会话或 Cookie 中。但是，我们不能这样做。区域设置应该是透明的，并作为 URL 地址的一部分。这样，我们就不会打破用户的正常预期：如果我们发送一个 URL 地址给朋友，他们应该看到和我们一样的页面和内容。这就是所谓的 REST 规则。关于 REST 规则的更多介绍，请参阅 [Stefan Tilkov 写的系列文章](http://www.infoq.com/articles/rest-introduction)。后文将讨论这个规则的一些例外情况。

<a class="anchor" id="internationalization-and-localization"></a>

## 国际化和本地化

现在，我们已经完成了对 Rails 应用 I18n 支持的初始化，进行了区域设置，并在不同请求中应用了区域设置。

接下来，我们要通过抽象本地化相关元素，完成应用的国际化。最后，通过为这些抽象元素提供必要翻译，完成应用的本地化。

下面给出一个例子：

```ruby
# config/routes.rb
Rails.application.routes.draw do
  root to: "home#index"
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :set_locale

  def set_locale
    I18n.locale = params[:locale] || I18n.default_locale
  end
end
```

```ruby
# app/controllers/home_controller.rb
class HomeController < ApplicationController
  def index
    flash[:notice] = "Hello Flash"
  end
end
```

```erb
# app/views/home/index.html.erb
<h1>Hello World</h1>
<p><%= flash[:notice] %></p>
```

![demo untranslated](images/i18n/demo_untranslated.png)

<a class="anchor" id="abstracting-localized-code"></a>

### 抽象本地化代码

在我们的代码中有两个英文字符串（`"Hello Flash"` 和 `"Hello World"`），它们在响应用户请求时显示。为了国际化这部分代码，需要用 Rails 提供的 `#t` 辅助方法来代替这两个字符串，同时为每个字符串选择合适的键：

```ruby
# app/controllers/home_controller.rb
class HomeController < ApplicationController
  def index
    flash[:notice] = t(:hello_flash)
  end
end
```

```erb
# app/views/home/index.html.erb
<h1><%= t :hello_world %></h1>
<p><%= flash[:notice] %></p>
```

现在，Rails 在渲染 `index` 视图时会显示错误信息，告诉我们缺少 `:hello_world` 和 `:hello_flash` 这两个键的翻译。

![demo translation missing](images/i18n/demo_translation_missing.png)

NOTE: Rails 为视图添加了 `t`（`translate`）辅助方法，从而避免了反复使用 `I18n.t` 这么长的写法。此外，`t` 辅助方法还能捕获缺少翻译的错误，把生成的错误信息放在 `<span class="translation_missing">` 元素里。

<a class="anchor" id="providing-translations-for-internationalized-strings"></a>

### 为国际化字符串提供翻译

下面，我们把缺少的翻译添加到翻译字典文件中：

```yml
# config/locales/en.yml
en:
  hello_world: Hello world!
  hello_flash: Hello flash!

# config/locales/pirate.yml
pirate:
  hello_world: Ahoy World
  hello_flash: Ahoy Flash
```

因为我们没有修改 `default_locale`，翻译会使用 `:en` 区域设置，响应请求时生成的视图会显示英文字符串：

![demo translated en](images/i18n/demo_translated_en.png)

如果我们通过 URL 地址（`http://localhost:3000?locale=pirate`）把区域设置为 `pirate`，响应请求时生成的视图就会显示海盗黑话：

![demo translated pirate](images/i18n/demo_translated_pirate.png)

NOTE: 添加新的区域设置文件后，需要重启服务器。

要想把翻译储存在 SimpleStore 中，我们可以使用 YAML（`.yml`）或纯 Ruby（`.rb`）文件。大多数 Rails 开发者会优先选择 YAML。不过 YAML 有一个很大的缺点，它对空格和特殊字符非常敏感，因此有可能出现应用无法正确加载字典的情况。而 Ruby 文件如果有错误，在第一次加载时应用就会崩溃，因此我们很容易就能找出问题。（如果在使用 YAML 字典时遇到了“奇怪的问题”，可以尝试把字典的相关部分放入 Ruby 文件中。）

如果翻译存储在 YAML 文件中，有些键必须转义：

*   true, on, yes
*   false, off, no

例如：

```yml
# config/locales/en.yml
en:
  success:
    'true':  'True!'
    'on':    'On!'
    'false': 'False!'
  failure:
    true:    'True!'
    off:     'Off!'
    false:   'False!'
```

```ruby
I18n.t 'success.true' # => 'True!'
I18n.t 'success.on' # => 'On!'
I18n.t 'success.false' # => 'False!'
I18n.t 'failure.false' # => Translation Missing
I18n.t 'failure.off' # => Translation Missing
I18n.t 'failure.true' # => Translation Missing
```

<a class="anchor" id="passing-variables-to-translations"></a>

### 把变量传递给翻译

成功完成应用国际化的一个关键因素是，避免在抽象本地化代码时，对语法规则做出不正确的假设。某个区域设置的基本语法规则，在另一个区域设置中可能不成立。

下面给出一个不正确抽象的例子，其中对翻译的不同组成部分的排序进行了假设。注意，为了处理这个例子中出现的情况，Rails 提供了 `number_to_currency` 辅助方法。

```erb
# app/views/products/show.html.erb
<%= "#{t('currency')}#{@product.price}" %>
```

```yml
# config/locales/en.yml
en:
  currency: "$"

# config/locales/es.yml
es:
  currency: "€"
```

如果产品价格是 10，那么西班牙语的正确翻译是“10 €”而不是“€10”，但上面的抽象并不能正确处理这种情况。

为了创建正确的抽象，I18n gem 提供了变量插值（variable interpolation）功能，它允许我们在翻译定义（translation definition）中使用变量，并把这些变量的值传递给翻译方法。

下面给出一个正确抽象的例子：

```erb
# app/views/products/show.html.erb
<%= t('product_price', price: @product.price) %>
```

```yml
# config/locales/en.yml
en:
  product_price: "$%{price}"

# config/locales/es.yml
es:
  product_price: "%{price} €"
```

所有的语法和标点都由翻译定义自己决定，所以抽象可以给出正确的翻译。

NOTE: `default` 和 `scope` 是保留关键字，不能用作变量名。如果误用，Rails 会抛出 `I18n::ReservedInterpolationKey` 异常。如果没有把翻译所需的插值变量传递给 `#translate` 方法，Rails 会抛出 `I18n::MissingInterpolationArgument` 异常。

<a class="anchor" id="adding-date-time-formats"></a>

### 添加日期/时间格式

现在，我们要给视图添加时间戳，以便演示日期/时间的本地化功能。要想本地化时间格式，可以把时间对象传递给 `I18n.l` 方法或者（最好）使用 `#l` 辅助方法。可以通过 `:format` 选项指定时间格式（默认情况下使用 `:default` 格式）。

```erb
# app/views/home/index.html.erb
<h1><%=t :hello_world %></h1>
<p><%= flash[:notice] %></p>
<p><%= l Time.now, format: :short %></p>
```

然后在 `pirate` 翻译文件中添加时间格式（Rails 默认使用的英文翻译文件已经包含了时间格式）：

```yml
# config/locales/pirate.yml
pirate:
  time:
    formats:
      short: "arrrround %H'ish"
```

得到的结果如下：

![demo localized pirate](images/i18n/demo_localized_pirate.png)

TIP: 现在，我们可能需要添加一些日期/时间格式，这样 I18n 后端才能按照预期工作（至少应该为 `pirate` 区域设置添加日期/时间格式）。当然，很可能已经有人通过翻译 Rails 相关区域设置的默认值，完成了这些工作。[GitHub 上的 rails-i18n 仓库](https://github.com/svenfuchs/rails-i18n/tree/master/rails/locale)提供了各种本地化文件的存档。把这些本地化文件放在 `config/locales/` 文件夹中即可正常使用。

<a class="anchor" id="inflection-rules-for-other-locales"></a>

### 其他区域的变形规则

Rails 允许我们为英语之外的区域定义变形规则（例如单复数转换规则）。在 `config/initializers/inflections.rb` 文件中，我们可以为多个区域定义规则。这个初始化脚本包含了为英语指定附加规则的例子，我们可以参考这些例子的格式为其他区域定义规则。

<a class="anchor" id="localized-views"></a>

### 本地化视图

假设应用中包含 `BooksController`，`index` 动作默认会渲染 `app/views/books/index.html.erb` 模板。如果我们在同一个文件夹中创建了包含本地化变量的 `index.es.html.erb` 模板，当区域设置为 `:es` 时，`index` 动作就会渲染这个模板，而当区域设置为默认区域时， `index` 动作会渲染通用的 `index.html.erb` 模板。（在 Rails 的未来版本中，本地化的这种自动化魔术，有可能被应用于 `public` 文件夹中的资源）

本地化视图功能很有用，例如，如果我们有大量静态内容，就可以使用本地化视图，从而避免把所有东西都放进 YAML 或 Ruby 字典里的麻烦。但要记住，一旦我们需要修改模板，就必须对每个模板文件逐一进行修改。

<a class="anchor" id="organization-of-locale-files"></a>

### 区域设置文件的组织

当我们使用 I18n 库自带的 SimpleStore 时，字典储存在磁盘上的纯文本文件中。对于每个区域，把应用的各部分翻译都放在一个文件中，可能会带来管理上的困难。因此，把每个区域的翻译放在多个文件中，分层进行管理是更好的选择。

例如，我们可以像下面这样组织 `config/locales` 文件夹：

```
|-defaults
|---es.rb
|---en.rb
|-models
|---book
|-----es.rb
|-----en.rb
|-views
|---defaults
|-----es.rb
|-----en.rb
|---books
|-----es.rb
|-----en.rb
|---users
|-----es.rb
|-----en.rb
|---navigation
|-----es.rb
|-----en.rb
```

这样，我们就可以把模型和属性名同视图中的文本分离，同时还能使用“默认值”（例如日期和时间格式）。I18n 库的不同后端可以提供不同的分离方式。

NOTE: Rails 默认的区域设置加载机制，无法自动加载上面例子中位于嵌套文件夹中的区域设置文件。因此，我们还需要进行显式设置：

```ruby
# config/application.rb
config.i18n.load_path += Dir[Rails.root.join('config', 'locales', '**', '*.{rb,yml}')]
```


<a class="anchor" id="overview-of-the-i18n-api-features"></a>

## I18n API 功能概述

现在我们已经对 I18n 库有了较好的了解，知道了如何国际化简单的 Rails 应用。在下面几个小节中，我们将更深入地了解相关功能。

这几个小节将展示使用 `I18n.translate` 方法以及 `translate` 视图辅助方法的示例（注意视图辅助方法提供的附加功能）。

所涉及的功能如下：

*   查找翻译
*   把数据插入翻译中
*   复数的翻译
*   使用安全 HTML 翻译（只针对视图辅助方法）
*   本地化日期、数字、货币等

<a class="anchor" id="looking-up-translations"></a>

### 查找翻译

<a class="anchor" id="basic-lookup-scopes-and-nested-keys"></a>

#### 基本查找、作用域和嵌套键

Rails 通过键来查找翻译，其中键可以是符号或字符串。这两种键是等价的，例如：

```ruby
I18n.t :message
I18n.t 'message'
```

`translate` 方法接受 `:scope` 选项，选项的值可以包含一个或多个附加键，用于指定翻译键（translation key）的“命名空间”或作用域：

```ruby
I18n.t :record_invalid, scope: [:activerecord, :errors, :messages]
```

上述代码会在 Active Record 错误信息中查找 `:record_invalid` 信息。

此外，我们还可以用点号分隔的键来指定翻译键和作用域：

```ruby
I18n.translate "activerecord.errors.messages.record_invalid"
```

因此，下列调用是等效的：

```ruby
I18n.t 'activerecord.errors.messages.record_invalid'
I18n.t 'errors.messages.record_invalid', scope: :activerecord
I18n.t :record_invalid, scope: 'activerecord.errors.messages'
I18n.t :record_invalid, scope: [:activerecord, :errors, :messages]
```

<a class="anchor" id="defaults"></a>

#### 默认值

如果指定了 `:default` 选项，在缺少翻译的情况下，就会返回该选项的值：

```ruby
I18n.t :missing, default: 'Not here'
# => 'Not here'
```

如果 `:default` 选项的值是符号，这个值会被当作键并被翻译。我们可以为 `:default` 选项指定多个值，第一个被成功翻译的键或遇到的字符串将被作为返回值。

例如，下面的代码首先尝试翻译 `:missing` 键，然后是 `:also_missing` 键。由于两次翻译都不能得到结果，最后会返回 `"Not here"` 字符串。

```ruby
I18n.t :missing, default: [:also_missing, 'Not here']
# => 'Not here'
```

<a class="anchor" id="bulk-and-namespace-lookup"></a>

#### 批量查找和命名空间查找

要想一次查找多个翻译，我们可以传递键的数组作为参数：

```ruby
I18n.t [:odd, :even], scope: 'errors.messages'
# => ["must be odd", "must be even"]
```

此外，键可以转换为一组翻译的（可能是嵌套的）散列。例如，下面的代码可以生成所有 Active Record 错误信息的散列：

```ruby
I18n.t 'activerecord.errors.messages'
# => {:inclusion=>"is not included in the list", :exclusion=> ... }
```

<a class="anchor" id="lazy-lookup"></a>

#### 惰性查找

Rails 实现了一种在视图中查找区域设置的便捷方法。如果有下述字典：

```yml
es:
  books:
    index:
      title: "Título"
```

我们就可以像下面这样在 `app/views/books/index.html.erb` 模板中查找 `books.index.title` 的值（注意点号）：

```erb
<%= t '.title' %>
```

NOTE: 只有 `translate` 视图辅助方法支持根据片段自动补全翻译作用域的功能。

我们还可以在控制器中使用惰性查找（lazy lookup）：

```yml
en:
  books:
    create:
      success: Book created!
```

用于设置闪现信息：

```ruby
class BooksController < ApplicationController
  def create
    # ...
    redirect_to books_url, notice: t('.success')
  end
end
```

<a class="anchor" id="pluralization"></a>

### 复数转换

在英语中，一个字符串只有一种单数形式和一种复数形式，例如，“1 message”和“2 messages”。其他语言（[阿拉伯语](http://www.unicode.org/cldr/charts/latest/supplemental/language_plural_rules.html#ar)、[日语](http://www.unicode.org/cldr/charts/latest/supplemental/language_plural_rules.html#ja)、[俄语](http://www.unicode.org/cldr/charts/latest/supplemental/language_plural_rules.html#ru)等）则具有不同的语法，有更多或更少的[复数形式](http://cldr.unicode.org/index/cldr-spec/plural-rules)。因此，I18n API 提供了灵活的复数转换功能。

`:count` 插值变量具有特殊作用，既可以把它插入翻译，又可以用于从翻译中选择复数形式（根据 CLDR 定义的复数转换规则）：

```ruby
I18n.backend.store_translations :en, inbox: {
  zero: 'no messages', # 可选
  one: 'one message',
  other: '%{count} messages'
}
I18n.translate :inbox, count: 2
# => '2 messages'

I18n.translate :inbox, count: 1
# => 'one message'

I18n.translate :inbox, count: 0
# => 'no messages'
```

`:en` 区域设置的复数转换算法非常简单：

```ruby
lookup_key = :zero if count == 0 && entry.has_key?(:zero)
lookup_key ||= count == 1 ? :one : :other
entry[lookup_key]
```

也就是说，`:one` 标记的是单数，`:other` 标记的是复数。如果数量为零，而且有 `:zero` 元素，用它的值，而不用 `:other` 的值。

如果查找键没能返回可转换为复数形式的散列，就会引发 `I18n::InvalidPluralizationData` 异常。

<a class="anchor" id="setting-and-passing-a-locale"></a>

### 区域的设置和传递

区域设置可以伪全局地设置为 `I18n.locale`（使用 `Thread.current`，例如 `Time.zone`），也可以作为选项传递给 `#translate` 和 `#localize` 方法。

如果我们没有传递区域设置，Rails 就会使用 `I18n.locale`：

```ruby
I18n.locale = :de
I18n.t :foo
I18n.l Time.now
```

显式传递区域设置：

```ruby
I18n.t :foo, locale: :de
I18n.l Time.now, locale: :de
```

`I18n.locale` 的默认值是 `I18n.default_locale` ，而 `I18n.default_locale` 的默认值是 `:en`。可以像下面这样设置默认区域：

```ruby
I18n.default_locale = :de
```

<a class="anchor" id="using-safe-html-translations"></a>

### 使用安全 HTML 翻译

带有 `'_html'` 后缀的键和名为 `'html'` 的键被认为是 HTML 安全的。当我们在视图中使用这些键时，HTML 不会被转义。

```yml
# config/locales/en.yml
en:
  welcome: <b>welcome!</b>
  hello_html: <b>hello!</b>
  title:
    html: <b>title!</b>
```

```erb
# app/views/home/index.html.erb
<div><%= t('welcome') %></div>
<div><%= raw t('welcome') %></div>
<div><%= t('hello_html') %></div>
<div><%= t('title.html') %></div>
```

不过插值是会被转义的。例如，对于：

```yml
en:
  welcome_html: "<b>Welcome %{username}!</b>"
```

我们可以安全地传递用户设置的用户名：

```erb
<%# This is safe, it is going to be escaped if needed. %>
<%= t('welcome_html', username: @current_user.username) %>
```

另一方面，安全字符串是逐字插入的。

NOTE: 只有 `translate` 视图辅助方法支持 HTML 安全翻译文本的自动转换。

![demo html safe](images/i18n/demo_html_safe.png)

<a class="anchor" id="translations-for-active-record-models"></a>

### Active Record 模型的翻译

我们可以使用 `Model.model_name.human` 和 `Model.human_attribute_name(attribute)` 方法，来透明地查找模型名和属性名的翻译。

例如，当我们添加了下述翻译：

```yml
en:
  activerecord:
    models:
      user: Dude
    attributes:
      user:
        login: "Handle"
      # 会把 User 的属性 "login" 翻译为 "Handle"
```

`User.model_name.human` 会返回 `"Dude"`，而 `User.human_attribute_name("login")` 会返回 `"Handle"`。

我们还可以像下面这样为模型名添加复数形式：

```yml
en:
  activerecord:
    models:
      user:
        one: Dude
        other: Dudes
```

这时 `User.model_name.human(count: 2)` 会返回 `"Dudes"`，而 `User.model_name.human(count: 1)` 或 `User.model_name.human` 会返回 `"Dude"`。

要想访问模型的嵌套属性，我们可以在翻译文件的模型层级中嵌套使用“模型/属性”：

```yml
en:
  activerecord:
    attributes:
      user/gender:
        female: "Female"
        male: "Male"
```

这时 `User.human_attribute_name("gender.female")` 会返回 `"Female"`。

NOTE: 如果我们使用的类包含了 `ActiveModel`，而没有继承自 `ActiveRecord::Base`，我们就应该用 `activemodel` 替换上述例子中键路径中的 `activerecord`。

<a class="anchor" id="error-message-scopes"></a>

#### 错误消息的作用域

Active Record 验证的错误消息翻译起来很容易。Active Record 提供了一些用于放置消息翻译的命名空间，以便为不同的模型、属性和验证提供不同的消息和翻译。当然 Active Record 也考虑到了单表继承问题。

这就为根据应用需求灵活调整信息，提供了非常强大的工具。

假设 `User` 模型对 `name` 属性进行了验证：

```ruby
class User < ApplicationRecord
  validates :name, presence: true
end
```

此时，错误信息的键是 `:blank`。Active Record 会在命名空间中查找这个键：

```
activerecord.errors.models.[model_name].attributes.[attribute_name]
activerecord.errors.models.[model_name]
activerecord.errors.messages
errors.attributes.[attribute_name]
errors.messages
```

因此，在本例中，Active Record 会按顺序查找下列键，并返回第一个结果：

```
activerecord.errors.models.user.attributes.name.blank
activerecord.errors.models.user.blank
activerecord.errors.messages.blank
errors.attributes.name.blank
errors.messages.blank
```

如果模型使用了继承，Active Record 还会在继承链中查找消息。

例如，对于继承自 `User` 模型的 `Admin` 模型：

```ruby
class Admin < User
  validates :name, presence: true
end
```

Active Record 会按下列顺序查找消息：

```
activerecord.errors.models.admin.attributes.name.blank
activerecord.errors.models.admin.blank
activerecord.errors.models.user.attributes.name.blank
activerecord.errors.models.user.blank
activerecord.errors.messages.blank
errors.attributes.name.blank
errors.messages.blank
```

这样，我们就可以在模型继承链的不同位置，以及属性、模型或默认作用域中，为各种错误消息提供特殊翻译。

<a class="anchor" id="error-message-interpolation"></a>

#### 错误消息的插值

翻译后的模型名、属性名，以及值，始终可以通过 `model`、`attribute` 和 `value` 插值。

因此，举例来说，我们可以用 `"Please fill in your %{attribute}"` 这样的属性名来代替默认的 `"cannot be blank"` 错误信息。

当 `count` 方法可用时，可根据需要用于复数转换：

| 验证 | 选项 | 信息 | 插值  |
|---|---|---|---|
| `confirmation` | - | `:confirmation` | `attribute`  |
| `acceptance` | - | `:accepted` | -  |
| `presence` | - | `:blank` | -  |
| `absence` | - | `:present` | -  |
| `length` | `:within`, `:in` | `:too_short` | `count`  |
| `length` | `:within`, `:in` | `:too_long` | `count`  |
| `length` | `:is` | `:wrong_length` | `count`  |
| `length` | `:minimum` | `:too_short` | `count`  |
| `length` | `:maximum` | `:too_long` | `count`  |
| `uniqueness` | - | `:taken` | -  |
| `format` | - | `:invalid` | -  |
| `inclusion` | - | `:inclusion` | -  |
| `exclusion` | - | `:exclusion` | -  |
| `associated` | - | `:invalid` | -  |
| `non-optional association` | - | `:required` | -  |
| `numericality` | - | `:not_a_number` | -  |
| `numericality` | `:greater_than` | `:greater_than` | `count`  |
| `numericality` | `:greater_than_or_equal_to` | `:greater_than_or_equal_to` | `count`  |
| `numericality` | `:equal_to` | `:equal_to` | `count`  |
| `numericality` | `:less_than` | `:less_than` | `count`  |
| `numericality` | `:less_than_or_equal_to` | `:less_than_or_equal_to` | `count`  |
| `numericality` | `:other_than` | `:other_than` | `count`  |
| `numericality` | `:only_integer` | `:not_an_integer` | -  |
| `numericality` | `:odd` | `:odd` | -  |
| `numericality` | `:even` | `:even` | -  |

<a class="anchor" id="translations-for-the-active-record-error-messages-for-helper"></a>

#### 为 Active Record 的 `error_messages_for` 辅助方法添加翻译

在使用 Active Record 的 `error_messages_for` 辅助方法时，我们可以为其添加翻译。

Rails 自带以下翻译：

```yml
en:
  activerecord:
    errors:
      template:
        header:
          one:   "1 error prohibited this %{model} from being saved"
          other: "%{count} errors prohibited this %{model} from being saved"
        body:    "There were problems with the following fields:"
```

NOTE: 要想使用 `error_messages_for` 辅助方法，我们需要在 `Gemfile` 中添加一行 `gem 'dynamic_form'`，还要安装 [DynamicForm](https://github.com/joelmoss/dynamic_form) gem。

<a class="anchor" id="translations-for-action-mailer-e-mail-subjects"></a>

### Action Mailer 电子邮件主题的翻译

如果没有把主题传递给 `mail` 方法，Action Mailer 会尝试在翻译中查找主题。查找时会使用 `<mailer_scope>.<action_name>.subject` 形式来构造键。

```ruby
# user_mailer.rb
class UserMailer < ActionMailer::Base
  def welcome(user)
    #...
  end
end
```

```yml
en:
  user_mailer:
    welcome:
      subject: "Welcome to Rails Guides!"
```

要想把参数用于插值，可以在调用邮件程序时使用 `default_i18n_subject` 方法。

```ruby
# user_mailer.rb
class UserMailer < ActionMailer::Base
  def welcome(user)
    mail(to: user.email, subject: default_i18n_subject(user: user.name))
  end
end
```

```yml
en:
  user_mailer:
    welcome:
      subject: "%{user}, welcome to Rails Guides!"
```

<a class="anchor" id="overview-of-other-built-in-methods-that-provide-i18n-support"></a>

### 提供 I18n 支持的其他内置方法概述

在 Rails 中，我们会使用固定字符串和其他本地化元素，例如，在一些辅助方法中使用的格式字符串和其他格式信息。本小节提供了简要概述。

<a class="anchor" id="action-view-helper-methods"></a>

#### Action View 辅助方法

*   `distance_of_time_in_words` 辅助方法翻译并以复数形式显示结果，同时插入秒、分钟、小时的数值。更多介绍请参阅 [datetime.distance_in_words](https://github.com/rails/rails/blob/master/actionview/lib/action_view/locale/en.yml#L4)。
*   `datetime_select` 和 `select_month` 辅助方法使用翻译后的月份名称来填充生成的 `select` 标签。更多介绍请参阅 [date.month_names](https://github.com/rails/rails/blob/master/activesupport/lib/active_support/locale/en.yml#L15)。`datetime_select` 辅助方法还会从 [date.order](https://github.com/rails/rails/blob/master/activesupport/lib/active_support/locale/en.yml#L18) 中查找 `order` 选项（除非我们显式传递了 `order` 选项）。如果可能，所有日期选择辅助方法在翻译提示信息时，都会使用 [datetime.prompts](https://github.com/rails/rails/blob/master/actionview/lib/action_view/locale/en.yml#L39) 作用域中的翻译。
*   `number_to_currency`、`number_with_precision`、`number_to_percentage`、`number_with_delimiter` 和 `number_to_human_size` 辅助方法使用 [number](https://github.com/rails/rails/blob/master/activesupport/lib/active_support/locale/en.yml#L37) 作用域中的数字格式设置。

<a class="anchor" id="active-model-methods"></a>

#### Active Model 方法

*   `model_name.human` 和 `human_attribute_name` 方法会使用 [activerecord.models](https://github.com/rails/rails/blob/master/activerecord/lib/active_record/locale/en.yml#L36) 作用域中可用的模型名和属性名的翻译。像 [错误消息的作用域](#error-message-scopes)中介绍的那样，这两个方法也支持继承的类名的翻译（例如，用于 `STI`）。
*   `ActiveModel::Errors#generate_message` 方法（在 Active Model 验证时使用，也可以手动使用）会使用上面介绍的 `model_name.human` 和 `human_attribute_name` 方法。像 [错误消息的作用域](#error-message-scopes)中介绍的那样，这个方法也会翻译错误消息，并支持继承的类名的翻译。
*   `ActiveModel::Errors#full_messages` 方法使用分隔符把属性名添加到错误消息的开头，然后在 [errors.format](https://github.com/rails/rails/blob/master/activemodel/lib/active_model/locale/en.yml#L4) 中查找（默认格式为 `"%{attribute} %{message}"`）。

<a class="anchor" id="active-support-methods"></a>

#### Active Support 方法

*   `Array#to_sentence` 方法使用 [support.array](https://github.com/rails/rails/blob/master/activesupport/lib/active_support/locale/en.yml#L33) 作用域中的格式设置。

<a class="anchor" id="how-to-store-your-custom-translations"></a>

## 如何储存自定义翻译

Active Support 自带的简单后端，允许我们用纯 Ruby 或 YAML 格式储存翻译。

通过 Ruby 散列储存翻译的示例如下：

```ruby
{
  pt: {
    foo: {
      bar: "baz"
    }
  }
}
```

对应的 YAML 文件如下：

```yml
pt:
  foo:
    bar: baz
```

正如我们看到的，在这两种情况下，顶层的键是区域设置。`:foo` 是命名空间的键，`:bar` 是翻译 `"baz"` 的键。

下面是来自 Active Support 自带的 YAML 格式的翻译文件 `en.yml` 的“真实”示例：

```yml
en:
  date:
    formats:
      default: "%Y-%m-%d"
      short: "%b %d"
      long: "%B %d, %Y"
```

因此，下列查找效果相同，都会返回短日期格式 `"%b %d"`：

```ruby
I18n.t 'date.formats.short'
I18n.t 'formats.short', scope: :date
I18n.t :short, scope: 'date.formats'
I18n.t :short, scope: [:date, :formats]
```

一般来说，我们推荐使用 YAML 作为储存翻译的格式。然而，在有些情况下，我们可能需要把 Ruby lambda 作为储存的区域设置信息的一部分，例如特殊的日期格式。

<a class="anchor" id="customize-your-i18n-setup"></a>

## 自定义 I18n 设置

<a class="anchor" id="using-different-backends"></a>

### 使用不同的后端

由于某些原因，Active Support 自带的简单后端只为 Ruby on Rails 做了“完成任务所需的最少量工作”，这意味着只有对英语以及和英语高度类似的语言，简单后端才能保证正常工作。此外，简单后端只能读取翻译，而不能动态地把翻译储存为任何格式。

这并不意味着我们会被这些限制所困扰。Ruby I18n gem 让我们能够轻易地把简单后端替换为其他更适合实际需求的后端。例如，我们可以把简单后端替换为 Globalize 的 Static 后端：

```ruby
I18n.backend = Globalize::Backend::Static.new
```

我们还可以使用 Chain 后端，把多个后端链接在一起。当我们想要通过简单后端使用标准翻译，同时把自定义翻译储存在数据库或其他后端中时，链接多个后端的方式非常有用。例如，我们可以使用 Active Record 后端，并在需要时退回到默认的简单后端：

```ruby
I18n.backend = I18n::Backend::Chain.new(I18n::Backend::ActiveRecord.new, I18n.backend)
```

<a class="anchor" id="using-different-exception-handlers"></a>

### 使用不同的异常处理程序

I18n API 定义了下列异常，这些异常会在相应的意外情况发生时由后端抛出：

```ruby
MissingTranslationData       # 找不到键对应的翻译
InvalidLocale                # I18n.locale 的区域设置无效（例如 nil）
InvalidPluralizationData     # 传递了 count 参数，但翻译数据无法转换为复数形式
MissingInterpolationArgument # 翻译所需的插值参数未传递
ReservedInterpolationKey     # 翻译包含的插值变量名使用了保留关键字（例如，scope 或 default）
UnknownFileType              # 后端不知道应该如何处理添加到 I18n.load_path 中的文件类型
```

当后端抛出上述异常时，I18n API 会捕获这些异常，把它们传递给 `default_exception_handler` 方法。这个方法会再次抛出除了 `MissingTranslationData` 之外的异常。当捕捉到 `MissingTranslationData` 异常时，这个方法会返回异常的错误消息字符串，其中包含了所缺少的键/作用域。

这样做的原因是，在开发期间，我们通常希望在缺少翻译时仍然渲染视图。

不过，在其他上下文中，我们可能想要改变此行为。例如，默认的异常处理程序不允许在自动化测试期间轻易捕获缺少的翻译；要改变这一行为，可以使用不同的异常处理程序。所使用的异常处理程序必需是 I18n 模块中的方法，或具有 `#call` 方法的类。

```ruby
module I18n
  class JustRaiseExceptionHandler < ExceptionHandler
    def call(exception, locale, key, options)
      if exception.is_a?(MissingTranslationData)
        raise exception.to_exception
      else
        super
      end
    end
  end
end

I18n.exception_handler = I18n::JustRaiseExceptionHandler.new
```

这个例子中使用的异常处理程序只会重新抛出 `MissingTranslationData` 异常，并把其他异常传递给默认的异常处理程序。

不过，如果我们使用了 `I18n::Backend::Pluralization` 异常处理程序，则还会抛出 `I18n::MissingTranslationData: translation missing: en.i18n.plural.rule` 异常，而这个异常通常应该被忽略，以便退回到默认的英语区域设置的复数转换规则。为了避免这种情况，我们可以对翻译键进行附加检查：

```ruby
if exception.is_a?(MissingTranslationData) && key.to_s != 'i18n.plural.rule'
  raise exception.to_exception
else
  super
end
```

默认行为不太适用的另一个例子，是 Rails 的 `TranslationHelper` 提供的 `#t` 辅助方法（和 `#translate` 辅助方法）。当上下文中出现了 `MissingTranslationData` 异常时，这个辅助方法会把错误消息放到 `<span class="translation_missing">` 元素中。

不管是什么异常处理程序，这个辅助方法都能够通过设置 `:raise` 选项，强制 `I18n#translate` 方法抛出异常：

```ruby
I18n.t :foo, raise: true # 总是重新抛出来自后端的异常
```

<a class="anchor" id="conclusion"></a>

## 结论

现在，我们已经对 Ruby on Rails 的 I18n 支持有了较为全面的了解，可以开始着手翻译自己的项目了。

如果想参加讨论或寻找问题的解答，可以注册 [rails-i18n 邮件列表](http://groups.google.com/group/rails-i18n)。

<a class="anchor" id="contributing-to-rails-i18n"></a>

## 为 Rails I18n 作贡献

I18n 是在 Ruby on Rails 2.2 中引入的，并且仍在不断发展。该项目继承了 Ruby on Rails 开发的优良传统，各种解决方案首先应用于 gem 和真实应用，然后再把其中最好和最广泛使用的部分纳入 Rails 核心。

因此，Rails 鼓励每个人在 gem 或其他库中试验新想法和新特性，并将它们贡献给社区。（别忘了在邮件列表上宣布我们的工作！）

如果在 Ruby on Rails 的[示例翻译数据](https://github.com/svenfuchs/rails-i18n/tree/master/rails/locale)库中没找到想要的区域设置（语言），可以[派生仓库](https://github.com/guides/fork-a-project-and-submit-your-modifications)，添加翻译数据，然后发送[拉取请求](https://help.github.com/articles/about-pull-requests/)。

<a class="anchor" id="resources"></a>

## 资源

*   [rails-i18n Google 群组](http://groups.google.com/group/rails-i18n)：项目的邮件列表。
*   [GitHub 中的 rails-i18n 仓库](https://github.com/svenfuchs/rails-i18n)：rails-i18n 项目的代码仓库和问题跟踪器。最重要的是，我们可以在这里找到很多 Rails 的[示例翻译](https://github.com/svenfuchs/rails-i18n/tree/master/rails/locale)，在大多数情况下，它们都适用于我们的应用。
*   [GitHub 中的 i18n 仓库](https://github.com/svenfuchs/i18n)：i18n gem 的代码仓库和问题追踪系统。

<a class="anchor" id="authors"></a>

## 作者

*   [Sven Fuchs](http://svenfuchs.com/)（最初的作者）
*   [Karel Minařík](http://www.karmi.cz/)