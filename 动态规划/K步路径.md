# 原地方案数

https://leetcode.cn/problems/number-of-ways-to-stay-in-the-same-place-after-some-steps/description/

> *暴力枚举*
>
> ```c++
> int mod = 1e9 + 7, n;
> int _numWays(int cur,int step) { //_numWays返回经过step步后还在cur位置的方案数
>     if (step < 0 || cur < 0 || cur >= n) return 0;
>     if (step == 0) return cur == 0 ? 1 : 0;
>     int leftMove = _numWays(cur - 1, step - 1);
>     int rightMove = _numWays(cur + 1, step - 1);
>     int staticMove = _numWays(cur, step - 1);
>     return ((long long)leftMove + rightMove + staticMove) % mod;
> }
> int numWays(int steps, int arrLen) {
>     n = arrLen;
>     return _numWays(0, steps);
> }
> ```

> *DP* --- 递归函数的变量决定DP维度
>
> ```c++
> int numWays(int k, int n) {
>     int mod=1e9+7;
>     n=min(n,k+1);  //不加会超时 k的最大值远小于n的
>     auto dp=vector(k+1,vector(n,(long)0));
>     //dp(i,j)表示经过i步后在下标j处的方案数
>     dp[0][0]=1;
>     for(int i=1;i<=k;++i){
>         for(int j=1;j<n-1;++j)
>             dp[i][j]=(dp[i-1][j]+dp[i-1][j-1]+dp[i-1][j+1])%mod;
>         if(n>1) dp[i][0]=(dp[i-1][0]+dp[i-1][1])%mod; else dp[i][0]=dp[i-1][0];
>         if(n>1) dp[i][n-1]=(dp[i-1][n-1]+dp[i-1][n-2])%mod;
>     }
>     return dp[k][0];
> 
> }
> ```

> *滚动数组优化*
>
> ```c++
> int numWays(int k,int n){
>     int mod=1e9+7;
>     n=min(n,k+1);
>     vector<long> tmp(n);
>     vector<long> dp(n,0);dp[0]=1;
>     for(int i=1;i<=k;++i){
>         tmp.swap(dp);
>         for(int j=1;j<n-1;++j)
>             dp[j]=(tmp[j-1]+tmp[j]+tmp[j+1])%mod;
>         if(n>1) dp[0]=(tmp[0]+tmp[1])%mod; else dp[0]=tmp[0];
>         if(n>1) dp[n-1]=(tmp[n-1]+tmp[n-2])%mod; 
>     }
>     return dp[0];
> }
> ```

# 骑士拨号器

https://leetcode.cn/problems/knight-dialer/description/

```c++
vector<vector<int>> preJump={
    {4,6},{6,8},{7,9},{4,8},{3,9,0},{},{0,1,7},{2,6},{1,3},{2,4}
}; //*和#可以经过,但不可停留
int knightDialer(int n) {
    int mod=1e9+7,cnt=0;
    auto dp=vector(n,vector(10,(long)0));
    //dp(k,i)::=第k步落在数字i的方案数
    dp[0].assign(10,1); //起点

    for(int k=1;k<n;++k)
        for(int i=0;i<10;++i)
            for(int pre : preJump[i]) dp[k][i]=(dp[k][i]+dp[k-1][pre])%mod;
    for(int x : dp[n-1]) cnt=(cnt+x)%mod;
    return cnt;
}
```

# 出界路径数

https://leetcode.cn/problems/out-of-boundary-paths/description/

```c++
int findPaths(int m, int n, int maxMove, int startX, int startY) {
    if(maxMove==0) return 0; //exclude special case 
    long mod=1e9+7;
    auto dp=vector(maxMove,vector(m,vector(n,(long)0)));
    //dp(k,i,j)::=k步后到达(i,j)位置的方案数
    dp[0][startX][startY]=1;
    for(int k=1;k<maxMove;++k){
        for(int i=0;i<m;++i){
            for(int j=0;j<n;++j){
                if(i!=0) dp[k][i][j]=(dp[k][i][j]+dp[k-1][i-1][j])%mod;
                if(i!=m-1) dp[k][i][j]=(dp[k][i][j]+dp[k-1][i+1][j])%mod;
                if(j!=0) dp[k][i][j]=(dp[k][i][j]+dp[k-1][i][j-1])%mod;
                if(j!=n-1) dp[k][i][j]=(dp[k][i][j]+dp[k-1][i][j+1])%mod;
            }
        }
    }
    long count=0;
    for (int k = 0; k < maxMove; ++k) {
        for (int i = 0; i < m; ++i) {
            count = (count + dp[k][i][0]) % mod;
            count = (count + dp[k][i][n - 1]) % mod;
        }
        for (int j = 0; j < n; ++j) {
            count = (count + dp[k][0][j]) % mod;
            count = (count + dp[k][m - 1][j]) % mod;
        }
    }
    return count;
}
```

