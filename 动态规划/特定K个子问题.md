# 格雷编码

https://leetcode.cn/problems/gray-code/description/

<img src="https://chx-typora.oss-cn-hangzhou.aliyuncs.com/typora/image-20250226142016555.png" alt="image-20250226142016555" style="zoom:50%;" />

```c++
vector<int> grayCode(int n) {
    auto dp=vector(n,vector<int>());
    //dp(i)::=i+1位格雷码排列
    dp[0]={0,1};
    for(int i=1;i<n;++i){
        dp[i].resize(dp[i-1].size()<<1); //n位格雷码有2^n个码值
        int m=dp[i].size();
        bool tag=true; //tag为T则先补0后补1
        for(int j=0;j<m;j+=2){
            dp[i][j]=dp[i-1][j/2]<<1;dp[i][j+1]=(dp[i-1][j/2]<<1)+1;
            if(!tag) {++dp[i][j];--dp[i][j+1];}
            tag=!tag;
        }
    }
    return dp[n-1];
}
```

# 循环码排列

https://leetcode.cn/problems/circular-permutation-in-binary-representation/description/

```c++
vector<int> circularPermutation(int n, int start) {
    auto dp=vector(n,vector<int>());
    //dp(i)::=i+1位格雷码排列
    dp[0]={0,1};
    for(int i=1;i<n;++i){
        dp[i].resize(dp[i-1].size()<<1); //n位格雷码有2^n个码值
        int m=dp[i].size();
        bool tag=true; //tag为T则先补0后补1
        for(int j=0;j<m;j+=2){
            dp[i][j]=dp[i-1][j/2]<<1;dp[i][j+1]=(dp[i-1][j/2]<<1)+1;
            if(!tag) {++dp[i][j];--dp[i][j+1];}
            tag=!tag;
        }
    }
    vector<int> res(dp[n-1].size());
    auto it=find(dp[n-1].begin(),dp[n-1].end(),start);
    copy(it,dp[n-1].end(),res.begin());
    copy(dp[n-1].begin(),it,res.begin()+(dp[n-1].end()-it)); //把start及其之后的数字全部移至前面即可
    return res;
}
```

# 鸡蛋掉落

https://leetcode.cn/problems/super-egg-drop/description/

```c++
int superEggDrop(int k, int n) {
    //f(t,k)::=t次操作k个鸡蛋能确定的最大楼层
    if (n == 1) return 1;
    vector<vector<int>> f(n + 1, vector<int>(k + 1));
    for (int i = 1; i <= k; ++i) f[1][i] = 1;
    int ans = -1;
    for (int i = 2; i <= n; ++i) {
        for (int j = 1; j <= k; ++j) 
            f[i][j] = 1 + f[i - 1][j - 1] + f[i - 1][j];
        if (f[i][k] >= n) {ans = i;break;}
    }
    return ans;
}
```

# 斐波那契数

https://leetcode.cn/problems/fibonacci-number/description/

```c++
int fib(int n) {
    if(n<=1) return n;
    int f0=0,f1=1;
    for(int i=2;i<=n;++i){
        int t=f0;
        f0=f1;f1+=t;
    }
    return f1;
}
```

# 爬楼梯

https://leetcode.cn/problems/climbing-stairs/description/

```c++
int climbStairs(int n) {
    if(n<=2) return n;
    int f1=1,f2=2;
    for(int i=3;i<=n;++i){
        int t=f1;
        f1=f2;f2+=t;
    }
    return f2;
}
```

# 泰波那契数

https://leetcode.cn/problems/n-th-tribonacci-number/description/

```c++
int tribonacci(int n) {
    if(n==0) return 0;
    if(n==1 || n==2) return 1;
    int a=0,b=1,c=1;
    for(int i=3;i<=n;++i)
    {
        int t1=b;
        int t2=c;
        c=c+b+a;
        b=t2;
        a=t1;
    }
    return c;
}
```

# 比特位计数

https://leetcode.cn/problems/counting-bits/description/

```c++
vector<int> countBits(int n) {
    vector<int> f(n+1);
    for(int i=1;i<=n;++i){
        if((i-1)%2==0) f[i]=f[i-1]+1; //i-1末位为0~i-1为偶数
        else{
            int t=i-1,bitCnt=0; //找到i-1不为1的最低位
            while(t%2){
                t=t>>1;++bitCnt;
            }
            f[i]=f[i-1]-bitCnt+1;
        }
    }
    return f;
}
```

# 丑数II

https://leetcode.cn/problems/ugly-number-ii/description/

```c++
int nthUglyNumber(int n) {
    //每一个大于1的自然数可以被唯一地分解为质数乘积(第二数学归纳法可证)
    //固2,3,5相乘所得到数其质因子只有2,3,5
    vector<int> f(n+1);f[1]=1;
    int p2=1,p3=1,p5=1;
    for(int i=2;i<=n;++i){
        int num2=f[p2]*2,num3=f[p3]*3,num5=f[p5]*5;
        int t=min({num2,num3,num5});
        if(t==num2) ++p2;
        if(t==num3) ++p3;
        if(t==num5) ++p5;
        f[i]=t;
    }
    return f[n];
}
```

# 超级丑数

https://leetcode.cn/problems/super-ugly-number/description/

与丑数II同理*

```c++
int nthSuperUglyNumber(int n, vector<int>& primes) {
    int m=primes.size();
    vector<unsigned long> f(n+1),p(m,1),t(m);f[1]=1;
    for(int i=2;i<=n;++i){
        unsigned long M=INT_MAX;
        for(int j=0;j<m;++j) {t[j]=f[p[j]]*primes[j];M=min(M,t[j]);}
        for(int j=0;j<m;++j) if(M==t[j]) ++p[j];
        f[i]=M;
    }
    return f[n];
}
```

