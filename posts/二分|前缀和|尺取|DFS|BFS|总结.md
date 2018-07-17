# 总结

二分法、前缀和、尺取法、DFS、BFS 五种算法的分别总结和相应题目的解决。

### 二分法
二分法就是在有序序列中快速查找出某个值的一种算法，但是其实有一些情况可以转换为依靠二分法的方式来解决。比较常见的就是求最大值的最小值（或最小值的最大值）。这种题的思路就是，找出可行解的上界和下界，之后通过二分法（迭代的，否则易爆栈）配合check函数来进行二分查找解空间。check函数根据题目的不同而不同。 <br>
例:http://poj.org/problem?id=3273 <br>
[My Solution](https://github.com/Humbertzhang/DataStructure/blob/master/acm/set4/A.cpp)<br>

这个题意思就是说，给你一组序列，让你分成给定数目的块，问你块的和的最小值最大是多少.
使用二分法来解决。
首先确定每一块的上界和下界（这里上界下届不要求一定准确，只要是包含解可能在的区间的就可以）.这里取上界为每一个输入加起来的总和，下届为所有块中的最大值（因为不可能 分割开的块的和的最小值的最大值 比 所有序列中的最大值小 ）。
之后二分的check函数如下:
```C
int check(int maxb)
{
	int sum = 0;
	int cnt = 0;
	for(int i = 0; i < n; i++){
		if(sum + money[i] > maxb){
			sum = 0;
			cnt += 1;
			sum += money[i];
		} else {
			sum += money[i];
		}
	}
	cnt += 1;	
	return cnt <= m; 
}
```
解释: 从序列开始处分块，若该块的和大于等于maxb(假设为最小值的最大值),则开始下一块的分割，若最终发现分块数为<=所要求的数目，则说明是一个有可能的解，返回true, 否则返回false. 而如果是true则在主函数中就按照二分法的方式来进一步增大猜测值（maxb），依次来达到逼近最大值的目的。
主函数关键代码如下
```C
int l = max, r = sum;
while(r-l > 1){   // r-l>1目的是因为只需要逼近整数即可.
		int mid = (r+l)/2;
		if(check(mid)){
			r = mid;
		} else {
			l = mid;
		}
}
cout << r << endl;  // 之所以此处打印r是因为其为最后一个检测通过的数值.
```


### 前缀和
即使用一个数组来记录到达此处的某个值pre[i] , 而下一个值pre[i-1]又可以通过pre[i] + 某个数值来得到。常用于求某一段的和等问题。可以在一开始通过O(n)的算法计算出pre数组，之后以O(1)的时间复杂度来计算每一次查询的值.

例子:http://acm.csu.edu.cn/csuoj/problemset/problem?pid=1642 <br>
已知两个正整数a和b，求在a与b之间（包含a和b）的所有整数的十进制表示中1出现的次数。
此题就是通过一个用于记录到第i个数时已经有了多少1的数组，来处理之后出现的大量查询.<br>
[My Solution](https://github.com/Humbertzhang/DataStructure/blob/master/acm/day8/B.cpp) <br>

### 尺取法
也叫做双指针法，在leetcode上其实碰见过好几次.
主要的思想就是利用两个指针(其实就是数组下标)根据情况改变指针位置，来在O(n)的时间复杂度内遍历序列.

例:https://vjudge.net/contest/237234#problem/A <br>
[My Solution](https://github.com/Humbertzhang/DataStructure/blob/master/acm/day8/A.cpp) <br>
给出一个字符串，求该字符串的一个子串s，s包含A-Z中的全部字母，并且s是所有符合条件的子串中最短的，输出s的长度。如果给出的字符串中并不包括A-Z中的全部字母，则输出No Solution。

关键代码
```C
int check(int start, int end){   // 返回 start 到 end 包含的不同字母个数
char a[26] = {0};
for(int i = start; i <= end; i++){
  a[s[i]-'A'] = 1;
}
int count = 0;
for(int i = 0; i < 26; i++){
  count += a[i];
}
return count;
}

for(i = 0, j = 0; i < slen && j < slen; ) {
  int checked = check(i, j); 
	if(checked < 26){           // 不足26字母，需要扩展，故 j+=1
    j += 1;
	}  else {                   // 等于26字母，记录长度，i += 1
    if(j-i+1 < minlen){
  	  minlen = j-i+1;
		}
    i += 1;
	}
}
if(minlen != slen + 1){
  cout << minlen << endl;
} else {
	cout << "No Solution" << endl;
}
```


### DFS

深度优先搜索，一般以递归的方式实现. <br>
DFS常用于给出一个图，求图中分割开的区域的个数，或者在图中求某种类型块的个数，更抽象的则在解空间树中以深度优先搜索的方式（即为回溯法）找到相应的解. <br>
dfs伪代码
```C
void dfs()
{
	//1,边界处理	
	if(到达边界){
		code here
		return;	
	}
	//2,扩展
	else{
		for(枚举可能的分支){
			dfs()	 // 递归	
		}		
	}
}
```

深度优先遍历解空间树以N皇后为例
[N皇后](http://acm.hdu.edu.cn/showproblem.php?pid=2553) <br>
[My Solution](https://github.com/Humbertzhang/DataStructure/blob/master/acm/day9/A.cpp)

DFS在网格图中的使用:
[Red and Black](http://poj.org/problem?id=1979)
关键代码及解释
```C
/* 因为递归的缘故，大量数据被开在全局变量中了，这也是ACM的一个特点，就是为了避免栈空间的限制或者为了减少传参数，而把数据写在全局变量中，，毕竟不需要维护... */
char graph[25][25];
bool vis[25][25];
int w, h;
int count = 0;

int dx[4] = {1, 0, -1, 0};
int dy[4] = {0, 1, 0, -1};


void dfs(int x0, int y0)
{
	if(vis[x0][y0]){    // 访问过，则直接return 
		return ;
	}
	vis[x0][y0] = 1;   // 标识访问
	if(graph[x0][y0] == '.'){  
		count += 1;
	}
	for(int i = 0; i < 4; i++){
		int x = x0 + dx[i];         // 采取这种方式来简单地实现上下左右行走的策略
		int y = y0 + dy[i];
		if(x > 0 && x <= h && y > 0 && y <= w && graph[x][y] == '.'){
			dfs(x, y);              // 递归访问下一个
		}
	}
	return;
}

```


### BFS 
BFS算法为广度优先搜索，最常用的地方就是来求最短路径。但是除了明确的给出一个图还有可能是这种形式:
```
给一个整数x， 可对x + a, x * b, x/2等， 问你几步可以把x变成y
把x看做一个节点，把他操作的三个当作跟他相邻的点，因此可以用BFS来查找
```

BFS伪代码
```C
int bfs(start, end){
    初始节点入队
    访问记录
    
    while(!q.empty()) {
        node = q.pop()
        枚举所有可能节点{
            if(!到达边界) {
                节点入队
                visited = true;
                else code
            }
            if (到达终点) {
                return;
            }
        }
        
    }
}
```

网格图上寻求最短路径的最经典的操作就是[走迷宫](https://vjudge.net/contest/237513#problem/A) <br>
[My solution](https://github.com/Humbertzhang/DataStructure/blob/master/acm/day10/A.cpp)

而像抽象的以此题为例. <br>
[Catch That Cow](http://poj.org/problem?id=3278)  <br>
[My solution](https://github.com/Humbertzhang/DataStructure/blob/master/acm/day10/B.cpp) <br>
在一个一维空间中，你有X + 1 , X - 1和 X * 2 三种操作，问你从 X1 -> X2 的最短操作步数是多少.

关键代码


```C
int bfs(int n, int k)  // n 为起始点， k为结束点 
{
    q.push(n);          // 初始节点入队
    step[n] = 0;        // 记录已用步数
    visited[n] = 1;     // 记录访问.
    while(!q.empty()){  // 广度优先遍历
        int pos = q.front();
        q.pop();
        int stepnow = step[pos];    
        /*广度优先 入队*/       // 所有可能节点入队
        for(int i = 0; i < 3; i++){
            int np = 0;
            if(i == 0){         // 三种可能操作的下一节点计算
                np = pos-1;
            } else if(i == 1){
                np = pos+1;
            } else {
                np = pos * 2;
            }
            if(np < 0 || np > 100000){      // 超出范围处理
                continue;
            }
            if(!visited[np]){               // 边界处理
                q.push(np);                 // 入队
                visited[np] = 1;            // 记录访问
                step[np] = stepnow + 1;     // 步数记录
            }
            if(np == k){return step[np];}   // 到达终点返回
        }
    }
}
```
