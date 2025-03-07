==**DP数组定义为长度为i且末尾状态为j的最优值**==(DP[i]\[j])

## 掷骰子模拟

https://leetcode.cn/problems/dice-roll-simulation/description/

*内存超限*

```c++
int dieSimulator(int n, vector<int>& rollMax) {
    long mod=1e9+7;
    auto dp=vector(n,vector(6,vector(n,(long)0)));
    //dp(i,j,k)::=投i+1次,最后k+1次结果为j+1的序列数
    dp[0][0][0]=dp[0][1][0]=dp[0][2][0]=dp[0][3][0]=dp[0][4][0]=dp[0][5][0]=1;
    for(int i=1;i<n;++i){
        for(int j=0;j<6;++j){
            //k>1
            for(int k=1;k<=i && k+1<=rollMax[j];++k)
                dp[i][j][k]=dp[i-1][j][k-1];
            //k=1
            for(int x=0;x<6;++x){
                if(x==j) continue;
                dp[i][j][0]=(dp[i][j][0]+
                             +accumulate(dp[i-1][x].begin(),dp[i-1][x].end(),(long)0))%mod;
            }
        }
    }
    long cnt=0;
    for(int j=0;j<6;++j) 
        cnt=(cnt+accumulate(dp[n-1][j].begin(),dp[n-1][j].end(),(long)0))%mod;
    return cnt;
}
```

*滚动数组*

```c++
int dieSimulator(int n, vector<int>& rollMax) {
    long mod=1e9+7;
    auto dp=vector(6,vector(n,(long)0));
    auto temp=dp;
    //dp(j,k)::=投i+1次,最后k+1次结果为j+1的序列数
    dp[0][0]=dp[1][0]=dp[2][0]=dp[3][0]=dp[4][0]=dp[5][0]=1;
    for(int i=1;i<n;++i){
        for(auto& v:temp) for(auto& x:v) x=0;
        for(int j=0;j<6;++j){
            //k=1
            for(int x=0;x<6;++x){
                if(x==j) continue;
                temp[j][0]=(temp[j][0]+
                            +accumulate(dp[x].begin(),dp[x].end(),(long)0))%mod;
            }
            //k>1
            for(int k=1;k<=i && k+1<=rollMax[j];++k)
                temp[j][k]=dp[j][k-1];
        }
        temp.swap(dp);
    }
    long cnt=0;
    for(int j=0;j<6;++j) 
        cnt=(cnt+accumulate(dp[j].begin(),dp[j].end(),(long)0))%mod;
    return cnt;
}
```

## 统计元音序列数目

https://leetcode.cn/problems/count-vowels-permutation/description/

```c++
int countVowelPermutation(int n) {
    //a,e,i,o,u~0,1,2,3,4
    //dp(i,j)::=长度为i+1且首个字母是(j)的序列数(从后往前拼单词)
    auto dp=vector(n,vector(5,(long)1));long mod=1e9+7;
    for(int i=1;i<n;++i){
        dp[i][0]=dp[i-1][1];
        dp[i][1]=(dp[i-1][0]+dp[i-1][2])%mod;
        dp[i][2]=(dp[i-1][0]+dp[i-1][1]+dp[i-1][3]+dp[i-1][4])%mod;
        dp[i][3]=(dp[i-1][2]+dp[i-1][4])%mod;
        dp[i][4]=dp[i-1][0];
    }
    return (accumulate(dp[n-1].begin(),dp[n-1].end(),(long)0))%mod;
}
```

## 出勤记录II

https://leetcode.cn/problems/student-attendance-record-ii/description/

```c++
int checkRecord(int n) {
    auto dp=vector(n+1,vector(2,vector(3,(long)0)));long mod=1e9+7;
    //dp(i,j,k) i天内j次缺勤和最后k天迟到可获得奖励的序列数
    dp[0][0][0]=1;
    for(int i=1;i<=n;++i){
        //0次缺勤
        dp[i][0][0]=(dp[i-1][0][0]+dp[i-1][0][1]+dp[i-1][0][2])%mod;//第i天到场
        dp[i][0][1]=dp[i-1][0][0];dp[i][0][2]=dp[i-1][0][1];//第i天迟到
        //1次缺勤
        dp[i][1][0]=(dp[i-1][1][0]+dp[i-1][1][1]+dp[i-1][1][2]+ //缺勤在前几天
                     dp[i-1][0][0]+dp[i-1][0][1]+dp[i-1][0][2])%mod;//当天缺勤
        dp[i][1][1]=dp[i-1][1][0];dp[i][1][2]=dp[i-1][1][1]; //迟到
    }
    long res=0;
    for(auto& v:dp[n]) for(auto& t:v) res+=t;
    return res%mod;
}
```

## 播放列表数量

https://leetcode.cn/problems/number-of-music-playlists/description/

```c++
int numMusicPlaylists(int n, int goal, int k) {
    auto dp=vector(goal+1,vector(n+1,(long)0));dp[0][0]=1;
    //dp(i,j)::=前i首歌恰好j首不同的数量
    long mod=1e9+7;
    for(int i=1;i<=goal;++i){
        for(int j=1;j<=n && j<=goal;++j){
            //第i首选择未曾播放过的歌
            dp[i][j]=dp[i-1][j-1]*(n-j+1);
            //第i选择播放过的歌
            dp[i][j]+=dp[i-1][j]*max(j-k,0);
            dp[i][j]%=mod;
        }
    }
    return dp[goal][n];
}
```

## 自由之路

https://leetcode.cn/problems/freedom-trail/description/

```c++
int findRotateSteps(string ring, string key) {
    unordered_map<char, vector<int>> help;
    int m = ring.length(), n = key.length();
    // 保存每个字符在 ring 中的所有出现位置
    for (int i = 0; i < m; ++i) help[ring[i]].push_back(i);
    // dp[i][j] 表示找到 key 的第 i 个字符，并且 ring 的第 j 个字符与 key[i]匹配时的最小操作次数
    vector<vector<int>> dp(n, vector<int>(m, INT_MAX));
    for (int pos : help[key[0]]) {
        int dist = min(pos, m - pos); // 顺时针或逆时针的最小距离
        dp[0][pos] = dist + 1;        // 按按钮的操作次数为 1
    }
    for (int i = 1; i < n; ++i) 
        for (int prev_pos : help[key[i - 1]]) 
            for (int curr_pos : help[key[i]]) {
                int dist =min(abs(curr_pos - prev_pos),m - abs(curr_pos - prev_pos)); // 顺时针或逆时针的最小距离
                dp[i][curr_pos] =min(dp[i][curr_pos], dp[i - 1][prev_pos] + dist + 1);
            }
    int result = INT_MAX;
    for (int pos : help[key[n - 1]]) {
        result = min(result, dp[n - 1][pos]);
    }
    return result;
}
```

## K个逆序对数组

https://leetcode.cn/problems/k-inverse-pairs-array/description/

```c++

```

## DI序列有效排列

https://leetcode.cn/problems/valid-permutations-for-di-sequence/description/

```c++

```

## 多米诺-托米诺平铺

https://leetcode.cn/problems/domino-and-tromino-tiling/description/

<img src="https://pic.leetcode.cn/1668157188-nBzesC-790-5.png" alt="790-5.png" style="zoom:20%;" />

```c++
int numTilings(int n) {
    if(n==1 || n==2) return n;
    if(n==3) return 5;
    vector<long> dp(n+1); //dp(i)::=拼出2*n的方案数
    int mod=1e9+7;
    dp[1]=1,dp[2]=2,dp[3]=5;
    for(int i=4;i<=n;++i)
        dp[i]=(dp[i-1]*2+dp[i-3])%mod;
    return dp[n];
}
```

## 应试学生最大数

https://leetcode.cn/problems/maximum-students-taking-exam/description/

```c++

```

## 解码方法

https://leetcode.cn/problems/decode-ways/description/

```c++
int numDecodings(string s) {
    if(s[0]=='0') return 0;
    int n=s.length();
    vector<int> dp(n+1);//dp(i)::=s前i个字符的解码方案
    dp[1]=dp[0]=1;
    for(int i=2;i<=n;++i){
        if(s[i-1]>='7') {
            dp[i]=dp[i-1]; //7~9
            if(s[i-2]=='1') dp[i]+=dp[i-2];
        }else if(s[i-1]>='1'){  //1~6
            dp[i]=dp[i-1];
            if(s[i-2]=='1' || s[i-2]=='2') dp[i]+=dp[i-2];
        }else{
            if(s[i-2]=='1' || s[i-2]=='2') dp[i]=dp[i-2];
            else return 0;
        }
    }
    return dp[n];
}
```

## 解码方法II

https://leetcode.cn/problems/decode-ways-ii/description/

```c++
int numDecodings(string s) {
    if(s[0]=='0') return 0;
    int n=s.length(),mod=1e9+7;
    vector<long> dp(n+1);//dp(i)::=前i个字符解码方案数
    dp[0]=1,dp[1]=s[0]=='*'?9:1;
    for(int i=2;i<=n;++i){
        if(s[i-1]=='*'){
            dp[i]=9*dp[i-1];
            if(s[i-2]=='1') dp[i]+=9*dp[i-2];
            if(s[i-2]=='2') dp[i]+=6*dp[i-2];
            if(s[i-2]=='*') dp[i]+=15*dp[i-2];
        }else if(s[i-1]>='7'){  //7~9
            dp[i]=dp[i-1];
            if(s[i-2]=='1' || s[i-2]=='*') dp[i]+=dp[i-2];
        }else if(s[i-1]>='1'){  //1~6
            dp[i]=dp[i-1];
            if(s[i-2]=='1' || s[i-2]=='2') dp[i]+=dp[i-2];
            if(s[i-2]=='*') dp[i]+=2*dp[i-2];
        }else{ //0
            if(s[i-2]=='1' || s[i-2]=='2') dp[i]=dp[i-2];
            else if(s[i-2]=='*') dp[i]=2*dp[i-2];
            else return 0;
        }
        dp[i]%=mod;
    }
    return dp[n];
}
```

