---
title: '[转载]欧拉路径和Hierholzer算法'
date: 2020-09-24 12:06:42
tags: [Algorithm]
categories:
cover: https://upload-images.jianshu.io/upload_images/7749027-8ea3598caabe688a.PNG?imageMogr2/auto-orient/strip|imageView2/2/w/568/format/webp
---
<meta name="referrer" content="no-referrer" />

1. 欧拉回路和欧拉路径
2. Hierholzer算法求解欧拉回路和欧拉路径
3. 欧拉回路的应用：LeetCode753破解密码箱
4. 德布鲁因序列

### 欧拉图

问题来源：1736年瑞士数学家欧拉发表论文讨论哥尼斯堡七桥问题。欧拉图问题也是图论研究的起源。
 **基本概念：**
 圈：任选图中一个顶点为起点，沿着不重复的边，经过不重复的顶点为途径，之后又回到起点的闭合途径称为圈。
 欧拉路径：通过图中所有边一次且仅一次遍历所有顶点的路径称为欧拉(Euler)路径；
 欧拉回路：通过图中所有边一次且仅一次行遍所有顶点的回路称为欧拉回路；
 欧拉图：具有欧拉回路的图称为欧拉图；
 半欧拉图：有欧拉路径但没有欧拉回路的图称为半欧拉图。
 **欧拉图与半欧拉图的判定：**

1. G是欧拉图![\Leftrightarrow](https://math.jianshu.com/math?formula=%5CLeftrightarrow)G中所有顶点的度均为偶数![\Leftrightarrow](https://math.jianshu.com/math?formula=%5CLeftrightarrow)G是若干个边不重的圈的并。
2. G是半欧拉图![\Leftrightarrow](https://math.jianshu.com/math?formula=%5CLeftrightarrow)G中恰有两个奇数度顶点。

注意：以上判定是基于无向图，有向欧拉图的判定与此类似，这里先略去，在讨论有向图时会补充。另外，研究无向欧拉图，可以有平行边，这里也不考虑。
 由于欧拉图有严格的拓扑学性质和简明的充分必要条件，所以欧拉图的判定和欧拉回路的求解要比哈密顿图的判定和哈密顿回路的求解简单的多。
 **欧拉图判定算法**
 首先判定是否连通，然后遍历每个顶点检查顶点的度是否是偶数即可。



```java
public boolean hasEulerLoop(){
        // 欧拉回路存在的前提是连通，首先判断连通性
        CC cc = new CC(G);
        if(cc.count() > 1) return false;
        for(int v = 0; v < G.V(); v ++)
            if(G.degree(v) % 2 == 1)
                return false;
        return true;
    }
```

### 欧拉图中求解欧拉回路

**方法1：回溯法**
 遍历所有边，每遍历一个边则删除该边继续遍历，如果中间过程还没有遍历所有边就无法继续遍历了，则往前回溯继续遍历。该算法时间复杂度是指数级别。

**方法2：弗罗莱(Fluery)算法**
 设G是一无向欧拉图，Fluery算法求解一条欧拉回路算法如下：

- (1) 任取![v_0∈V(G)](https://math.jianshu.com/math?formula=v_0%E2%88%88V(G))，令![P_0=v_0](https://math.jianshu.com/math?formula=P_0%3Dv_0).
- (2) 设![P_i=v_0e_1v_1e_2…e_iv_i](https://math.jianshu.com/math?formula=P_i%3Dv_0e_1v_1e_2%E2%80%A6e_iv_i)已经行遍，按下面方法来从![E(G)-\{e_1,e_2,…,e_i\}](https://math.jianshu.com/math?formula=E(G)-%5C%7Be_1%2Ce_2%2C%E2%80%A6%2Ce_i%5C%7D)中选取![e_{i+1}](https://math.jianshu.com/math?formula=e_%7Bi%2B1%7D)
   （a）![e_{i+1}](https://math.jianshu.com/math?formula=e_%7Bi%2B1%7D)与![v_i](https://math.jianshu.com/math?formula=v_i)相关联； 
   （b）除非无别的边可供行遍，否则![e_{i+1}](https://math.jianshu.com/math?formula=e_%7Bi%2B1%7D)不应该为![G_i=G-\{e_1,e_2,…,e_i\}](https://math.jianshu.com/math?formula=G_i%3DG-%5C%7Be_1%2Ce_2%2C%E2%80%A6%2Ce_i%5C%7D)中的桥。
- (3) 当（2）不能再进行时，算法停止。

当算法停止时所得简单回路![P_m=v_0e_1v_1e_2…e_mv_m(v_m=v_0)](https://math.jianshu.com/math?formula=P_m%3Dv_0e_1v_1e_2%E2%80%A6e_mv_m(v_m%3Dv_0))为![G](https://math.jianshu.com/math?formula=G)中一条欧拉回路。
 Fluery算法的原则是当来到某个顶点有多条边可以选择的时候，除非无路可选否则不走桥(删除走过的边之后)。这个算法的时间复杂度是![O(V+E)^2](https://math.jianshu.com/math?formula=O(V%2BE)%5E2)级别的。

**方法3：Hierholzer算法(插入回路法)**
 该算法的思想是一步步构造出回路。由欧拉图的充要条件：G是欧拉图![\Leftrightarrow](https://math.jianshu.com/math?formula=%5CLeftrightarrow)G是若干个边不重的圈(环)的并，我们可以先找到一个环，而剩下的边一定还存在环，且这两个部分必有公共点，从而可以形成更大的环，这样直到包括所有边，即可找到欧拉回路。该算法时间复杂度为![O(V+E)](https://math.jianshu.com/math?formula=O(V%2BE))，非常高效。
 设置两个栈，curPath和loop。算法过程：
 （1）选择任一顶点为起点，入栈curPath，深度搜索访问顶点，将经过的边都删除，经过的顶点入栈curPath。
 （2）如果当前顶点没有相邻边，则将该顶点从curPath出栈到loop。
 （3）loop栈中的顶点出栈顺序，就是从起点出发的欧拉回路。
 (不过其实loop的入栈顺序也是欧拉回路，刚好和我们遍历的欧拉回路方向反向，所以loop也可以设计为ArrayList或队列。)



```cpp
import java.util.ArrayList;
import java.util.Stack;

public class EulerLoop {
    private Graph G;
    public EulerLoop(Graph G){
        this.G = G;
    }
    public boolean hasEulerLoop(){
        // 是否存在欧拉回路
        CC cc = new CC(G);
        if(cc.count() > 1) return false;// 欧拉回路存在的前提是连通，首先判断连通性
        for(int v = 0; v < G.V(); v ++)
            if(G.degree(v) % 2 == 1)
                return false;
        return true;
    }
    public ArrayList<Integer> result(){
        // 返回欧拉回路结果
        ArrayList<Integer> res = new ArrayList<>();// 充当Loop栈
        if(!hasEulerLoop()) return res;
        Graph g = (Graph) G.clone();// 用 G 的副本 g 寻找欧拉回路
        // 删除 g 的边不会影响 G
        Stack<Integer> stack = new Stack<>(); // curPath 栈
        int curv = 0;
        stack.push(curv);
        while (!stack.isEmpty()){
            if(g.degree(curv) != 0){
                // 度不为0说明当前顶点连的还有边，也就是还有路可走
                stack.push(curv);
                int w = g.adj(curv).iterator().next(); // 可迭代列表的第一个元素,即取g的任意邻点
                g.removeEdge(curv, w);
                curv = w;
            }else {
                // curv 到不了其它顶点，则已经找到一个环
                res.add(curv);
                curv = stack.pop();
            }
        }
        return res;
    }
    public static void main(String args[]){
        Graph g = new Graph("g.txt");
        EulerLoop el = new EulerLoop(g);
        System.out.println(el.result());
    }
}
```

### 半欧拉图求解欧拉路径

半欧拉图求解欧拉路径同样基于Hierholzer算法，由半欧拉图的充分必要条件：G是半欧拉图![\Leftrightarrow](https://math.jianshu.com/math?formula=%5CLeftrightarrow)G中恰有两个奇数度顶点。我们可以先找到这两个奇数度顶点，从其中任意一个开始，按照寻找欧拉回路相同的步骤，当遍历完所有边得到的就是一个欧拉路径。



```cpp
import java.util.ArrayList;
import java.util.Stack;

public class EulerPath {
    private Graph G;
    ArrayList<Integer> startAndEnd; // start end,欧拉路径的起点和终点
    boolean isEuler = false, isHalfEuler = false;
    public EulerPath(Graph G){
        this.G = G;
        startAndEnd = new ArrayList<>();
    }
    public void hasEulerPath(){
        // 是否存在欧拉路径
        CC cc = new CC(G);
        if(cc.count() > 1) {
            isEuler = false;// 首先判断连通性
            isHalfEuler = false;
        }
        for(int v = 0; v < G.V(); v ++)
            if(G.degree(v) % 2 == 1)
                startAndEnd.add(v);
        if(startAndEnd.size() == 0){
            isEuler = true; isHalfEuler = true;
        }else if(startAndEnd.size() == 2){
            isHalfEuler = true;
        }
    }
    public ArrayList<Integer> result(){
        // 返回欧拉路径结果
        ArrayList<Integer> res = new ArrayList<>();// 充当Path栈
        hasEulerPath();
        if(!isHalfEuler) return res;
        Graph g = (Graph) G.clone();
        // 删除 g 的边不会影响 G
        Stack<Integer> stack = new Stack<>(); // curPath 栈
        int curv = 0;
        if(startAndEnd.size() == 2)
            curv = startAndEnd.get(0);
        stack.push(curv);
        while (!stack.isEmpty()){
            if(g.degree(curv) != 0){
                // 度不为0说明当前顶点连的还有边，也就是还有路可走
                stack.push(curv);
                int w = g.adj(curv).iterator().next(); // 可迭代列表的第一个元素,即取g的任意邻点
                g.removeEdge(curv, w);
                curv = w;
            }else {
                // curv 到不了其它顶点，则已经找到一个环
                res.add(curv);
                curv = stack.pop();
            }
        }
        return res;
    }
    public static void main(String args[]){
        Graph g = new Graph("g.txt");
        EulerPath ep = new EulerPath(g);
        System.out.println(ep.result());
    }
}
```

### 欧拉回路的应用

**LeetCode753破解保险箱**

![img](https:////upload-images.jianshu.io/upload_images/7749027-91573c0f8df8399e.PNG?imageMogr2/auto-orient/strip|imageView2/2/w/749/format/webp)

LeetCode753

对题意的说明：
 题目的含义是现在有一个有记忆功能的密码箱，其密码有`n`位，每一位是`[0,k)`之间的整数，即一个`k`进制数，在利用密码箱记忆特性的基础下如何找到最短的串，使得该串的相邻n位包含所有的密码组合。
 我们并不知道密码是多少，想要开箱子，只能去试，密码空间共有`k^n`个密码，每个密码`n`个字符。以`n=3,k=2`为例：所有可能的密码为`000,001,010,011,100,101,110,111`，我们可以输入`000 001 010 011 000 101 110 111`，由于这个序列的邻3位包含了所有可能密码，所以可以打开密码箱。但其实输入`010011101`也可以打开密码箱，因为在密码箱的记忆特性下，末3位同样遍历了密码空间。现在问题是如何找到最短的这样的串。问题抽象出来就是：
 **如何构造一个长度为n的k进制序列，使得所有长度为n的序列都在它的子序列中出现并且仅出现一次。**
 这和组合数学中的德布鲁因序列(De Bruijn sequence)几乎一模一样，德布鲁因序列B(k, n)，是k元素构成的循环序列。所有长度为n的k元素构成序列都在它的子序列（以环状形式）中，出现并且仅出现一次。

![img](https:////upload-images.jianshu.io/upload_images/7749027-ca355956a5a26d5b.PNG?imageMogr2/auto-orient/strip|imageView2/2/w/771/format/webp)

德布鲁因序列_维基百科

现在来求解这个题，构造一个有向图，具体求解描述如下：

- 这个图有`k^(n-1)`个顶点，每个顶点是一个`n-1`长度的`k`进制序列；
- 每个顶点有`k`条入边和`k`条出边，`k`条边分别代表数字`0,1,2,...,k-1`；
- 显然这个图是一个欧拉图，找到图中的一条欧拉回路，将回路上顶点和边的数字按遍历顺序写出来就是本题的解。

为什么要这样来构造图呢，首先这个欧拉回路恰好对应De Bruijn序列，De Bruijn序列最后的长度是`k^n`，这正是密码空间中所有可能密码的数量，构造这样一个有向图后，我们让每个顶点沿着一条边到另一个顶点后转移到这个顶点表示的状态，以`k=2,n=3`为例：

![img](https:////upload-images.jianshu.io/upload_images/7749027-8ea3598caabe688a.PNG?imageMogr2/auto-orient/strip|imageView2/2/w/568/format/webp)

状态转移

这样每个顶点加上与它相连的一条出边恰好是一个密码空间中的密码，所有可能的这样的顶点和边的组合个数是`k^(n-1) * k = k^n`，也就是是密码空间中所有可能密码的数量，如果能找到一条遍历到所有的边一次且仅一次的回路，就意味着这个序列出现过密码空间中的所有密码一次且仅一次，不可能有比它更短的回路了，所以我们求这个图的一条欧拉回路就好了。



```dart
import java.util.Collections;
import java.util.TreeSet;

class Solution {
    TreeSet<String> visited;
    StringBuilder res;
    public String crackSafe(int n, int k) {
        if(n == 1 && k == 1) return "0";
        visited = new TreeSet<>();
        res = new StringBuilder();
        //  从顶点 00..0 开始
        String start = String.join("", Collections.nCopies(n-1, "0"));;
        findEuler(start, k);

        res.append(start); // 回路添加最后的end顶点，end 就是 start
        return res.toString(); // return new String(res);
    }
    public void findEuler(String curv, int k){

        for(int i = 0; i < k; i ++){
            // 往顶点的 k 条出边检查，顶点加一条出边就是一种密码可能
            String nextv = curv + i;
            if(!visited.contains(nextv)){
                visited.add(nextv);
                findEuler(nextv.substring(1), k);
                res.append(i);
            }
        }
    }
}
```

