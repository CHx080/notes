==*FloodFill---性质相同的连通块*==

# 图像渲染

https://leetcode.cn/problems/flood-fill/description/

```cpp
int prev_color,new_color;
int dx[4]={1,-1,0,0},dy[4]={0,0,1,-1};
void dfs(vector<vector<int>>& image,int i,int j)
{
    image[i][j]=new_color;
    for(int k=0;k<4;++k)
    {
        int x=dx[k]+i,y=dy[k]+j;
        if(x>=0 && y>=0 && x<image.size() && y<image[0].size() && image[x][y]==prev_color)
            dfs(image,x,y);
    }
}

vector<vector<int>> floodFill(vector<vector<int>>& image, int sr, int sc, int color) {
    if(image[sr][sc]==color) return image;
    prev_color=image[sr][sc];
    new_color=color;
    dfs(image,sr,sc);
    return image;
}
```

# 岛屿数量

https://leetcode.cn/problems/number-of-islands/description/

```cpp
int dx[4]={1,-1,0,0};
int dy[4]={0,0,1,-1};
int cnt=0,m,n;
void dfs(vector<vector<char>>& grid,int i,int j)
{
    grid[i][j]='0';
    for(int k=0;k<4;++k)
    {
        int x=dx[k]+i,y=dy[k]+j;
        if(x>=0 && y>=0 && x<m && y<n && grid[x][y]=='1')
        {
            dfs(grid,x,y);
        }
    }
}

int numIslands(vector<vector<char>>& grid) {
    m=grid.size(),n=grid[0].size();
    for(int i=0;i<m;++i)
        for(int j=0;j<n;++j)
        {
            if(grid[i][j]=='1')
            {
                ++cnt;
                dfs(grid,i,j);
            }
        }
    return cnt;
}
```

# 岛屿最大面积

http://leetcode.cn/problems/max-area-of-island/description/

```cpp
int square=0;
int temp=0;
int dx[4]={1,-1,0,0},dy[4]={0,0,1,-1};
void dfs(vector<vector<int>>& grid,int i,int j)
{
    ++temp;
    grid[i][j]=0;
    for(int k=0;k<4;++k)
    {
        int x=dx[k]+i,y=dy[k]+j;
        if(x>=0 && y>=0 && x<grid.size() && y<grid[0].size() && grid[x][y]) 
            dfs(grid,x,y);
    }
}

int maxAreaOfIsland(vector<vector<int>>& grid) {
    for(int i=0;i<grid.size();++i)
        for(int j=0;j<grid[0].size();++j)
        {
            if(grid[i][j])
            {
                temp=0;dfs(grid,i,j);
            }
            square=max(square,temp);
        }
    return square;
}
```

# 围绕区域

https://leetcode.cn/problems/surrounded-regions/description/

```cpp
unordered_set<char*> cannot;
int dx[4]={1,-1,0,0},dy[4]={0,0,1,-1};
void dfs(vector<vector<char>>& board,int i,int j,bool modify=false)
{
    cannot.insert(&board[i][j]);
    if(modify) board[i][j]='X';
    for(int k=0;k<4;++k)
    {
        int x=dx[k]+i,y=dy[k]+j;
        if(x>=0 && x<board.size() && y>=0 && y<board[0].size() 
           && board[x][y]=='O' && !cannot.count(&board[x][y])) dfs(board,x,y,modify);
    }
}

void solve(vector<vector<char>>& board) {
    int m=board.size(),n=board[0].size();
    for(int j=0;j<n;++j)
    {
        if(board[0][j]=='O') dfs(board,0,j);
        if(board[m-1][j]=='O') dfs(board,m-1,j);
    }
    for(int i=1;i<m-1;++i)
    {
        if(board[i][0]=='O') dfs(board,i,0);
        if(board[i][n-1]=='O') dfs(board,i,n-1);
    }
    for(int i=1;i<m-1;++i)
        for(int j=1;j<n-1;++j)
        {
            if(board[i][j]=='O' && !cannot.count(&board[i][j])) dfs(board,i,j,true);
        }
}
};
//进行2次深度优先，第1次把与边缘相接的o区域做一个标记，第2次把未标记的区域全部改成0
```

# 大洋水流

https://leetcode.cn/problems/pacific-atlantic-water-flow/description/

*逆向*

```cpp
int m,n;
vector<vector<bool>> pac;
vector<vector<bool>> atl;
int dx[4]={1,-1,0,0},dy[4]={0,0,1,-1};
void dfs(vector<vector<int>>& heights,int i,int j,vector<vector<bool>>& sea)
{
    sea[i][j]=true;
    for(int k=0;k<4;++k)
    {
        int x=dx[k]+i,y=dy[k]+j;
        if(x>=0 && x<m && y>=0 && y<n && !sea[x][y] && heights[x][y]>=heights[i][j]) dfs(heights,x,y,sea);
    }
}

vector<vector<int>> pacificAtlantic(vector<vector<int>>& heights) {
    m=heights.size(),n=heights[0].size();
    pac.resize(m,vector<bool>(n));
    atl.resize(m,vector<bool>(n));
    for(int j=0;j<n;++j)
    {
        dfs(heights,0,j,pac);
        dfs(heights,m-1,j,atl);
    }
    for(int i=0;i<m;++i)
    {
        dfs(heights,i,0,pac);
        dfs(heights,i,n-1,atl);
    }
    vector<vector<int>> ret;
    for(int i=0;i<m;++i)
        for(int j=0;j<n;++j)
        {
            if(atl[i][j] && pac[i][j]) ret.push_back({i,j});
        }
    return ret;
}
```

# 扫雷

https://leetcode.cn/problems/minesweeper/description/

```cpp
int dx[8]={1,-1,0,0,1,-1,-1,1};
int dy[8]={0,0,1,-1,1,-1,1,-1};
int row=0,col=0;

void dfs(vector<vector<char>>& board,int i,int j)
{
    if(i<0 || i==row || j==col || j<0 || board[i][j]=='B') return;

    //检查九宫格范围内是否有地雷
    int mine=0;
    for(int k=0;k<8;++k)
    {
        int x=i+dx[k],y=j+dy[k];
        if(x>=0 && x<row && y>=0 && y<col &&
           board[x][y]=='M') mine++;
    }
    if(mine>0) {board[i][j]=mine+'0';return;}

    board[i][j]='B';
    for(int k=0;k<8;++k)
    {
        int x=i+dx[k],y=j+dy[k];
        dfs(board,x,y);
    }
}
vector<vector<char>> updateBoard(vector<vector<char>>& board, vector<int>& click) {
    int i=click[0],j=click[1];
    row=board.size();col=board[0].size();
    if(board[i][j]=='M') {board[i][j]='X';return board;}
    dfs(board,i,j);
    return board;
}
```

# 衣柜整理

https://leetcode.cn/problems/ji-qi-ren-de-yun-dong-fan-wei-lcof/description/

```cpp
int ret=0,_cnt=0,_m=0,_n=0;
int dx[2]={1,0},dy[2]={0,1};
bool vis[100][100]={false};
int digit(int x)
{
    int sum=0;
    string&& s=to_string(x);
    for(auto c:s)sum+=c-'0';
    return sum;
}
void dfs(int i,int j)
{
    if(i==_m || j==_n || vis[i][j]) return;
    if(digit(i)+digit(j)>_cnt) return;
    vis[i][j]=true;
    ret++;
    for(int k=0;k<2;++k)
    {
        int x=i+dx[k],y=j+dy[k];
        dfs(x,y);
    }
}
int wardrobeFinishing(int m, int n, int cnt) {
    _cnt=cnt;_m=m;_n=n;
    dfs(0,0);
    return ret;
}
```