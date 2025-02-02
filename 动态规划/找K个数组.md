# 找K个数组

j=1需要特殊处理，j=0仅有凑数作用，规避使用。无法找到一个值填充j=0这一列使得不对结果造成影响。

## 最大平均值和

https://leetcode.cn/problems/largest-sum-of-averages/description/

```c++
double largestSumOfAverages(vector<int>& nums, int k) {
    int n = nums.size();
    vector<double> preSum(n + 1);
    for (int i = 1; i <= n; ++i) {
        preSum[i] = preSum[i - 1] + nums[i - 1];
    }
    vector<vector<double>> dp(n + 1, vector<double>(k + 1));
    for (int i = 1; i <= n; ++i) {
        //dp[i][1] = preSum[i] / i;
        for (int j = 1; j <= i and j <= k; ++j) {
            for (int x = j - 1; x < i; ++x) {
                double v = dp[x][j - 1] + (preSum[i] - preSum[x]) / (i - x);
                dp[i][j] = max(dp[i][j], v);
            }
        }
    }
    return dp[n][k];
}
```

## 分割数组的最大值

https://leetcode.cn/problems/split-array-largest-sum/description/

```c++
int splitArray(vector<int>& nums, int k) {
    int n=nums.size();
    vector<int> preSum(n+1);
    for(int i=1;i<=n;++i) preSum[i]=preSum[i-1]+nums[i-1];
    vector<vector<int>> dp(n+1,vector<int>(k+1,0x3f3f3f3f));
    for(int i=1;i<=n;++i){
        dp[i][1]=preSum[i];
        for(int j=2;j<=i and j<=k;++j){
            for(int x=j-1;x<i;++x){
                int v=max(dp[x][j-1],preSum[i]-preSum[x]);
                dp[i][j]=min(dp[i][j],v);
            }
        }
    }
    return dp[n][k];
}
```

## 分割回文串

https://leetcode.cn/problems/palindrome-partitioning-iii/description/

```c++
int palindromePartition(string s, int k) {
    int n=s.length();
    vector<vector<int>> g(n+1,vector<int>(n+1)); //g(i,j)::= i~j变成回文串的修改次数
    for(int i=n;i>=1;--i){
        for(int j=i;j<=n;++j){
            if(i==j) continue;
            else if(i+1==j and s[i-1]!=s[j-1])  g[i][j]=1;
            else{
                g[i][j]=g[i+1][j-1];
                if(s[i-1]!=s[j-1]) g[i][j]++;
            }
        }
    }
    vector<vector<int>> f(n+1,vector<int>(k+1,0x3f3f3f3f)); 
    //f(i,j)::= 前i个字符拆为j个回文串最少修改次数
    for(int i=1;i<=n;++i){
        f[i][1]=g[1][i];
        for(int j=2;j<=i and j<=k;++j){
            for(int x=j;x<=i;++x){
                f[i][j]=min(f[i][j],f[x-1][j-1]+g[x][i]);
            }
        }
    }
    return f[n][k];
}
```

