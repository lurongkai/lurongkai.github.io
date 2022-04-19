---
title: "coolshell puzzle guides"
date: 2014-08-05T00:00:00+08:00
draft: false
type: "post"
tags:
    - "Puzzle"
---
[CoolShell](http://www.coolshell.cn/)博主[陈皓](http://weibo.com/haoel)做了一个在线的puzzle很有意思，链接在[这里](http://fun.coolshell.cn/)，这里记录一下解题的一些步骤。

## Puzzle 0
```
++++++++[>+>++>+++>++++>+++++>++++++>+++++++>++++++++>+++++++++>++++++++++>+++++++++++>++++++++++++>+++++++++++++>++++++++++++++>+++++++++++++++>++++++++++++++++<<<<<<<<<<<<<<<<-]>>>>>>>>>>>>>>>-.+<<<<<<<<<<<<<<<>>>>>>>>>>>>>---.+++<<<<<<<<<<<<<>>>>>>>>>>>>>>----.++++<<<<<<<<<<<<<<>>>>>>>>>>>>+++.---<<<<<<<<<<<<>>>>>>>>>>>>>>-.+<<<<<<<<<<<<<<>>>>>>>>>>>>>>---.+++<<<<<<<<<<<<<<>>>>>>>>>>>>>---.+++<<<<<<<<<<<<<>>>>>>--.++<<<<<<>>>>>>>>>>>>>.<<<<<<<<<<<<<>>>>>>>>>>>>>>>----.++++<<<<<<<<<<<<<<<>>>>>>>>>>>>>>---.+++<<<<<<<<<<<<<<>>>>>>>>>>>>>>----.++++<<<<<<<<<<<<<<.
```
如果之前没有听说过变态的编程语言，就让你见识一下。[BrainFuck](http://www.muppetlabs.com/~breadbox/bf/)也称BF，是一门只有8个指令构成的图灵完备的语言。CoolShell博主陈皓写过一篇简单的介绍在[这里](http://coolshell.cn/articles/1142.html)
具体的指令解释不多说了，直接打长[这里](http://esoteric.sange.fi/brainfuck/impl/interp/i.html)，把上面的指令粘进去，运行得到下一关的地址：`welcome.html`。

## Puzzle welcome.html
```
X * Y
2, 3, 6, 18, 108, ?
What is the meaning of life, the universe and everything?
生命、宇宙以及任何事情的终极答案
```
这题有两个线索，首先是这串数字，其次是`生命、宇宙以及任何事情的终极答案`。数字序列找规律并不复杂，每个数字是前两个数字之积，那么直接用`18 * 108`的结果`1944`尝试进入下一关，发现只找到了一个答案。第二个答案很有意思，或者说很`极客`很`宅`，直接google发现和《银河系漫游指南》有关，wiki地址在[这里](http://en.wikipedia.org/wiki/Answer_to_Life,_the_Universe,_and_Everything)。

用`1944 * 42`的答案`81648`进入下一关。

## Puzzle 81648.html
```
macb() ? lpcbyu(&gbcq/_\021%ocq\012\0_=w(gbcq)/_dak._=}_ugb_[0q60)s+
```
放眼忘去是`Dvorak`键盘，点图片可以看到详细信息。那么意图很明显了，`Dvorak`和`QWERTY`键盘转换一下看看会怎么样？这里有个在线的[转换工具](http://wbic16.xedoloh.com/dvorak.html)，然后：

```
main() { printf(&unix["\021%six\012\0"],(unix)["have"]+"fun"-0x60);}
```
WTF……搜了一下，这是87年国际C语言混乱大赛的一段代码。C语言了解不多，趁这个机会了解了解戳[这里](http://blog.csdn.net/jishu360/article/details/8104499)，还有[这里](http://blog.itpub.net/12443821/viewspace-671745)，列一些解此题的关键知识点。

# unix关键字相当于`#define unix 1`
# 数组的引用，`array[num]`和`num[array]`效果相同，所以(unix)["have"] 等于"have"[unix]，结果是`a`，ASCII是`0x61`
# `0x61 + "fun" - 0x60`相当于对`fun`右移`0x61`指针再左移`0x60`指针，也就是说`fun`右称一位，结果是`un`
# `\021`是换页，于是`&unix["\021%six\012\0"]`约为`&unix["\n%six\n\0"]`，`%s`被替为前面的`un`，`&unix`再跳过了第一个`\n`，所以结果是`unix`
使用`unix`，进入下一题。

## Puzzle unix.html
微信扫码，得到：

```
[abcdefghijklmnopqrstuvwxyz] <=> [pvwdgazxubqfsnrhocitlkeymj]
```
看来是一个简单的替代，请出大shell

```
echo Wxgcg txgcg ui p ixgff, txgcg ui p epm. I gyhgwt mrl lig txg ixgff wrsspnd tr irfkg txui hcrvfgs, nre, hfgpig tcm liunz txg crt13 ra "ixgff" tr gntgc ngyt fgkgf. | tr "pvwdgazxubqfsnrhocitlkeymj" "abcdefghijklmnopqrstuvwxyz"
```
得到：

```
Where there is a shell, there is a way. I expect you use the shell command to solve this problem, now, please try using the rot13 of shell to enter next level.
```
这里我脑抽了一下，没看仔细以为是用shell命令，尝试各种`tr`以及`tr xxx xxx`都不行，后来出去跑了个步，回来再读一遍发现是用`rot13`转换一下`shell`这个字符串，wtf……

```
echo "shell" | tr "[A-Za-z]" "[N-ZA-Mn-za-m]"
```
顺利得到下一关地址：`furyy`

### Puzzle furyy.html
啊回文，差点一口老血死在这里……

刚开始思路有点歪，总以为是用`cat`再加上什么关键字得出一个单词，`cat`的规律其实已经注意到了，只是在纠结于怎么找到后续字母的规律……妈的走的有点远扯着蛋了。

后来又是认真读了下揭示语`The answer has been lost in the source`，咦……

`Ctrl + Shift + I`打开Chrome的源代码查看工具（IE F12），看见一坨乱码躺在那里，就是它了，很明确，正则表达式。把乱码扔到一个文本文件里，然后执行：

```
cat data.txt | grep -E "([A-Z])(\d{1})[a-z](\2)(\1)|(\d{1})([A-Z])[a-z](\6)(\5)" -o
```
输出

```
E1v1E
4FaF4
9XrX9
O3i3O
0MaM0
4GbG4
M5l5M
0WeW0
Y0s0Y
```
使用所有中间的字母`variables`过关。

## Puzzle variables.html
点图片，进入[http://fun.coolshell.cn/n/2014](http://fun.coolshell.cn/n/2014)，显示了一个数字，这时候我脑残的用显示的数据作为answer，显然不行。
看到url里的`2014`了吗？用网页里的数字替换`2014`，然后又出现一个新数据，哦原来是这样……接下来就简单多了，上bash脚本。

```
#!/bin/bash

baseUrl="http://fun.coolshell.cn/n"
result="2014"

while [[ $result -ne "" ]]; do
    result=$(curl -s $baseUrl/$result)
    echo "=> $result"
done
```
输出片断：

```
=> 32722
=> 13310
...
=> 16626
=> 20446
=> Cool! the next level is "tree"
run.sh: line 6: [[: Cool! the next level is "tree": syntax error in expression (error token is "! the next level is "tree"")
```
虽然脚本不完美，但是没办法，因为不知道最后会输出什么，这样看来搞出error也是不错的选择……使用`tree`过关。

## Puzzle tree.html
这道题很直白，纯考功底。先还原二叉树，然后找出最深路径，使用此关键字解密字符串`U2FsdGVkX1+gxunKbemS2193vhGGQ1Y8pc5gPegMAcg=`。

由于只有中序的后序的遍历序列，所以需要先从后序序列倒着取出根节点，然后再通过中序序列和后序序列来确立左子树和右子树，此过程递归完成。用js写了一段代码来还原二叉树：

```
var inOrderSeq   = ["T", "b", "H", "V", "h", "3", "o", "g", "P", "W", "F", "L", "u", "A", "f", "G", "r", "m", "1", "x", "J", "7", "w", "e", "0", "i", "Q", "Y", "n", "Z", "8", "K", "v", "q", "k", "9", "y", "5", "C", "N", "B", "D", "2", "4", "U", "l", "c", "p", "I", "E", "M", "a", "j", "6", "S", "R", "O", "X", "s", "d", "z", "t"];
var postOrderSeq = ["T", "V", "H", "o", "3", "h", "P", "g", "b", "F", "f", "A", "u", "m", "r", "7", "J", "x", "e", "w", "1", "Y", "Q", "i", "0", "Z", "n", "G", "L", "K", "y", "9", "k", "q", "v", "N", "D", "B", "C", "5", "4", "c", "l", "U", "2", "8", "E", "I", "R", "S", "6", "j", "d", "s", "X", "O", "a", "M", "p", "W", "t", "z"];

function parseTree(inOrder, postOrder) {
    if (inOrder.length == 0 || postOrder.length == 0) {
        return null;
    }

    var root = postOrder[postOrder.length - 1];
    var inOrderIndex = inOrder.indexOf(root);

    var leftTreeInOrder = inOrder.slice(0, inOrderIndex);
    var leftTreePostOrder = postOrder.slice(0, inOrderIndex);

    var rightTreeInOrder = inOrder.slice(inOrderIndex + 1);
    var rightTreePostOrder = postOrder.slice(inOrderIndex, -1);

    var node = {
        node: root,
        left: parseTree(leftTreeInOrder, leftTreePostOrder),
        right: parseTree(rightTreeInOrder, rightTreePostOrder)
    };

    return node;
}

var tree = parseTree(inOrderSeq, postOrderSeq);
```
OK，树还原出来了，还需要进行最深路径查找脑中瞬间闪出DFS，完全由于`深度优先`四个字。

，这个和深度优先搜索有点像但并不是搜索，通过递归的方式得到最长的路径。顺便提一嘴，BFS(广度优先)借助队列实现，DFS(深度优先)借助栈实现。好了，大学课程回忆完毕，上代码：

```
function findDeepestPath(selfNode) {
    if (selfNode == null) {
        return [];
    }

    if (selfNode.left == null || selfNode.right == null) {
        return [selfNode];
    }

    var leftPathSeq  = findDeepestPath(selfNode.left);
    var rightPathSeq = findDeepestPath(selfNode.right);

    if (leftPathSeq.length > rightPathSeq.length) {
        return [selfNode].concat(leftPathSeq);
    } else {
        return [selfNode].concat(rightPathSeq);
    }
}
```
得到结果`zWp8LGn01wxJ7`，保留这个结果，因为下道题还用的到。

```
echo U2FsdGVkX1+gxunKbemS2193vhGGQ1Y8pc5gPegMAcg= | openssl enc -aes-128-cbc -a -d -pass pass:zWp8LGn01wxJ7
```
`nqueens`，进入下一题。

## Puzzle nqueens.html
这个倒是简单粗暴，考回溯算法，求解n后，然后暴力穷举出答案。写求n后解。本想用C#写个小程序，然后生成一个带有所有解txt文件，后来想想何必呢，还是继续用js吧，有node呢……

```
var crypto = require('crypto');
var passwd = "zWp8LGn01wxJ7";
var target = "e48d316ed573d3273931e19f9ac9f9e6039a4242";

function writeResult(results) {
    var resultStr = results.map(function (v) {
        return v + 1;
    }).join("");

    var hashString = passwd + resultStr + "\n";
    var sha1 = crypto.createHash('sha1');
    sha1.update(hashString)
    var digest = sha1.digest('hex') ;
    if(digest == target) {
        console.log(resultStr);
    }
}

function conflict(results, theLocation) {
    for (var row = 0; row < results.length; row++) {
        var queenLoction = results[row];

        if (queenLoction == theLocation) {
            return true;
        }

        var rowDiff = results.length - row;
        var locationDiff = Math.abs(theLocation - queenLoction);

        if (rowDiff == locationDiff) {
            return true;
        }
    }

    return false;
}

function solveQueens(size, results) {
    if (typeof results === "undefined") { results = []; }
    for (var pos = 0; pos < size; pos++) {
        if (conflict(results, pos)) {
            continue;
        }

        // pos is good.
        var newResults = results.slice(0);
        if (newResults.length == size - 1) {
            newResults.push(pos);
            writeResult(newResults);
        } else {
            newResults.push(pos);
            solveQueens(size, newResults);
        }
    }
}
```
调用`solveQueens(9)`并在node下运行即可。
最终的代码合并了计算sha1的部分，注意sha1字符串的最后一定要加`\n`，否则是算不出结果的。
顺利拿到下一关入口``953172864`。

## Puzzle 953172864.html
通过对第一列的观察，发现这是一个26进制/10进制的对应表。那么接下来的总是就是如果将`COOLSHELL / SHELL`转化为10进制并转换回26进制对应的字母的问题。

首先COOLSHELL对应的10进制数字分别为`3 15 15 12 19 8 5 12 12`，计算公式如下：

```
3*26^8+15*26^7+15*26^6+12*26^5+19*26^4+8*26^3+5*26^2+12*26^1+12*26^0
```
使用Excel或Google计算结果为`751743486376`。同理，SHELL结果为`8826856`。
接下来将两个数相除的结果`85165`（这个数是带小数，但是因为除数和被除数都是整数所以结果也是整数）转回26进制，即`辗转相除法`，重复`模`26并取余数，得到`15 25 21 4`，对应的字母为`DUYO`，Bingo进入最后一题。

## Puzzle DUYO.html
最后一关是最轻松的一关，从图片得出线索是[猪圈密码](http://en.wikipedia.org/wiki/Pigpen_cipher)，然后通过一幅简单的图可轻松逆向出答案：`helloworld`.
ps: 通过图形的形状找图中该图形所对应的字母。

![pigpen_cipher_key](pigpen_cipher_key.png)

pps: 昨天答完的时候通过[Top100](http://fun.coolshell.cn/top100.html)看到是六十多名，今天再打开发现自己的成绩不见了很奇怪，耗子叔叔我没作弊啊……