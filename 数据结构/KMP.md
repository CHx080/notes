https://leetcode.cn/problems/find-the-index-of-the-first-occurrence-in-a-string/

**KMP算法的本质是利用前缀函数求解的,经典KMP不仅需要掌握公共前后缀函数的算法,还需要记忆基于前缀函数的检索方式显得稍嫌麻烦**

**经过优化的KMP算法可以直接利用公共前后缀数组计算结果**

<img src="https://pic.leetcode.cn/1740296473-gbYwUm-7c0fe30537f16f85be2ab7162a2d2be.jpg" alt="img" style="zoom: 50%;" />

假如把needle和haystack进行拼接(needle在前),我们对这个新字符串进行计算最长公共前后缀长度,在计算的过程中存在最长公共前后缀长度等于needle的长度,那么就说明haystack中一定存在needle

<img src="https://pic.leetcode.cn/1740297024-OUnbsl-5add06fffcbdea409cd40dbe7085a86.jpg" alt="img" style="zoom:50%;" />

*C++ code*

```c++
int strStr(string haystack, string needle) {
    string str=needle+' '+haystack;
    int n=str.length(),m=needle.length();
    vector<int> prefix(n,0);
    `	//prefix(i)::=str 0~i 的最长公共前后缀长度
    for(int i=1;i<n;++i){
        int len=prefix[i-1];
        while(len>0 and str[len]!=str[i]) 
            len=prefix[len-1]; //☆ 关键点
        if(str[len]==str[i]) prefix[i]=len+1;
        if(prefix[i]==m) return i-(m<<1);
    }
    return -1;
}
```

