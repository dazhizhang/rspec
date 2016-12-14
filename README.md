# rspec

install :

gem install rspec

http://xhh.me/post/2012/10/21/rails-testing-with-rspec



使用 Rspec, Capybara 和 Zeus 等测试 Rails 项目

1. Rails 项目测试

在此我们关注下面两种类型的测试:

单元测试:
针对 models, helpers, mailers 和 lib 等下的代码.
测试代码位于 spec 文件夹下的同名的子文件夹, 例如 spec/models.
测试文件的相对路径和要测试的文件保持一致, 文件名加 _spec, 例如
app/models/user.rb 的单元测试文件应该位于 spec/models/user_spec.rb

验收测试:
模拟用户在浏览器的操作, 对网站的功能进行测试.
验收测试以功能模块为单位进行测试, 例如用户注册, 添加文章等.
验收测试的代码位于 spec/requests 文件夹下 (也可以使用 Cucumber 来做验收测试,
请参见 在 Rails 应用中使用 Cucumber 进行验收测试)

2. 用到的工具

本文使用 rails 3.2 和 ruby 1.9.3

*rspec: BDD 测试框架, 替代 Rails 默认的 TestUnit
*factory_girl: 用于方便的创建测试数据, Rails Test Fixture 的替代品
*shoulda-matchers: 提供一些方便的测试 rails 应用的验证语句
*capybara: 验收测试框架, 用于模拟用户操作测试网站功能
*capybara-webkit: 为 capybara 提供 headless driver, 即在测试时不需要浏览器, 以便在 Linux 持续集成服务器上执行测试
*launchy: 在使用 capybara 测试时通过 save_and_open_save 方法来在浏览器打开当时状态的页面
*Database Cleaner: 用于在测试时清理数据库, 因为 capybara 不支持 rspec 默认的 transactional fixtures
*simplecov: 用于生成测试覆盖率报告
*zeus: 测试辅助工具, 用于加快执行测试和其他 Rails 命令的启动速度

3. 配置及使用

本文中使用到的代码可以在 github上 看到

3.1 创建 rails 应用

rails new my-app --skip-test-unit --skip-bundle
cd my-app
这里使用了 --skip-test-unit 选项来跳过生成 Rails 默认的 test-unit 文件, 因为我们将使用 RSpec

3.2 配置 Gemfile

添加以下内容到 Gemfile

group :test, :development do
  gem 'rspec-rails', '~> 2.0'
end

group :test do
  gem 'factory_girl_rails'
  gem 'shoulda-matchers'
  gem 'capybara'
  gem 'capybara-webkit'
  gem 'launchy'
  gem 'database_cleaner'
  gem 'simplecov', require: false
end
如果你觉得官方的 rubygems 比较慢, 可以使用淘宝的 rubygems 源, 即把 Gemfile 第一行改为:

source 'http://ruby.taobao.org/'
运行命令安装 gems:

bundle install
3.3 配置 rspec

运行命令初始化 rspec:

rails generate rspec:install
这个命令会创建两个文件: .spec 保存运行 RSpec 时的需要使用的选项;
spec/spec_helper.rb 是 RSpec 的加载和配置文件, 供测试文件 require

3.4 一个简单的测试

下面我们来写一个简单的单元测试. 假设我们需要实现一个方法用于产生指定长度的随机字符串,
我们采用 TDD 的方式, 即先写测试, 通过测试来 分析实现的方法 并 保证代码的正确性.

单元测试文件的命名规则 是: 位于 spec 父文件夹下, 子文件夹的组织与要测试的文件一致,
测试文件的名字是 xxx_spec.rb (假设要测试的文件名是 xxx.rb).

我们给这个方法命名为 random_string, 并定义在 MyApp module 的 Utility 类中,
也就是说, 用 MyApp::Utility.random_string 来调用这个方法. 由于这个类属于我们自己的类库,
所以会放在 lib 文件夹下, 于是测试文件就应该是 spec/lib/my_app/utility_spec.rb.
我们先创建这个测试文件及它需要的文件夹结构:

mkdir -p spec/lib/my_app
touch spec/lib/my_app/utility_spec.rb
然后用你喜欢的编辑器打开这个测试文件, 输入:

# encoding: utf-8

require 'spec_helper'

describe MyApp::Utility, '#random_string' do
  context '参数正确' do
    it '应该生成长度正确的字符串' do
      MyApp::Utility.random_string(0).should == ''
      MyApp::Utility.random_string(1).length.should == 1
      MyApp::Utility.random_string(10).length.should == 10
      MyApp::Utility.random_string(100).length.should == 100
    end

    it '多次生成的随机字符串应该不同' do
      MyApp::Utility.random_string(10).should_not == MyApp::Utility.random_string(10)
      # 或者这样测, 如果每次生成同样的字符串, results 数组 uniq 之后就会只有一个元素
      results = 3.times.map { MyApp::Utility.random_string(4) }
      results.uniq.length.should_not == 1
    end

    it '生成的字符串只包含大小写字母和数字' do
      10.times do
        MyApp::Utility.random_string(10).should =~ /\A[A-Za-z0-9]*\Z/
      end
    end
  end

  context '参数非法 - 指定长度为负数' do
    it '应该报错' do
      expect{ MyApp::Utility.random_string(-1) }.to raise_error
      expect{ MyApp::Utility.random_string(-10) }.to raise_error
    end
  end
end
由于 Rails 3.2 不会自动加载 lib 下的文件, 所以我们需要打开 config/application.rb
并找到 config.autoload_paths 这一行, 改为:

    config.autoload_paths += %W(#{config.root}/lib)
然后执行测试. 虽然我们知道测试一定会失败, 因为我们根本还没有编写实现这个方法的文件,
但是 看着测试失败 也是 TDD 步骤的一部分, 是为了和写好实现方法之后测试通过的情况区分开.

rspec spec/lib/my_app/utility_spec.rb
执行结果提示我们 MyApp::Utility 没有定义. 现在我们来写实现. 同样先创建需要的文件和文件夹结构:

mkdir lib/my_app
touch lib/my_app/utility.rb
然后用编辑器打开文件并输入:

module MyApp::Utility
  def self.random_string(_length)
    length = _length.to_i
    raise "Wrong argument 'length' given: #{_length.inspect}" if length < 0

    chars = ('A'..'Z').to_a + ('a'..'z').to_a + ('0'..'9').to_a
    chars_length = chars.length
    length.times.map{ chars[rand(chars_length)] }.join
  end
end
再次执行测试, 并且反复进行 修改代码 -> 执行测试 直到测试通过

完整的测试判定方法请参考 rspec-rails

3.5 执行测试的方式

在 Rails 应用执行 rspec 测试有两种方式:

使用 rake 命令, 示例:
rake test # 执行所有测试, 也可以用 rake spec, 或直接用 rake
rake spec:models # 执行 spec/models 下面的测试
rake spec:helpers # 执行 spec/helpers 下面的测试
rake spec:requests # 执行 spec/requests 下面的测试 (验收测试)
第一次执行这个命令时, 有可能会提示要先执行 rake db:migrate.

使用 rspec 命令 (推荐), 示例:
rspec spec # 执行 spec 下面的测试, 即执行所有测试
rspec spec/models # 执行 spec/models 下面的测试
rspec spec/helpers # 执行 spec/helpers 下面的测试
rspec spec/requests # 执行 spec/requests 下面的测试 (验收测试)
rspec spec/lib # 执行 spec/lib 下面的测试

rspec spec/lib/my_app/utility_spec.rb # 只测试一个文件
rspec spec/lib/my_app/utility_spec.rb:7 # 只执行这个文件第7行所在的单个测试 (每个 "it" 块是一个测试)

rspec spec -t type:request # 执行 type 为 request 的测试, 即验收测试
rspec spec -t '~type:request' # 执行 type 不是 request 的测试

rspec spec -f d # 指定测试结果的格式为 documentation (文档格式)
详细的 rspec 使用方法可以执行 rspec --help 查看.

例如前面那个简单的测试, 使用 -f d 选项会将测试中的 context 等信息按非常便于阅读的格式打印出来:

rspec documentation 格式输出

如果需要某些选项又不想每次都手动输入, 可以把它们添加到 .rspec 文件里作为默认选项, 例如:

--color -f d
3.6 使用 factory_girl 构造测试数据

Rails 默认的 test fixture 可用来构造测试需要的数据, 但是不够灵活. factory_girl
提供了一个更灵活的解决方案.

为了演示, 假设我们需要有个 model Post (文章), 并且有个方法可以发布批量设置多篇文章,
即将给定数组里的所有文章的 published 属性设置为 true. 首先, 我们创建这个 model:

rails g model Post title:string body:text published:boolean
然后执行数据库迁移命令和测试准备命令:

rake db:migrate
rake db:test:prepare # 在数据库结构发生变化时, 需要执行这句来保证用 rspec 执行测试时不会出错
创建 model 时会自动生成对应的测试文件, 即 spec/models/post_spec.rb. 打开这个文件并编写测试代码:

# encoding: utf-8

require 'spec_helper'

describe Post do
  it { should validate_presence_of(:title) }
  it { should validate_presence_of(:body) }
end

describe Post, '#publish_posts' do
  before do
    @posts = FactoryGirl.create_list :post, 10, published: false
  end

  context '全部发布' do
    before { Post.publish_posts(@posts) }
    it 'should work' do
      @posts.all?{ |post| post.published }.should be_true
      # 重新从数据库查询数据
      Post.all.all?{ |post| post.published }.should be_true
    end
  end

  context '部分发布' do
    before { Post.publish_posts(@posts.first(3)) }
    it 'should work' do
      @posts.first(3).all?{ |post| post.published }.should be_true
      Post.find(@posts.first(3).map(&:id)).all?{ |post| post.published }.should be_true

      @posts.last(7).all?{ |post| !post.published }.should be_true
      Post.find(@posts.last(7).map(&:id)).all?{ |post| !post.published }.should be_true
    end
  end
end
第5到8行是使用 shoulda 提供的测试判定语句来测试 model 的验证定义.

测试文件里使用了 FactoryGirl.create_list 方法来方便的创建多个 post. 为此,
我们需要定义 post 的 factory, 创建 spec/factories 文件夹并在下面新建
posts.rb 文件:

FactoryGirl.define do
  factory :post do
    sequence(:title) { |n| "Post #{n}" }
    sequence(:body) { |n| "This is post #{n}" }
    published false
  end
end
这个文件定义了使用 FactoryGirl.create, FactoryGirl.create_list,  FactoryGirl.build,
FactoryGirl.build_list 等方法创建 post 时, 采用的默认属性值 (关于 factory_girl
的详细使用方法请参考其官方文档).

然后执行测试并确认测试是失败的:

rspec spec/models/post_spec.rb
编写实现, 之后再次运行测试. 实现代码示例:

class Post < ActiveRecord::Base
  attr_accessible :title, :body, :published

  validates :title, :body, presence: true

  def self.publish_posts(posts)
    transaction do
      posts.each do |post|
        post.update_attributes published: true
      end
    end
  end
end
3.7 使用 capybara 编写验收测试

(注: 也可以使用 Cucumber 来做验收测试, 请参见 在 Rails 应用中使用 Cucumber 进行验收测试)

关于 capybara 的使用参考自 RailsCasts 257: Request Specs and Capybara.
下面列出配置步骤和一个测试示例.

首先配置 capybara. 打开 spec/spec_helper.rb 文件, 按下面的注释修改:

# 在 require 'rspec/autorun' 这行下面添加
require 'capybara/rails'
require 'capybara/rspec'
Capybara.javascript_driver = :webkit

  # 下面这行配置由 true 改为 false
  config.use_transactional_fixtures = false

  # 在文件的最后一个 end 之前, 即 config.order = "random" 下面, 添加下面的代码
  config.before(:suite) do
    DatabaseCleaner.strategy = :truncation
  end

  config.before(:each) do
    DatabaseCleaner.start
  end

  config.after(:each) do
    DatabaseCleaner.clean
  end
下面是一个测试示例, 用来定义及验证文章列表和文章详情页面是否符合要求:

# encoding: utf-8

require 'spec_helper'

feature '查看文章列表和文章详情' do
  background do
    # 三个已发布的文章: Post 1, Post 2, Post 3
    @published_posts = FactoryGirl.create_list :post, 3, published: true
    # 两个未发布的文章: Post 4, Post 5
    @unpublished_posts = FactoryGirl.create_list :post, 2, published: false

    visit posts_path
  end

  scenario '可以看到已发布的文章及链接' do
    @published_posts.each do |post|
      page.should have_content(post.title)
    end
  end

  scenario '应该看不到未发布的文章' do
    @unpublished_posts.each do |post|
      page.should_not have_content(post.title)
    end
  end

  context '查看文章详情' do
    background do
      @post = @published_posts.first
      click_link @post.title
    end

    scenario '可以看到文章的标题和内容' do
      page.should have_content(@post.title)
      page.should have_content(@post.body)
    end

    scenario '可以通过 "返回" 链接回到文章列表' do
      click_link '返回'
      current_path.should == posts_path
    end
  end
end
具体实现请参考 项目源码.

3.8 使用 simplecov 查看测试覆盖率

首先需要配置 simplecov, 在 spec/spec_helper.rb 的第一行之前添加下面的代码:

require 'simplecov'
SimpleCov.start 'rails'
以后在你运行过测试之后, 就可以用浏览器可以打开 coverage/index.html 查看测试覆盖率情况.
例如:

rake spec # 运行所有测试
open coverage/index.html # 打开测试覆盖率报告, 此命令适用于 Mac OS X 系统

# Linux 用户可以直接调用浏览器的命令来打开, 例如使用 Chromium 浏览器:
chromium coverage/index.html
# 或者使用 Firefox:
firefox coverage/index.html
别忘了把 coverage 文件夹加入源码管理工具的忽略列表, 例如 git 的 .gitignore:

/coverage
3.9 使用 zeus 加速测试

每次执行测试时都需要启动 Rails 环境, 浪费不少时间. Zeus 的原理就是启动好一个 Rails
环境, 供之后的测试直接使用. 另外 rails console, rake 等命令也可以使用
zeus console, zeus rake 来快速执行.

首先安装并配置 zeus (不需要把 zeus 加到 Gemfile):

gem install zeus
zeus init
会创建一个 zeus.json 的配置文件, 可以修改它来配置 zeus, 例如不需要 Cucumber
的话可以去掉 Cucumber 相关的配置.

这样, 在以后的开发过程中, 只要在一个 console 里启动 zeus, 就可以通过快速执行测试等命令了.

启动 zeus: zeus start

使用 zeus 执行测试, 例如:

zeus rspec spec # 执行所有测试
zeus rspec spec/models/post_spec.rb # 执行单个测试文件
目前使用 zeus 执行测试会有一个 产生空白测试覆盖率报告的 bug.
如果要查看测试覆盖率, 可以直接运行 rsepc 命令 (不使用 zeus) 来运行一次测试, 然后查看测试覆盖率.

与 zeus 类似的工具有很多, 例如 spork. 在我使用 spork 时, 发现在测试
Active Admin 实现的页面时有问题, 而 zeus 没有问题, 并且 zeus 的配置更加简单.

参考

在 Rails 应用中使用 Cucumber 进行验收测试
