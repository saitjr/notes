# DSL

实现自己的 DSL，比如...Rake，再比如...Fastlane

```ruby
def event(name)
  yield if block_given?
end

event :m_1 do
  puts "this is m_1"
end
```

细节，单独写博客吧 ~
