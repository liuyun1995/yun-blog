我们知道正则表达式是一种简洁而强大的符号表示方法，它具有高度的概括性，因此人们可以使用简短的字符串来描述一类基于特定字符集的形式语言。可是计算机不能直接识别正则表达式，就像编程语言需要编译器来将代码翻译成机器码一样，正则表达式需要通过正则引擎来翻译成可执行的程序。一般来讲翻译出来的程序会通过构造一个状态机来判断给定的字符串是否可以被接受。由于状态机的识别字符串的过程不能无限地执行下去，因此状态机的状态必须是有限的。

#### 1 有穷自动机

有穷自动机是一种抽象数学模型，它可以根据外部输入来改变自身的状态，从而达到模拟和控制执行流的目的。有穷自动机由五个部分组成，可以用一个五元组 $(S，\Sigma，s，F，\delta)$ 来表示，其中各部分的含义如下所示：

$S$：表示一个有穷状态集合。

$\Sigma$：表示一个输入符号集合。

$s$：代表一个初始状态。

$F$：代表一个接受状态集合。

$\delta$：表示状态之间的转换函数集合。

例如，假设需要识别一个英文字符串是否包含"main"子串，我们可以利用程序来模拟这样一个有穷自动机。

![](https://gitee.com/liuyun1995/yun-blog-image/raw/master/%E6%89%93%E9%80%A0%E6%AD%A3%E5%88%99%E5%BC%95%E6%93%8E%5B2%5D--Thompson%E7%AE%97%E6%B3%95/1.png)

上述是一个非常简单的有穷自动机模型，它从初始状态0开始不断的读入下一个字符并执行状态转换，如果最终自动机能到达接受状态4，则表明输入字符串里面包含"main"子串，否则表明不包含该子串。这样的自动机同样也是由五个部分组成，其中每个部分的具体含义如下：

$S$ ：有限状态集合 $\{0，1，2，3，4\}$

$\Sigma$ ：英文字母表 $\{a，b，c，\dots，z，A，B，C，\dots，Z\}$

$s$ ：初始状态 $0$

$F$：接受状态集合 $\{4\}$

$\delta$ ：状态转换函数集合 $\{(0，m)\to 1，(1，a)\to 2，(2，i)\to 3，(3，n)\to 4 \}$

根据状态转移的性质，有穷自动机(FA)又分为不确定有穷自动机(NFA)和确定有穷自动机(DFA)，NFA允许对空串输入 $\epsilon$ 进行状态转移，并且对同一个输入字符允许转移到多个目标状态。DFA则对这些做了限制，不允许基于空串的状态转移，对同一个输入字符只能转移到一个目标状态。NFA和DFA在表达力上是等价的，任何DFA都是某个NFA的一个特例，同时任何NFA都可以通过一个DFA来模拟。例如下面的NFA和DFA描述的是同一种语言。

1）可以识别模式 $a(b|c)^*$ 的NFA如下所示。

![](https://gitee.com/liuyun1995/yun-blog-image/raw/master/%E6%89%93%E9%80%A0%E6%AD%A3%E5%88%99%E5%BC%95%E6%93%8E%5B2%5D--Thompson%E7%AE%97%E6%B3%95/2.png)

2）可以识别模式 $a(b|c)^*$ 的DFA如下所示。

![](https://gitee.com/liuyun1995/yun-blog-image/raw/master/%E6%89%93%E9%80%A0%E6%AD%A3%E5%88%99%E5%BC%95%E6%93%8E%5B2%5D--Thompson%E7%AE%97%E6%B3%95/3.png)

从上面两幅图可以看出，NFA的状态转移具有不确定性而DFA的状态转移是确定的，对于机器来说不确定性会产生大量回溯，从而导致NFA的执行性能不如DFA。另一方面，基于正则表达式直接构造NFA会比直接构造DFA更加简单并且所需的时间更少，所以在实际应用中需要结合场景来使用NFA或者DFA。一般来说，对于复杂并且需要多次复用的正则表达式，直接编译成DFA来模拟效果会更好；而对于简单并且只使用一次的正则表达式而言，使用NFA来模拟效果会更好。对二者之间具体区别的概括如下表所示。

| 描述                                   | 不确定有穷自动机(NFA) | 确定有穷自动机(DFA) |
| -------------------------------------- | --------------------- | ------------------- |
| 是否允许基于空串 $\epsilon$ 的状态转换 | 是                    | 否                  |
| 单个输入可转换的目标状态数量           | 多个                  | 一个                |
| 基于正则表达式进行构建的复杂度         | 简单                  | 复杂                |
| 初始构建所需时间                       | 少                    | 多                  |
| 识别字符串所需时间                     | 多                    | 少                  |
#### 2 Thompson算法概述

之前我们讨论过复杂的正则表达式可以由简单的正则表达式通过并，连接，闭包等基础的运算构造而成。Thompson算法就是利用这种归纳思想来将一个正则表达式转化成为等价的NFA的。因为单个字符就是最小的正则表达式，所以首先为单个字符生成对应的NFA，然后通过对两个NFA进行并，连接，闭包等运算来构造一个更大的NFA，就这样通过层层递归我们可以构造任意的NFA。Thomspon算法为这些基础运算制定了相应的模版，假设正则表达式s和t对应的NFA分别是N(s)和N(t)，下面展示了如何用两个小NFA构造大NFA的模版。

1）最小NFA构造

空串 $\epsilon$ 和单个字符可以构造一个最小NFA，如下图所示，$i$ 是NFA的开始状态，$f$ 是NFA的接受状态。下图左侧是基于空串构造的NFA，下图右侧是基于字母表中的单个字符构造的NFA。

![](https://gitee.com/liuyun1995/yun-blog-image/raw/master/%E6%89%93%E9%80%A0%E6%AD%A3%E5%88%99%E5%BC%95%E6%93%8E%5B2%5D--Thompson%E7%AE%97%E6%B3%95/4.png)
2）并运算

假设 $r=s|t$， $r$ 的 NFA 即 $N(r)$ 可以通过下图构造得到。这里 $i$ 和 $f$ 是新状态，分别是 $N(r)$ 的开始状态和接受状态。从 $i$ 到 $N(s)$ 和 $N(t)$ 的开始状态各有一个 $\epsilon$ 转换，从 $N(s)$ 和 $N(t)$ 到接受状态 $f$ 也各有一个 $\epsilon$ 转换。请注意，$N(s)$ 和 $N(t)$ 的接受状态在 $N(r)$ 中不是接受状态。因为从 $i$ 到 $f$ 的任何路径要么只通过 $N(s)$，要么只通过 $N(t)$ ，且离开 $i$ 或进入 $f$ 的 $\epsilon$ 转换都不会改变路径上的标号，因此我们可以判定 $N(r)$ 识别。

![](https://gitee.com/liuyun1995/yun-blog-image/raw/master/%E6%89%93%E9%80%A0%E6%AD%A3%E5%88%99%E5%BC%95%E6%93%8E%5B2%5D--Thompson%E7%AE%97%E6%B3%95/5.png)
3）连接运算

假设 $r=st$，$r$ 的 NFA 即 $N(r)$ 可以通过下图构造得到。$N(s)$的开始状态变成了 $N(r)$ 的开始状态。$N(t)$ 的接受状态成为 $N(r)$ 的唯一接受状态。$N(s)$ 的接受状态和$N(t)$ 的开始状态合并为一个状态，合并后的状态拥有原来进入和离开合并前的两个状态的全部转换。一条从 $i$ 到 $f$ 的路径必须首先经过 $N(s)$，因此这条路径的标号以 $L(s)$ 中的某个串开始。然后，这条路径继续通过 $N(t)$，因此这条路径的标号以 $L(t)$ 中的某个串结束。

![](https://gitee.com/liuyun1995/yun-blog-image/raw/master/%E6%89%93%E9%80%A0%E6%AD%A3%E5%88%99%E5%BC%95%E6%93%8E%5B2%5D--Thompson%E7%AE%97%E6%B3%95/6.png)
4）闭包运算

假设 $r=s^*$，$r$ 的 NFA 即 $N(r)$ 可以通过下图构造得到。这里 $i$ 和 $f$ 是两个新状态，分别是 $N(r)$ 的开始状态和接受状态。要从 $i$ 到达 $f$，我们可以沿着新引入的标号为 $\epsilon$ 的路径前进，这个路径对应于 $L(s)^0$ 中的一个串。我们也可以到达 $N(s)$ 的开始状态，然后经过该NFA，再零次或多次从它的接受状态回到它的开始状态并重复上述过程。这些选项使得 $N(r)$ 可以接受$L(s)^1$、$L(s)^2$ 等集合中的所有串，因此 $N(r)$ 识别的所有串的集合就是 $L(s)^*$。

![](https://gitee.com/liuyun1995/yun-blog-image/raw/master/%E6%89%93%E9%80%A0%E6%AD%A3%E5%88%99%E5%BC%95%E6%93%8E%5B2%5D--Thompson%E7%AE%97%E6%B3%95/7.png)

#### 3 Thompson算法实现

```java
package com.liuyun.github.test;

import java.util.ArrayList;
import java.util.List;
import java.util.Stack;

public class Thompson {

    /**
     * 编译正则表达式
     * @param regex
     * @return
     */
    public static NFA compile(String regex){
        //验证输入字符串是否是正则表达式
        if (!validRegEx(regex)){
            System.out.println("Invalid Regular Expression Input.");
            return new NFA();
        }
        //操作符栈
        Stack<Character> operators = new Stack();
        //操作数栈
        Stack <NFA> operands = new Stack();
        //连接数栈
        Stack <NFA> concatStack = new Stack();
        //连接标识
        boolean ccflag = true;
        //当前操作符和字符
        char op, c;
        int paraCount = 0;
        NFA nfa1, nfa2;

        for (int i = 0; i < regex.length(); i++){
            c = regex.charAt(i);
            if (alphabet(c)){
                operands.push(new NFA(c));
                if (ccflag){
                    //用'.'替换连接符
                    operators.push('.');
                } else {
                    ccflag = true;
                }
            } else{
                if (c == ')'){
                    ccflag = true;
                    if (paraCount == 0){
                        System.out.println("Error: More end paranthesis than beginning paranthesis");
                        System.exit(1);
                    } else{
                        paraCount--;
                    }
                    //处理操作符栈直到遇到左括号
                    while (!operators.empty() && operators.peek() != '('){
                        op = operators.pop();
                        if (op == '.'){
                            nfa2 = operands.pop();
                            nfa1 = operands.pop();
                            operands.push(concat(nfa1, nfa2));
                        } else if (op == '|'){
                            nfa2 = operands.pop();
                            if(!operators.empty() && operators.peek() == '.'){
                                concatStack.push(operands.pop());
                                while (!operators.empty() && operators.peek() == '.'){
                                    concatStack.push(operands.pop());
                                    operators.pop();
                                }
                                nfa1 = concat(concatStack.pop(), concatStack.pop());
                                while (concatStack.size() > 0){
                                    nfa1 = concat(nfa1, concatStack.pop());
                                }
                            } else{
                                nfa1 = operands.pop();
                            }
                            operands.push(union(nfa1, nfa2));
                        }
                    }
                } else if (c == '*'){
                    operands.push(kleene(operands.pop()));
                    ccflag = true;
                } else if (c == '('){
                    operators.push(c);
                    paraCount++;
                    ccflag = false;
                } else if (c == '|'){
                    operators.push(c);
                    ccflag = false;
                }
            }
        }
        while (operators.size() > 0){
            if (operands.empty()){
                System.out.println("Error: imbalanace in operands and operators");
                System.exit(1);
            }
            op = operators.pop();
            if (op == '.'){
                nfa2 = operands.pop();
                nfa1 = operands.pop();
                operands.push(concat(nfa1, nfa2));
            } else if (op == '|'){
                nfa2 = operands.pop();
                if( !operators.empty() && operators.peek() == '.'){
                    concatStack.push(operands.pop());
                    while (!operators.empty() && operators.peek() == '.'){
                        concatStack.push(operands.pop());
                        operators.pop();
                    }
                    nfa1 = concat(concatStack.pop(), concatStack.pop());
                    while (concatStack.size() > 0){
                        nfa1 = concat(nfa1, concatStack.pop());
                    }
                } else{
                    nfa1 = operands.pop();
                }
                operands.push(union(nfa1, nfa2));
            }
        }
        return operands.pop();
    }

    /**
     * 判断输入字符是否是字母
     * @param c
     * @return
     */
    public static boolean alpha(char c){ return c >= 'a' && c <= 'z';}

    /**
     * 判断输入字符是否是字母或空串
     * @param c
     * @return
     */
    public static boolean alphabet(char c){ return alpha(c) || c == 'E';}

    /**
     * 判断是否是正则运算符
     * @param c
     * @return
     */
    public static boolean regexOperator(char c){
        return c == '(' || c == ')' || c == '*' || c == '|';
    }

    /**
     * 校验输入字符是否合法
     * @param c
     * @return
     */
    public static boolean validRegExChar(char c){
        return alphabet(c) || regexOperator(c);
    }

    /**
     * 验证是否是正则表达式
     * @param regex
     * @return
     */
    public static boolean validRegEx(String regex){
        if (regex.isEmpty()) {
            return false;
        }
        for (char c: regex.toCharArray()) {
            if (!validRegExChar(c)) {
                return false;
            }
        }
        return true;
    }

    /**
     * 并运算
     * @param n
     * @param m
     * @return
     */
    public static NFA union(NFA n, NFA m){
        //新NFA的状态数是原NFA状态数加2
        NFA result = new NFA(n.states.size() + m.states.size() + 2);

        //添加一条从0到1的空转换
        result.transitions.add(new Transition(0, 1, 'E'));

        //复制n的转移函数到新NFA中
        for (Transition t : n.transitions){
            result.transitions.add(new Transition(t.from + 1, t.to + 1, t.symbol));
        }

        //添加一条从n的最终状态到新NFA最终状态的空转换
        result.transitions.add(new Transition(n.states.size(), n.states.size() + m.states.size() + 1, 'E'));

        //添加一条从0到m的初始状态的空转换
        result.transitions.add(new Transition(0, n.states.size() + 1, 'E'));

        //复制m的转移函数到新NFA中
        for (Transition t : m.transitions){
            result.transitions.add(new Transition(t.from + n.states.size() + 1, t.to + n.states.size() + 1, t.symbol));
        }

        //添加一条从m的最终状态到新NFA最终状态的空转换
        result.transitions.add(new Transition(m.states.size() + n.states.size(), n.states.size() + m.states.size() + 1, 'E'));

        //设置新NFA的最终状态
        result.finalState = n.states.size() + m.states.size() + 1;
        return result;
    }


    /**
     * 连接运算
     * @param n
     * @param m
     * @return
     */
    public static NFA concat(NFA n, NFA m){
        //删除m的初始状态
        m.states.remove(0);

        //复制m的转移函数到n
        for (Transition t : m.transitions){
            n.transitions.add(new Transition(t.from + n.states.size() - 1, t.to + n.states.size() - 1, t.symbol));
        }

        //添加m的状态到n中
        for (Integer s : m.states){
            n.states.add(s + n.states.size() + 1);
        }

        //设置n的最终状态
        n.finalState = n.states.size() + m.states.size() - 2;
        return n;
    }

    /**
     * 柯林闭包
     * @param n
     * @return
     */
    public static NFA kleene(NFA n) {
        //新NFA的状态数是原NFA状态数加2
        NFA result = new NFA(n.states.size() + 2);

        //添加一条从0到1的空转换
        result.transitions.add(new Transition(0, 1, 'E'));

        //复制原NFA的转移函数到新NFA中
        for (Transition t : n.transitions){
            result.transitions.add(new Transition(t.from + 1, t.to + 1, t.symbol));
        }

        //添加一条从原NFA最终状态到新NFA最终状态的空转换
        result.transitions.add(new Transition(n.states.size(), n.states.size() + 1, 'E'));

        //添加一条从原NFA的最终状态到初始状态的空转换
        result.transitions.add(new Transition(n.states.size(), 1, 'E'));

        //添加一条从新NFA的初始状态到最终状态的空转换
        result.transitions.add(new Transition(0, n.states.size() + 1, 'E'));

        //设置新NFA的最终状态
        result.finalState = n.states.size() + 1;
        return result;
    }

    public static class NFA {
        /** 状态集合 */
        public List<Integer> states;
        /** 状态转移函数集合 */
        public List <Transition> transitions;
        /** 最终状态 */
        public int finalState;

        public NFA(){
            this.states = new ArrayList();
            this.transitions = new ArrayList();
            this.finalState = 0;
        }

        public NFA(int size){
            this.states = new ArrayList();
            this.transitions = new ArrayList();
            this.finalState = 0;
            this.setStateSize(size);
        }

        public NFA(char c){
            this.states = new ArrayList();
            this.transitions = new ArrayList();
            this.setStateSize(2);
            this.finalState = 1;
            this.transitions.add(new Transition(0, 1, c));
        }

        public void setStateSize(int size){
            for (int i = 0; i < size; i++) {
                this.states.add(i);
            }
        }

        public void display(){
            for (Transition t: transitions){
                System.out.println("("+ t.from +", "+ t.symbol + ", "+ t.to +")");
            }
        }
    }

    private static class Transition {
        /** 当前状态 */
        public int from;
        /** 目标状态 */
        public int to;
        /** 标号字符 */
        public char symbol;

        public Transition(int from, int to, char symbol){
            this.from = from;
            this.to = to;
            this.symbol = symbol;
        }
    }

    public static void main(String[] args) {
        NFA nfa = Thompson.compile("a(b|c)*");
        nfa.display();
    }

}
```

