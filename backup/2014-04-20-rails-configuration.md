---
layout: post
tags : [rails]
title: rails 中的Configuration

---

* 每个Rails::Railtie（和子类）对象都有一个独立的config对象（但是所属class不相同）

* `config.paths` 对象是`Rails::Paths::Root` 实例

* Configuration 继承链

        Rails::Application::Configuration.ancestors
        => [Rails::Application::Configuration, Rails::Engine::Configuration, Rails::Railtie::Configuration, Object, PP::ObjectMixin, ActiveSupport::Dependencies::Loadable, JSON::Ext::Generator::GeneratorMethods::Object, Kernel, BasicObject]


* 类Rails::Application中的config定义（实例方法）

        def config #:nodoc:
          @config ||= Application::Configuration.new(find_root_with_flag("config.ru", Dir.pwd))
        end

* Rails::Engine的config定义（实例方法）

        delegate :middleware, :root, :paths, to: :config
        ......
        def config
          @config ||= Engine::Configuration.new(find_root_with_flag("lib"))
        end

* Rails::Railtie的config定义（实例方法）

        def config
          @config ||= Railtie::Configuration.new
        end

* `config.middleware`

  定义：

          def middleware
            @middleware ||= Rails::Configuration::MiddlewareStackProxy.new
          end

  `Rails::Configuration::MiddlewareStackProxy`

        class MiddlewareStackProxy
          def initialize
            @operations = []
          end

          #除了merge_into，其他方法都只是存了类似一个操作日志，有什么好处？缓存动作延迟执行吗？
          def insert_before(*args, &block)
            @operations << [__method__, args, block]
          end

          alias :insert :insert_before

          def insert_after(*args, &block)
            @operations << [__method__, args, block]
          end

          def swap(*args, &block)
            @operations << [__method__, args, block]
          end

          def use(*args, &block)
            @operations << [__method__, args, block]
          end

          def delete(*args, &block)
            @operations << [__method__, args, block]
          end

          def merge_into(other) #:nodoc:
            # 虽然other要求相应相同的方法，但是可以是不同的类型（鸭子类型），这里实际使用时传的时ActionDispatch::MiddlewareStack实例,真实的中间件实例
            @operations.each do |operation, args, block|
              other.send(operation, *args, &block)
            end
            other
          end
        end

---

### Rails::Paths

**Rails::Paths::Root**

        class Root
          attr_accessor :path

          def initialize(path)
            @current = nil
            @path = path
            @root = {}                                 #存储键值对，key是标识名称，键是path对象
          end

          def []=(path, value)
            glob = self[path] ? self[path].glob : nil
            add(path, with: value, glob: glob)
          end

          def add(path, options = {})
            with = Array(options.fetch(:with, path))         #没有with（Path对象的pathes）的话，那就用key代替
            @root[path] = Path.new(self, path, with, options) #对应Path对象的 root current pathes
          end

          .....

          def autoload_once
            filter_by(:autoload_once?)
          end

          def eager_load
            filter_by(:eager_load?)
          end

          def autoload_paths
            filter_by(:autoload?)
          end

          def load_paths                         # Railtie对象根据config.paths.load_paths 获得
            filter_by(:load_path?)
          end


**Rails::Paths::Path**

    class Path
      include Enumerable

      attr_accessor :glob

      def initialize(root, current, paths, options = {}) #options 可以有这几个参数 eager_load, autoload, #autoload_once and glob (应该还有load_path吧)
        @paths    = paths
        @current  = current
        @root     = root
        @glob     = options[:glob]   # 表明这个Path对象的pathes想要的是匹配期望

        options[:autoload_once] ? autoload_once! : skip_autoload_once!
        options[:eager_load]    ? eager_load!    : skip_eager_load!
        options[:autoload]      ? autoload!      : skip_autoload!
        options[:load_path]     ? load_path!     : skip_load_path!
      end

      def children
        keys = @root.keys.select { |k| k.include?(@current) }
        keys.delete(@current)                                   通过root的key和自己的key（current）对比得到children，去掉自己，剩下的children的所有path
        @root.values_at(*keys.sort)
      end

      ......

      # Expands all paths against the root and return all unique values.
      def expanded                                               # 依据root展开所有path（root连接自己的path），并唯一
        raise "You need to set a path root" unless @root.path
        result = []

        each do |p|
          path = File.expand_path(p, @root.path)

          if @glob && File.directory?(path)            #glob 只有在pathes是目录才有效
            Dir.chdir(path) do
              result.concat(Dir.glob(@glob).map { |file| File.join path, file }.sort)
            end
          else
            result << path
          end
        end

        result.uniq!
        result
      end

      # Returns all expanded paths but only if they exist in the filesystem.
      def existent                                        #存在的所有
        expanded.select { |f| File.exists?(f) }
      end

      def existent_directories
        expanded.select { |d| File.directory?(d) }
      end

      alias to_a expanded

---

### Engine 的各种path设置和来源：


* `config.path` 的实例方法

        def autoload_once
          filter_by(:autoload_once?)
        end

        def eager_load
          filter_by(:eager_load?)
        end

        def autoload_paths
          filter_by(:autoload?)
        end

        def load_paths
          filter_by(:load_path?)
        end

* `config.paths.load_paths`

        Engine::Configuration

        paths.add "lib",                 load_path: true
        paths.add "vendor",              load_path: true

* 下面3个来自`Engine::Configuration`的实例方法

        attr_writer :middleware, :eager_load_paths, :autoload_once_paths, :autoload_paths #所以这三个支持外部写入
                                                                                          #也支持从paths读取
        def eager_load_paths
          @eager_load_paths ||= paths.eager_load
        end

        def autoload_once_paths
          @autoload_once_paths ||= paths.autoload_once
        end

        def autoload_paths
          @autoload_paths ||= paths.autoload_paths
        end

* `config.autoload_paths`

  `Engine::Configuration` 中无配置

* `config.eager_load_paths`

        Engine::Configuration

        paths.add "app",                 eager_load: true, glob: "*"
        paths.add "app/controllers",     eager_load: true
        paths.add "app/helpers",         eager_load: true
        paths.add "app/models",          eager_load: true
        paths.add "app/mailers",         eager_load: true

        paths.add "app/controllers/concerns", eager_load: true
        paths.add "app/models/concerns",      eager_load: true

  Engine 中定义了实例方法

        def eager_load!
          config.eager_load_paths.each do |load_path|
            matcher = /\A#{Regexp.escape(load_path)}\/(.*)\.rb\Z/
            Dir.glob("#{load_path}/**/*.rb").sort.each do |file|
              require_dependency file.sub(matcher, '\1')
            end
          end
        end

  同时也有类方法： `delegate :eager_load!, to: :instance`

  调用处`Rails::Application::Finisher` 中的initializer

        initializer :eager_load! do
          if config.eager_load     # 这个是 Rails::Application::Configuration的实例方法，在
            ActiveSupport.run_load_hooks(:before_eager_load, self)
            config.eager_load_namespaces.each(&:eager_load!) # 对所有非虚拟的Engine调用类方法eager_load!
          end
        end

  Engine 中：

        def inherited(base)
          unless base.abstract_railtie?
            Rails::Railtie::Configuration.eager_load_namespaces << base

  把所有非虚拟的Engine存入eager_load_namespaces

  最后`config.eager_load`  这个是 `Rails::Application::Configuration`的实例方法，在项目的`config/environments/XXXX.rb`里进行了赋值：

        /Users/zhonghua/code/work/r4test/config/environments/development.rb:11:10:  config.eager_load = false
        /Users/zhonghua/code/work/r4test/config/environments/production.rb:11:10:  config.eager_load = true

* `config.autoload_once_paths`

  `Engine::Configuration` 中无配置


