# Method Missing

消息转发的最后一层。

## 运用 1. Hashie::Mash

Hashie::Mash 可以看做一个将 Hash 映射为 Object 的类。

```ruby
# Hash
map = {}
map[:a] = 1
map[:a] # => 1

# Hashie::Mash
map = Hashie::Mash.new
map.a = 1
map.a # => 1
```

### 使用场景

- 用来做 response 的 body 解析

### 具体实现

可以看出 `Hashie::Mash` 是 Hash 的一个增强版，继承自 `Hash`，内部通过 `method_missing` 实现。

```ruby
module Hashie
  class Mash < Hashie::Hash
    def method_missing(method_name, *args, &block)
      method_name_s = method_name.to_s
      # 如果存在，则直接返回 value
      return self.[](method_name, &block) if has_key?(method_name_s)
      # 如果不存在，则通过正则 method name，来判断是 getter，setter，或者是 ?，! 等
      matched = method_name_s.match(/(.*?)([?=!]?)$/)
      case matched[2]
      # 赋值操作
      when "="
        self[matched[1]] = args.first
      # 返回默认值
      else
        default(method_name, *args, &block)
      end
    end
  end
end
```

## 运用 2. 动态代理

利用 `method_missing` 将未实现的方法，转发给别的对象。

### 使用场景

- 可以考虑用来做三方类的封装，避免所有方法需要逐一实现
- 数据库与 Model 层的映射（ActiveRecord）
- iOS 中用 weakProxy 解决 NSTimer 强引用的问题

### 比如...

```ruby
class ModelProxy
  attr_reader :sql_obj

  def self.find_by(*options)
    @sql_obj = SQLLayer.new.get_obj(self.class.name, options)
    self.new
  end

  def method_missing(method_name, *args)
    @sql_obj.send method_name
  end
end

class SQLLayer
  def get_obj(t, *where)
    `SELECT * FROM #{t} WHERE #{where}`
  end
end

class User < ModelProxy
end

user = User.find_by(name: "saitjr") # 实例 User 对象，内部其实是创建了一个 SQL 的对象
user.name # => 调用 name 属性，但是 user 没有这个属性，通过 method_missing 转发给 SQL 对象处理
user.email # => 同 name 属性

```

## 注意事项

### 1. `respond_to?`

如果用 `method_missing` 的方式动态代理了某个方法，如 `user.name`。此时，`respond_to` 会始终返回 `false`。

```ruby
class User
  def method_missing(method_name, *args)
    super if method_name != :name
    "saitjr"
  end
end

user = User.new
user.respond_to?(:name) # => false
```

正确的做法是，实现 `respond_to_missing?` 方法，并给对应的方法返回 `true`：

```ruby
class User
  def respond_to_missing?(method_name, *args)
    return true if method_name == :name
    super
  end
end

user.respond_to?(:name) # => true
```

### 2. 被遗忘的 `super`

无论是重现 `method_missing` 还是 `respond_to_missing?`，都不要忘记在不满足特定条件时，call super。确保能返回正确的值，或及时抛出异常。

### 3. 死循环

在重写 `method_missing` 的时候，小心在 `method_missing` 又调用了不存在的方法，而出现死循环。

```ruby
class User
  def method_missing(method_name, *args)
    send :my_email if method_name == :email
  end
end

User.new.email
```

### 4. 继承自 `Object` 带来的污染

如果不显式指定父类，对象将会继承自 `Object`。此时，如果调用如 `display` 这类 `Object` 的方法，并不会执行 `method_missing`，导致出现比较隐蔽的 bug。

```ruby
class User
  def method_missing(method_name, *args)
    puts "call method missing"
  end
end

User.new.display # => 不会输出 call method missing
```

对此，Objective-C 的 `NSProxy` 更贴切。在 Ruby 中，应该显式继承 `BasicObject`，并且 `undef_method`，删除可能会造成冲突的方法。
