---
title: "Wordle优化"
date: 2022-03-31T11:49:28+08:00
tags: [Rust, Wordle]
math: true
---



### 结果

目前的代码能通过可以被认为是作弊的方法得到hard模式下的最优解：
初始词 salet, 一共猜测8116次，最多的单词猜7次。

如果需要证明这是最优的，需要遍历所有的单词，枚举所有的可能步骤。这需要大量的时间。不过能力有限，我打算就到此为止了，本文会简单的记录我所发现和使用的一些优化方法。

### 起点

前文已经给出了目标解的形式，一棵决策树，节点是猜测的单词，状态是转移条件。同时，这棵树是具有最优子结构的。

```
best[guess] = min(sum(best[next_guess], pattern for next_guess)) for next_guess in all available guesses
```

从形式上来说，具备树形dp的一切条件。不过这并没有太多改善，因为每一步的状态转移可能性仍然有12000种，而我们也看到最优解最深需要猜7次。这个复杂度是不能接受的。


### 用整数表示状态

其实这个优化我在很晚才做，虽然这个优化是显然的，不管时间还是空间上，都会得到很大的改善。

但是它会导致状态丧失直观性，我几乎没有办法用肉眼检查计算是否正确。不过多亏了Rust严格的编译期检查，我在后期还算简单地完成了这个修改。

```rust
pub fn check(target: &str, guess: &str) -> u8 {
        let mut freq = BTreeMap::<char, usize>::new();
        for (guess_c, target_c) in guess.chars().zip(target.chars()) {
            if guess_c != target_c {
                let counter = freq.entry(target_c).or_insert(0);
                *counter += 1;
            }
        }

        let mut pattern: u8 = 0;
        let mut base: u8 = 1;
        for (guess_c, target_c) in guess.chars().zip(target.chars()) {
            if guess_c == target_c {
                pattern += 2 * base;
            } else {
                let counter = freq.entry(guess_c).or_insert(0);
                if *counter > 0 {
                    pattern += base;
                    *counter -= 1;
                } 
            }

            base *= 3;
        }

        pattern
    }
```

### 初步剪枝

首先，由于上文大坑的原因，每回合我们需要考虑的guess范围是所有可行解，而不只是可能解。这就导致很多guesses可能对这回合毫无帮助，例如总是猜同一个单词。这会导致无限循环。所以一个很简单的条件就是检查这个guess能把当前所有可能的解分成多少个pattern，如果猜完还是一个group，而且这个group不是GGGGG，那么这个猜测就是毫无意义的，我们可以直接忽略。

```rust
 if groups.len() == 1 && !groups.contains_key(242) {
            continue
        }
```

对于每一个guess，我们需要依次检查所有pattern, 但是可能检查到中间某一个pattern的时候，就发现它的猜测次数就已经超过当前找到的可行解。那么这个guess的剩下的pattern就不用继续检查了。 

```rust
if current_guess.total_count > best_of_all_guess.total_count {
    current_guess.has_result = false;
    break
}
```

这个检查还能不能继续改进呢？

想要这个不等式成立，我们可以分别考虑增加左边和减少右边。

我们先考虑减少右边，右边最小值也就是最优值了。意味着我们需要尽可能早的检查最有可能是最优解的guess，那么这里就可以用到前文提到的启发式策略了。我们先检查期望信息量最大的guess:

```rust
pub fn get_entropy_sum<'a>(guess: &'a str, answers: &BTreeSet<&'a str>) -> (&'a str, u32, BTreeMap<u8, BTreeSet<&'a str>>) {
    let groups = group_by_pattern(guess, answers);

    let entropy = groups.iter().map(|(pattern, group)| {
        get_entropy(*pattern, group.len() as u32)
    }).sum();

    (guess, entropy, groups)
} 
```

然后在枚举guess前，按照entropy排序，优先检查期望信息量最大的guess即可。

那么左边能否增加呢？左边是当前guess已经检查过的pattern需要的猜测次数之和，我们有没有可能预判一下剩下的pattern需要多少猜测次数呢？需要注意的是，这个预判的次数只能比真实的次数少，而不能多。

我在第一次预估这个次数时，考虑的是，对于一个比如长度为4的group，每回合我的猜测都能对半分，那么log2次就能猜出单词，所以总次数就是4 * log2(4)得到8次。即使不考虑第一次可能就猜对，导致总和可能只需要7次。最大的问题其实在于，这里最好情况并不只是2分，这里每一个guess可能的pattern有243种(当然，这里面有一些不可能出现的pattern, 比如GGGGY)！所以对于长度不超过243的group来说，最好情况可能是1次猜测就能他们全都分好组，最后再猜一次即可。这里应该是log243而不是log2。

当然，group大于243的情况并不是很常见。我们可以简单的用 2 * length - 1 的公式来估计这个最理想情况。一次猜测区分所有结果并且命中其中一个，然后其他结果的情况就最多再猜一次。

有了这个预估，我们可以做到两个优化：

1. 在检查具体的pattern前，我们就能计算得到这个guess的最理想情况：
    sum(2*group.len() - 1) for all groups.
```rust
    if current_guess.total_count + lower_bound > best_of_all_guess.total_count {
        continue;
    }

```

2. 在中间某一步，我们可以把当前的pattern的预估用已经检查完成过的最优值替换，然后再来比较：

```rust
    // 更新当前guess的猜测此时
    current_guess.update(pattern, sub_result);

    // 计算出当前guess的预估值
    let current_entropy = get_entropy(pattern, pattern_answers.len() as u32);
    // 从第一步计算得到的lower_bound减去当前group的预估值，相当于用真实值代替掉预估值
    lower_bound -= current_entropy;

    // 最后再来做比较
    if current_guess.total_count  + lower_bound > best_of_all_guess.total_count {
        current_guess.has_result = false;
        break
    }
```

### 猜测范围的小优化

在分析刚才的优化过程中，我发现了另外一个小的优化点。

1. 如果猜测只剩下两个值了，无论我猜哪一个，都只剩下一个解，我们随便猜一个都是最优解。这个时候就不用去检查每一个可能值了。

2. 如果猜测范围还剩下三个值，如果我猜三个可能值之外的值，最好情况也就是三分，然后各猜一次，一共6次。那我不如直接猜这个三个值，最坏情况也就是最后才猜到，总的猜测次数还是1+2+3=6次。

综上，当可能解的个数为2~3时，可以只考虑可行解，而不用去遍历全集。这个优化虽然小但非常有效，因为在搜索树的末端，大部分都是这种小的group。

### 确定初始词

其实对于每一步可以猜测的词来说，词与词之间的运算是完全独立的。所以我们可以先固定某个搜索词进行探索，然后针对不同的搜索词进行相同的计算即可。

初始词的选择也是按照启发式的顺序排序的，在3b1b的补充视频里有提到这个过程。但是这个过程不必很严谨，因为最后都是要遍历所有初始词的。

我这里直接取leaderboard上的最优词，salet。

### 限制搜索深度

显然，最优解的深度不会太深，但是很难证明这一点。我们可以通过慢慢放宽这个限制，从5次无解，到6次有解，到7次可能是最优解。但是无法证明8次，9次或者更多次的解不会更优。

如果我们需要证明解是最优的，我个人理解是不能有这个条件的。但是不妨碍我们在已知最优解深度的情况下，通过限制这个深度来得到最优解。

我的代码里设置的常数是5，但是得到的解可以为7。因为深度5，可行解还剩一个的时候，我是直接返回的一个解法，而没有做限制，加上当前这一层，所以最大深度可以到7。


### 去重
有一些猜测词猜完之后，会导致结果分组情况完全一致（极端一点就是他们都完全没用。）。这时我们能根据group的情况来去重。可以有效减少重复计算的运算量。

```rust
    // 去重的集合，key就是分组情况
    let mut group_patterns = BTreeSet::<BTreeMap<u8, BTreeSet<&str>>>::new();
    let mut preprocess_by_guess:Vec<_> = availables
        .iter()
        .filter_map(|guess| {
            let (guess, entropy, groups) = get_entropy_sum(guess, &answers);

            // 如果这个组合已经出现过，就不再重复计算了。
            if group_patterns.contains(&groups) {
                return None
            }
    
            group_patterns.insert(groups.clone());

            return Some((guess, entropy, groups))
        })
        .collect();
```

### group排序

对于一个待检查的guess，我们对于所有的pattern是从小到大来还是从大到小来呢？

理论上，这里配合上文预估total_count的优化，最好是能尽快达到best of guess的total count，这样就能尽早退出循环。

从小到大，计算快但是增长慢。 从大到小增长快，但是计算慢。试验下来，从小到大会快很多。原因不明。

### 一些看上去没有什么效果的优化

#### 预估深度

和前面预估计算量类似，我们可以预估深度。方法也很简单：

```rust
pub fn get_lower_bound_level(length: usize) -> u8 {

    match length {
        1 => 1,
        2..=243 => 2,
        _ => 3
    }
}
```

原理和之前类似，最理想情况就是243分。那么长度为1就是1次，其他都是2次。大于243的group很少，我们可以统一归为3次。

然后检查如果当前深度加上最少的深度大于上文中限制的最大深度，提前退出。但是好像这个条件不太会触发，所以最后也没有加上。

#### 并发

前面提到，由于词与词之间的运算是完全独立的。我们完全可以并发的来计算不同的词。

但是，我们不能在每一层的计算都完全并行，完全并行之后，上面的优化就全都无效了。这样我们就只能靠计算能力来应付所有的可能性。这样是得不偿失的。

其次，如果只在第一层并行，这里的瓶颈其实在于最大的一个group，其他的线程其实很快就能计算完成，然后一起围观这个最大的group。

那么如果把task继续细化呢，把前两层的所有可能性拿出来，然后并发是不是可以把任务分的更均匀呢？这里去重前的任务大概有80w：

1. 如果每一种情况，都提前计算剩余的可行解和可行的猜测集合，需要大量的内存，而且不管是插入还是查询都非常慢。

2. 如果只划分任务，线程自己单独计算可行解和可行的猜测，这里有大量重复的计算，时间上效果也并不好。

综上，这个优化的效果非常有限。

#### 记忆化

最后来介绍cache，其实这个优化很容易想到，但是做了很多次效果都很一般，即使参考了其他人对cache更进一步的利用之后，效果还是非常不明显。所以最后也没有用上。

首先，我们需要定义状态，判断当前状态是否出现过，其实有很多条件。最简单的一些有：

1. 当前的可行解范围
2. 猜测范围，通过不同路径得到相同的可行解范围，可猜测范围不一定相同，所以这也得是状态的一部分。

如果采用了上文中提到的搜索深度限制，我们还需要考虑剩余的搜索次数，如果之前到这个状态，剩余搜索次数是2次，而当前有三次，那显然，前面得到的不一定是最优解。

所以只有剩余搜索次数完全相同的情况下，才能直接返回前面计算的结果。

更深一层，如果当前剩余猜测次数是3次：

1. 前面有剩余4次的记录，而且结果是无解，我们可以直接返回无解。
2. 如果有剩余2次的记录，而且有解，虽然他可能不是最优解。但我们可以拿来作为当前最优解。

不过实现后发现，无解的记录一次都不会被利用到。

以下是相关代码：

```rust
// 定义
    pub type Cache<'a> = BTreeMap<Restriction, BTreeMap<BTreeSet<&'a str>, BTreeMap<u8, Best<'a>>>>;

// 存
    cache  
        .lock()
        .unwrap()
        .entry(restrictions)
        .or_insert_with(BTreeMap::new)
        .entry(availables.to_owned())
        .or_insert_with(BTreeMap::new)
        .insert(current, best_of_all_guess.clone());

// 取
    let mut best_of_all_guess = Best::new();

    if let Some(restrictions_cache) = cache.lock().unwrap().get(&restrictions) {
        if let Some(answers_cache) = restrictions_cache.get(answers) {

            // 有记录的最优解:
            if let Some(level_cache) = answers_cache.get(&current) {
                counter.result_counter += 1;
                return level_cache.clone();
            }

            // 有记录的无解:
            // for level in 0..(current - 1) {
            //     if let Some(level_cache) = answers_cache.get(&level) {
            //         if !level_cache.has_result || level_cache.max_level + current <= MAX_TURNS {
            //             counter.no_result_counter += 1;
            //             return level_cache.clone()
            //         }
            //     }
            // }
            
            // 有记录的解作为当前最优解:
            for level in (current + 1) .. MAX_TURNS {
                if let Some(level_cache) = answers_cache.get(&level) {
                    counter.baseline_counter += 1;
                    best_of_all_guess = level_cache.clone();
                    break
                }
            }
        }
    }

```

这里稍微记录一下Rust相关的知识点，由于cache需要被全局访问并更新，并发的情况下还要线程安全。在Rust里需要被包裹两次：

```rust
    // 真正的cache需要定义成这样
    let cache = Arc::new(Mutex::new(Cache::new()));

    // 访问的时候需要先lock再unwrap
    cache  
        .lock()
        .unwrap()
```

#### 状态压缩

这里其实是承接上面cache的，可以想象cache需要的内存非常之大。那么状态可不可以压缩呢？

其实，一串可行解和另一串可行解之间可能是同构的。我们以数字来举例：

比如a状态的可行解有以下三种： 111,112,113

b状态的可行解有以下三种： 444,445,446

可以看出，他们本质上是相同的状态，通过a状态的决策树也能很容易的得到b状态的决策树。

但是，我们很难判断两个状态是否是同构的。

另外，在wordle这个游戏里，可行解都必须是单词，想让两组单词同构，可想而知是不容易的。

### 作弊

其实上面限制深度已经是类似作弊的手段了，但至少可以得到一个在限制深度的情况下最优解。接下来要提到的优化则可能得不到最优解。

到目前为止，用上上述所有的方法，我也只能在合理时间(1min)里算出1400个单词的最优解。如果放开到全部2300个单词，没有尝试。

无奈之下，我尝试了这个剪枝方法：对于每个回合，我只猜测最有可能的几个guess，后面的直接不看了。

我把这个阈值限制在13的时候，可以计算出榜上的最优解。

显然这个结论几乎没有意义，只能给我们得出一个看上去很不错的解。但是不能证明它在任何情况下是最优的。加上这个剪枝后，可以在13s左右得到salet在hard模式下的最优解:8116。




