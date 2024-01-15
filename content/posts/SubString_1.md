+++
title =  "子字符串查找"
author = "AsukaMilet"
date =  "2023-02-03"
categories =  "Algorithm"
tags = ["字符串", "算法"]
description = "主要介绍了关于子字符串查找的基本思想和两种常用算法:BruteForce、Boyer-Moore"
math =  true
+++

字符串的一种基本操作就是*子字符串查找*：给定一段长度为*N*的文本和一个长度为*M*的模式(pattern)字符串，在文本中找到一个和该模式相符的子字符串。解决该问题的大部分算法都可以容易地扩展为找出文本中所有和该模式相符的子字符串、统计该模式在文本中的出现次数的算法。

![](https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202023-02-03%20202939.png)

<center>子字符串的查找</center>

在进行相关算法的研究之前，我们应该记住模式相对于文本是非常短的（*M*可能等于100或者1000），而文本相对于模式是很长的（*N*可能等于100万或10亿）。在字符串查找中，一般会对模式进行预处理来支持在文本中的快速查找。

# 暴力子字符串查找算法

子字符串查找的一个最显而易见的方法就是在文本中模式可能出现匹配的任何地方检查匹配是否存在。

```c++
int search(const string&pat,const string&txt)
{
    //也可以直接使用pat.length()和txt.length()
    auto M=pat.length();
    auto N=txt.length();
    //当i>N-M时，pat超过剩余字符串的长度，不可能匹配成功(设i=N-M+1，则M>N-i)
    for(int i=0;i<=N-M;i++)
    {
        int j;
        for(j=0;j<M;j++)
        {
            if(txt[i+j]!=pat[j])	//从匹配的字符开始往后扫描
                break;
        }
        if(j==M) return i;	//找到匹配
    }
    return N;	//未找到匹配
}
```

<center>暴力子字符串查找</center>

在上面的算法中，使用了一个指针`i`跟踪文本，一个指针`j`跟踪模式。对于每个`i`，首先将`j`重置为0并不断将它增大，直到找到了一个不匹配的字符或者模式结束(`j==M`)为止。如果在模式字符串结束之前文本字符串就已经结束了(`i==N-M+1`)，那么就没有找到匹配：模式字符串在文本中不存在。规定不匹配时返回文本字符串的长度。

<img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202023-02-03%20210204.png" style="zoom: 55%;" />

<center>暴力子字符串查找</center>

> 命题 在最坏情况下，暴力子字符串查找算法在长度为*N*的文本中查找长度为*M*的模式需要 *~NM* 次字符比较

## 显式回退

除了上述实现，暴力子字符串匹配算法还有另一种实现。和之前一样，我们使用一个文本指针`i`和一个模式指针`j`。在`i`和`j`指向的字符相匹配时，代码进行的字符比较和上一个实现相同。在下面的代码中，`i`值相当于上述代码的`i+j`：它指向文本中已经匹配过的字符序列的*末端*（`i`以前指向的为该序列的*开头*）。如果`i`和`j`指向的字符不匹配，那么*回退*这两个指针的值：将`j`重新指向模式的开头，将`i`指向本次匹配的开始位置的下一字符。

```c++
int search(const string&pat,const string&txt)
{
    int j,M=pat.length();
    int i,N=txt.length();
    for(i=0,j=0;i<N&&j<M;i++)
    {
        if(txt[i]==pat[j])	j++;
        else	//不匹配时i回退至开头再迭代指向下一个字符
        {
            i-=j;
            j=0;
        }
    }
    if(j==M)	return i-M;	//返回匹配的子字符串开头
    else	return N;
}
```

# 暴力子字符串匹配的缺陷

在暴力匹配中，算法的主要低效之处就是回退问题，例如：假定字母表中只有两个字符，待查找的模式字符串为B A A A A A A A A A。现在假设已经匹配了模式中的5个字符，第6个字符匹配失败。当发现不匹配的字符时，可以知道文本中的前6个字符肯定是B A A A A B(前5个匹配，第6个失败)，文本指针现在指向的是末尾的字符B。此时其实并不需要回退文本指针`i`，因为正文中的前4个字符都是A，不可能和模式的第一个字符B匹配。

![](https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202023-02-03%20224818.png)

<center>文本字符串的指针在子字符串查找中的回退</center>

为了解决这个问题，下面介绍一个性能优越的算法。

#  Boyer-Moore算法

Boyer-Moore算法会根据匹配失败时*文本*和*模式*中的字符来决定下一步的行动。为了介绍Boyer-Moore算法，首先看一个在文本F I N D I N A H A Y S T A C K N E E D L E中查找模式N E E D L E的过程。

![](https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/Boyer-Moore%20Algorithm.png)

<center>从右向左的(Boyer-Moore)子字符串查找</center>

## 启发式地处理不匹配的字符

1. 因为是从右向左的与模式进行匹配，首先会比较模式字符串中的E和文本中的N(位置为5的字符)。此时N和E失配，但N在模式字符串中，就在模式字符串的开头位置。为了避免错过N的匹配，将N与模式串中最近的N对齐。

   > Q：为什么不将位置为2的字符和模式串的N对齐？
   >
   > A：设想将位置为2的字符和模式串的N对齐，则位置为5的N会和模式串[N..E]之间的字符对齐，不可能匹配成功

2. 然后比较模式字符串最右侧的E和文本中的S(位置为10的字符)，匹配失败。因为S不在模式串中，因此6-10位置模式串均会包含S，不可能匹配。直接将模式串的位置移到S的下一位开始匹配。

3. 此时模式字符串最右侧的E和文本中的位置为16的E相匹配，但我们发现文本的下一个(位置为15的)字符为N，匹配再次失败。

4. 和第一次一样，我们将文本字符N和模式串中最近的N对齐。最后，从位置20处开始从右向左扫描，发现文本中含有与模式匹配的子字符串。

通过这种方法，找到匹配位置仅用了4次字符比较(以及6次比较来验证匹配)！

### 规律

通过上述示例，我们可以总结Boyer-Moore算法启发式地处理不匹配字符的规律：

1. 文本指针从左向右扫描，模式指针从右向左扫描

2. 失配的时候，右移模式串，使得文本串中失配的字符和模式串中左边最近的相同字符对齐。如果不存在相同字符，就把整个模式串右移到失配字符的右侧。

   ------
   
## 跳跃表

要实现启发式地处理不匹配地字符，我们使用`right[]`记录字母表中的每个字符在模式中出现的*最靠右*的地方，目的是找到文本串中失配的字符和模式串中左边最近的相同字符出现在模式串中的位置(如果字符在模式串中不存在则记为-1)。要将`right[]`数组初始化，首先将所有元素的值设为-1，然后对于0到M-1的`j`，将`right[pat.charAt(j)]`的值设为`j`。

> **Note：**`right[]`的长度由基数R决定

<img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/right%20array%20computation.png" style="zoom:45%;" />

<center>Boyer-Moore算法跳跃表的计算</center>

## 子字符串查找过程

现在让我们总结Boyer-Moore算法的整个过程：我们用一个索引`i`在文本中从左向右移动，用另一个索引`j`在模式中从右向左移动。如果从M-1到0的所有`j`，`txt.charAt(i+j)`都和`pat.charAt(j)`相等，那么就找到了一个匹配。否则匹配失败，出现以下三种情况：

- 如果造成匹配失败的字符不包含在模式串中，将模式串向右移动`j+1`个位置(即将`i`增加`j+1`)
- 如果造成匹配失败的字符包含在模式串中，那就可以使用`right[]`数组来将模式字符串和文本对齐，使得该字符和它在模式串中出现的最右位置相匹配
- 如果这种方式无法增大`i`，那就直接将`i`加1来保证模式串至少向右移动了一个位置(`i`不能回退，只能前进)

<img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/Basic%20Boyer_Moore.png" style="zoom: 50%;" />

<center>Boyer-Moore算法的基本思想</center>

## 算法实现

```c++
#define R 256	//ASCII表的大小
class BoyerMoore
{
    public:
    //计算跳跃表
    BoyerMoore(const std::string&pat):pat(pat)
    {
        for(int c=0;c<R;c++)
            right[c]=-1;
        //从左向右扫描模式串，反复更新模式串中出现的字符最右的位置
        for(int j=0;j<pat.length();j++)
            right[static_cast<int>(pat[j])]=j;
    }
    
    int search(const std::string&txt)
    {
        int N=txt.length();
        int M=pat.length();
        int skip;
        //i>N-M时，剩余的字符串比模式串短，不可能出现匹配
        for(int i=0;i<=N-M;i+=skip)
        {
            skip=0;
            //模式串从右向左进行扫描匹配
            for(int j=M-1;j>=0;j--)
            {
                if(pat[j]!=txt[i+j])
                {
                    //发生不匹配时文本指针才跳跃
                    skip=j-right[static_cast<int>(txt[i+j])];
                    if(skip<1)  skip=1;
                    break;
                }
            }
            //如果将模式串从右向左扫描完都不发生跳跃，说明找到匹配
            if(skip==0)	return i;
        }
        return N;
    }
    
    private:
    int right[R];
    std::string pat;
};
```

# 性能总结

| 算法            | 最坏情况   | 一般情况     | 在文本中回退 | 正确性 | 额外的空间需求 |
| --------------- | ---------- | ------------ | ------------ | ------ | -------------- |
| 暴力算法        | \\( MN \\) | \\( 1.1N \\) | 是           | 是     | 1              |
| Boyer-Moore算法 | \\( MN \\) | \\( N/M \\)  | 是           | 是     | \\( R \\)      |

# Reference

1. 算法：第4版/（美）Sedgewick,R. ，(美）Wayne,K.著；谢路云译.  ——北京：人民邮电出版社，2012.10
2. [字符串匹配 - Boyer–Moore 算法原理和实现 by 春水煎茶](https://writings.sh/post/algorithm-string-searching-boyer-moore)
3. [Substring Search Lecture by Sedgewick](https://algs4.cs.princeton.edu/lectures/keynote/53SubstringSearch.pdf)
