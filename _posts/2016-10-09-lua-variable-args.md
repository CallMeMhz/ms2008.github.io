---
layout:     post
title:      Lua 正确处理可变参数
subtitle:   一定要注意 table 中的 hole
date:       2016-10-09
author:     ms2008
header-img: img/post-bg-miui-ux.jpg
catalog:    true
tags:
    - Lua
    - pairs
---

为了在 Lua 里处理可变参数，我们可能会写下面这样的代码：

```lua
local function args(...)
    if next({...}) then
        for _, v in ipairs{...} do
            print(v)
        end
    else
        print("empty var")
    end
end

args(10, 20, 30)

-- output:
-- 10
-- 20
-- 30
```

咋一看，貌似没什么问题，但是当传入 `args(10, nil, 30)` 时，发现并不符合我们的预期：

```lua
-- output:
-- 10
```

原因就在于 `ipairs` 遍历的 table 必须是一个序列。序列的数字索引必须连续。table 中间包括 `nil`，这样的 table 就不是序列，例如：`a={10, nil, 30}` 它不是序列，因为它的数字索引是 1 3 不是连续的, 所以它不是序列。为了处理这个问题，就要引入 `select` 了。

调用 `select` 时，需要传入一个 selector 和变长参数。如果 selector 为数字 n, 那么 `select` 返回它的第 n 个可变实参，否则只能为字符串 #, 这样 `select` 会返回变长参数的总数。

增强后的函数为：

```lua
local function args(...)
    if next({...}) then
        -- get the count of the params
        for i = 1, select('#', ...) do
            -- select the param
            local param = select(i, ...)
            print(param)
        end
    else
        print("empty var")
    end
end

args(10, nil, 30)

-- output:
-- 10
-- nil
-- 30
```

---

> 问: 怎么不用 `pairs` 来处理？

**我个人认为 `pairs` 对 `nil` 的表达能力是极其有限的**，比如这个用例：

```lua
local t = {10, nil, 30}

for k,v in pairs(t) do
    print("in", k,v)
end

print("=============")

for i=1, select('#', unpack(t)) do
    local param = select(i, unpack(t))
    print(param)
end

-- output:
-- in	1	10
-- in	3	30
-- =============
-- 10
-- nil
-- 30
```

可以看到 `pairs` 只遍历了两次，1 => 10、3 => 20 其并没有表现出 2 => nil ，虽然我们可以从结果来推断出这个结论，但是显然，在某些场景下我们更需要的是直接还原出这个结果来。这其实和 cjson 的稀松数组类似。

而使用 `select` 后，我们就可以直接看到期待的效果了。
