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

  def method_missing(method_name)
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

### 1. `response_to?`


### 2. 继承自 `Object` 带来的污染
