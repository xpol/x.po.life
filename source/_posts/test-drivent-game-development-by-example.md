---
title: 测试驱动游戏开发：一个例子
date: 2017-01-07 17:26
---

什么是[测试驱动开发（Test-driven development）](https://en.wikipedia.org/wiki/Test-driven_development)？测试驱动开发在开发每一个功能时按照下面的步骤：

1. 编写单元测试
2. 运行测试，测试失败
3. 编写代码
4. 使测试通过
5. 重构改进代码

测试驱动开发适合游戏开发吗？

良好设计的游戏逻辑，适合测试驱动开发；界面啊、特效啊、交互啊……这些就不太容易进行测试驱动开发。


下面我们举一个例子来看一下如何对游戏逻辑进行测试驱动开发。

## 例子

### 背景

某种升级游戏中，为玩家和 AI 提供跟牌的必选牌和可选牌的组合。代码采用 Lua 编写，测试使用 [busted](https://olivinelabs.com/busted/) 。

### 需求分析

对问题进行分类，从简单的入手，分析如下：

```Markdown
## 头家出 n (n=1,2,4,6,8...) 张牌，跟牌的时候根据当前玩家手牌：
  - 1. 同花色无牌；则无强制出牌，可跟手牌中任意 n 张牌的组合。
  - 2. 同花色有牌但是牌不够或者刚好够：
  - 2.1. 同花色所有手牌必出
  - 2.2. 如果不够，用任意其他手牌补足 n 张
```

这里只列出了两种最简单的情况，实际的列表要长的多，列表嵌套的层级也更深。

需要注意的是，同等级的点之间一定要完整覆盖问题，而且不从叠。


### 添加单元测试

基于 busted 编写第一个需求，缺牌情况下的单元测试如下：

```lua
describe(':findFollows()', function()
  describe('1. when led suit is void in hand.', function()
    it('allows play any cards in hand', function()
      local hand = Hand('♥9', {'♥A','♥A','♠7','♣A','♣K','☆','★'})
      local mandatory, choices = hand:findFollows('♦', 1)
      assert.are.same({}, mandatory) -- no mandatory cards
    end)
  end)
end)
```
`describe` 中采用和需求相同的编号来表示这段代码对应的需求，下面实现代码注释也是这样标识的。

这个时候运行单元测试，将出现测试失败。

### 实现

接下来完成第一中情况下的代码实现：

```lua
function Hand:findFollows(ledsuit, n)
  local totalCardsInLeadSuit = ...
  local cardsInLedSuit = ...

  -- 1.
  if totalCardsInLeadSuit == 0 then
    return {}, combineOf(self:all(), n)
  end
end
```

这里省略了部分实现细节，只列出了代码骨架。在实现过程中不断运行测试，直到通过就表示完成改功能了。

### 重构

之后可以进行代码进行重构和改进。在这个过程中，有可能发现一些额外的测试需要添加。

### 第二个功能的单元测试

然后开始下一个需求的单元测试编写：

```lua
  describe('2. have not enough led suit cards in hand', function()
    local hand
    before_each(function()
      hand = Hand('♥9', {'♦A','♦A','♠7','♣A','♣K','☆','★'})
    end)
    it('2.1. all cards in led suit are mandatory', function()
      local mandatory, _ = hand:findFollows('♦', 4)
      assert.are.same({'♦A','♦A'}, mandatory)
    end)
    it('2.2. rest use all cards combinations', function()
      local _, choices = hand:findFollows('♦', 4)
      assert.are.same({low={'♠7','♣A'}, score={'♣K', '♠7'}}, choices)
    end)
  end)
```

### 第二个功能的实现

然后对应的实现：


```lua
function Hand:findFollows(ledsuit, n)
  -- 1.
  -- 2.
  if totalCardsInLeadSuit <= n then
    local nl = n-totalCardsInLeadSuit
    local others = {}
    if nl > 0 then
      others = self:cardsExcept(ledsuit)
    end
    return cardsInLedSuit, combineOf(others, nl)
  end
end
```

### 重构

再次，可能才实现中的代码有些不优雅的地方，重构它。

### 循环

按照上面的循环进行更多功能的开发：

到现在你大概理解单元测试和游戏开发如何结合了。

## 结束语

测试驱动游戏开发要点：

1. 测试从需求作为出发点，这样能保证实现的功能是需求想要的；
2. 按照 写测试、测试失败、写功能代码、测试通过、重构 的循环来进行开发；
3. 用和需求相同的编号来标识功能点，方便功能跟踪。
