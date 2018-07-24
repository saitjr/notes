# proc、lambda、&、Mehtod 转换

## 转换方式

```ruby
# Proc
proc = Proc.new { |num1, num2| num1 + num2 }
puts proc.call(1, 2)

# lambda
l1 = lambda { |num1, num2| num1 + num2 }
l2 = -> (num1, num2) { num1 + num2 }
puts l1.call(1, 2)
puts l2.call(1, 2)
puts l1.class # => Proc

# & 用于方法参数与传值
def test
  yield(1, 2) if block_given?
end
puts test(&l1) # & 表示将 l1 当做 block 传入，如果去掉 &，l1 就是一个 lambda

# Method 通过 to_proc 转换
def a_method(num1, num2)
  num1 + num2
end

m = Object.method :a_method
puts m.to_proc.call(1, 2)
```

## Proc 与 lamda 区别

|               | proc                                           | lambda                                    |
| ------------- | ---------------------------------------------- | ----------------------------------------- |
| return 关键字 | 从 proc 作用域返回，差不多就是直接 return 程序 | 从当前 lambda 返回，类似于函数内部 return |
| 传参          | 会自动截断或用 nil 补齐                        | 传入个数必须对应                          |
