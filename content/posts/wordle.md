---
title: "Wordle"
date: 2022-03-16 14:48:07
tags: [Python]
math: true
---


### 起因

在B站上看了3b1b的[科普视频](https://www.bilibili.com/video/BV1zZ4y1k7Jw)，介绍了wordle游戏中的一些信息论原理。在视频里，作者实现了一个solver，并且直观地介绍了它的原理。不过在视频的结尾提到了通过多探索一步，提升了solver的效果。当时看到这里我就在想，难道这个方法并不是最优解？很快就能发现，视频中的每一步guess，都是基于当前的最优值，所以这只是一个简单的贪心算法，很可能不是最优的。当然，视频的主要目的还是直观的介绍信息熵等概念，解法是不是最优并不重要。

后来，我在评论中看到了一个[Leader Board](https://freshman.dev/wordle/#/leaderboard), 里面给出了最优解，并且提供了一些解法自己的介绍链接. 然后通过这个名单，我才发现这个Wordle这个游戏还有Hard模式，然后Hard模式的平均猜测次数‘理所当然地’比普通模式的多。然而，后面发现，这里其实提示了一个非常大的坑，这个坑会导致普通模式错过最优解，具体关于这个坑的介绍在本文的最后面。

### 游戏介绍

Wordle 的游戏规则很简单，玩家每次猜测一个包含5个字母的单词，然后游戏会反馈一个结果，这个结果也包含5位，每个位置有三种颜色，绿色表示该字母正确且位置正确，黄色表示字母正确但是位置不对，灰色表示答案不包含这个字母，最后要求玩家在5次内猜到出这个单词。

思考了一下这个玩法，很容易联想到以前玩过的小游戏猜数字，每次猜测一个四位数字，游戏返回结果`XAXB`，A代表有几个位置数字和位置都正确，B代表猜对了数字，但是位置不对。

比较这两个游戏，有三个地方是明显的区别：

1. 结果的pattern不一样
2. 猜测的字符wordle可以重复，猜数字不能
3. wordle要求猜测的5个字符是一个valid的单词，而猜数字没有限制

但是这些区别都不本质，我们可以很容易地把这两个游戏抽象成一个游戏。

1. 首先，定义一个游戏猜测的全集，对于wordle来说，是5个字母的单词表，对于猜数字来说，是所有的4位数字。
2. 玩家给出一个猜测，游戏根据规则的不同会反馈不同的pattern，事实上这可以看做将当前所有可能的结果，根据pattern过滤出符合反馈的，缩小下一轮的猜测范围。 
3. 最后不断重复上一个步骤，直到pattern满足胜利条件。

需要注意的是，按照这种模式，给出新的猜测仅与当前可能的结果有关，与之前的猜测和反馈都没有关系。
所以基于这种交互，我们可以定义两个类 `Checker` 和 `Solver` : 

`Checker` 相当于游戏的裁判，每一轮的结果以及游戏是否结束都是`Checker`说了算.
`Solver` 相当于玩家，每一轮根据当前可能的猜测范围提供一个guess，然后根据Checker反馈的结果更新猜测范围。

```py
class Checker:
    # 根据 target 计算当前 guess 的结果
    def check(self, target: str, guess: str) -> str:
        pass

    def is_success(self, pattern) -> bool:
        pass

class Solver:
    def guess(self) -> str:
        pass
    
    def update(self, guess: str, pattern: str):
        pass
```

### 评估策略

想要比较解法的好坏，首先需要合理地评价一个解法。这个游戏通常有两个维度来评价：

1. 平均猜测次数
2. 最大猜测次数

通常，我们采用第一种来评估一个策略的好坏。

同时，根据上面定义的`Checker` 和 `Solver`，我们可以定义一个 `Evaluator`，他相当于一个评测机，遍历所有可能的结果让`Solver`猜，然后统计他的表现。

```python
class Evaluator:
    def evaluate(self):
        results = []
        for target in self.full_set:
            solver = self.solver_type(self.checker, self.full_set, self.max_length)

            current = 0
            while True:
                current += 1
                guess = solver.guess()
                pattern = self.checker.check(target, guess)

                if self.checker.is_success(pattern):
                    break
                else:
                    solver.update(guess, pattern)

            results.append(current)
        
        print(f"Avg: {sum(results) / len(results)}")
        print(f"Max: {max(results)}")
```

注意这里去掉了Solver猜不出来的处理代码，限制最大猜测次数即可。

### 启发式策略

既然是启发式，自然不是最优的。先来两个baseline算法：

1. 每次取可能结果里最小的
2. 可能结果里随机取一个

大部分情况下，随机的表现都比第一种好。

我们一般怎么评估一个猜测好不好呢？这里主要就是视频中的知识了，我们希望一个猜测的信息量最大。

在解释信息量之前，我们需要先定义熵的概念，熵代表不确定性。

假如猜测前有四种可能且四种可能是等概率的，那么此时熵是最大值(因为不确定性最高)。

如果猜完之后，只剩下唯一一种可能了，即结果已经确定了，那么这个猜测的信息量最大。所以我们可以认为信息量等于消除不确定性的大小。

那么我们在猜之前，能评估哪种猜测的效果最好吗？

其实，在`Checker`告诉我们结果之前，我们可以通过假设结果，来评估我们这次的猜测。

以猜数字游戏为例，假如目前还剩'1111', '1112', '1121', '1122' 四种可能，那么对于猜测 1111 来说

1. 如果结果是 1111，反馈即为 4A0B
2. 如果结果是 1112，反馈即为 3A1B
3. 如果结果是 1121，反馈即为 3A1B
4. 如果结果是 1122，反馈即为 2A2B

所以我们分成了三种反馈：

1. 4A0B -> 1111
2. 3A1B -> 1112, 1121
3. 2A2B -> 1122

在猜之前，我们的信息熵是2bit，猜完之后我们的预期信息熵是1bit，所以这个猜测的预期信息量是1bit

但是如果换成猜测1112会怎么样呢？

1. 如果结果是 1111，反馈即为 3A0B
2. 如果结果是 1112，反馈即为 4A0B
3. 如果结果是 1121，反馈即为 2A2B
4. 如果结果是 1122，反馈即为 3A1B

所以我们分成了四种反馈，每种反馈都能唯一确定结果，所以我们的预期信息熵就是0bit，那么这个预期信息量自然就是2bit了。

所以，从信息量的角度来看，1112 是比 1111 更好的一种猜测。

## 最优解

前面提到，上面的算法本质上只是一个贪心算法，不错，但是还不是最优。

对于最优解，我们目前需要通过暴力搜索，走完全部的分支才能得到。如何搜索呢？以下给出一个基本框架：

```py
def dfs(current: int, availables: Set[str]):

    results = {}

    # 遍历每一种guess
    for idx, guess in enumerate(full_set):

        # 假设所有可能的答案，得到下一步的猜测范围
        pattern_results = defaultdict(set)
        for target in availables:
            pattern = checker.check(target, guess)
            pattern_results[pattern].add(target)

        for pattern, pattern_availables in pattern_results.items():
            if not checker.is_success(pattern):
                # 猜对了是搜索的终止条件
                results[guess][pattern] = dfs(current + 1, pattern_availables)

    # 评估 best guess
    best_guess = find_best(results)
    
    return best_guess
```

对于每一棵搜索子树，我们需要返回两个值, 一个是当前子树的最优决策树，另一个是在这种决策下，总的猜测次数。
那么对于当前节点来说，我们首先通过比较每个guess的猜测次数，得到best_guess，然后他的决策树即为：

```
{best_guess: best_sub_decision_tree}
```

注意，这里的best_sub_decision_tree是包含best_guess所有的pattern子树的 `{pattern: sub_decision_tree}`

这里给一棵猜测两位数，每位3个字符的一颗决策树：

```json
{
    "01": {
        "GB": {
            "00": {
                "GG": {},
                "GY": {
                    "02": {
                        "GG": {}
                    }
                }
            }
        },
        "GG": {},
        "YY": {
            "10": {
                "GG": {}
            }
        },
        "BG": {
            "11": {
                "GG": {},
                "YG": {
                    "21": {
                        "GG": {}
                    }
                }
            }
        },
        "BY": {
            "12": {
                "GG": {}
            }
        },
        "YB": {
            "20": {
                "GG": {}
            }
        },
        "BB": {
            "22": {
                "GG": {}
            }
        }
    }
}
```

使用过程也很简单，上来先猜01，然后根据反馈的pattern进入对应的子树，然后猜测子树的父节点。注意，这里pattern的子节点只有一个。

## 大坑

这里大坑主要是在上面dfs过程的第一步，这里枚举所有的guess时，枚举的范围是 full_set 还是当前可行解。

一开始，我想当然的觉得是当前可行解，但是发现，~~wordle的Hard模式其实就是限制了猜测必须是个可行解~~只猜可行解会导致猜测效率的降低，即使是Hard模式，也需要把可行解之外的合法猜测单词考虑进来。这也是为什么[wordle-rs](https://github.com/petertseng/wordle-rs)会强调他的代码只考虑了2315可能的结果，但是考虑了12972种可能的guess.

这里举一个反例，假如我们猜测一个2位数，目前已知十位数是1，那么还剩10~19十种可能

如果我们只猜`1x`，那么每次只能排除一个数，最差可能需要10次才能才对。
但是如果我们猜`23`，`45`这种组合，虽然结果一定不对，但是信息量比`1x`要大。

所以，在搜索过程中，枚举guess时，需要枚举fullset。但是，这样会导致搜索空间膨胀得非常大，优化就变得非常重要了。关于优化的部分我们留到下一篇再来记录。

