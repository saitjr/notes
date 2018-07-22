# 重构 Computer 类

旧代码：

```ruby
class Computer
  def initialize(id, data)
    @id = id
    @data = data
  end

  def mouse
    info = data.get_mouse_info(@id)
    price = data.get_mouse_price(@id)
    return "#{info} - #{price}"
  end

  def cpu
    info = data.get_cpu_info(@id)
    price = data.get_cpu_price(@id)
    return "#{info} - #{price}"
  end

  def keyboard
    info = data.get_keyboard_info(@id)
    price = data.get_keyboard_price(@id)
    return "#{info} - #{price}"
  end

  # def ...
end
```

因为方法名不同，出现了非常多的重复代码。


## v1.0 利用 `send` 动态调用

```ruby
class Computer
  def initialize(id, data)
    @id = id
    @data = data
  end

  def mouse
    get_component_info :mouse
  end

  def cpu
    get_component_info :cpu
  end

  def keyboard
    get_component_info :keyboard
  end

  def get_component_info(name)
    info = data.send "get_#{name}_info", @id
    price = data.send "get_#{name}_price", @id
    "#{info} - #{price}"
  end
end
```


## v2.0 利用 `define_method` 动态生成

`define_method` 是 Module 的类方法，可通过该方法来动态生成方法。

```ruby
class Computer
  def initialize(id, data)
    @id = id
    @data = data
  end

  def self.define_component(name)
    define_method(name) do
      info = @data.send "get_#{name}_info", @id
      price = @data.send "get_#{name}_price", @id
      "#{info} - #{price}"
    end
  end

  define_component :mouse
  define_component :cpu
  define_component :keyboard
end
```

## v3.0 利用内省减少代码

### 内省与反射

反射（Reflective）：能在运行时动态修改程序结构，如动态方法调用、实现，创建新的对象等。

内省（Introspection）：能在运行时动态查看自身属性，如 `response_to?`，`instance_of?` 等。

```ruby
class Computer
  def initialize(id, data)
    @id = id
    @data = data

    # 遍历 data 全部 public 方法，筛选出 get_x_info，然后动态生成方法
    data.public_methods.grep(/get_(.*)_info/) do |matched|
    	self.class.define_component(matched)
    end
  end

  def self.define_component(name)
    define_method(name) do
      info = @data.send "get_#{name}_info", @id
      price = @data.send "get_#{name}_price", @id
      "#{info} - #{price}"
    end
  end
end
```

## v4.0 利用动态代理，直接转发

动态代理 ———— 捕获幽灵方法（Ghost Method），并转发给另一个对象的中间代理。

```ruby
class Computer
	def initialize(id, data)
    @id = id
    @data = data
  end

  # 由 Computer 代理，转发给真正相应 get_x_info 方法的 data
  def method_missing(method_name)
    info_method = "get_#{method_name}_info"
    price_method = "get_#{method_name}_price"
    super if !@data.respond_to?(info_method)
    info = @data.send info_method, @id
    price = @data.send price_method, @id
    "#{info} - #{price}"
  end
end
```
