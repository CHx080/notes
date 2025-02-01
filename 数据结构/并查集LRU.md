# 并查集
将n个不同的元素划分成一些不相交的集合。开始时，每个元素自成一个单元素集合，然后按一定的规律将归于同一组元素的集合合并。在此过程中要反复用到查询某一个元素归属于那个集合的运算。适合于描述这类问题的抽象数据类型称为**并查集(union find set)**

## ADT

> struct UnionFindSet
> attribute: 
> 		不相交的集合,抽象为数组
> function: 
> 		检查两个元素是否属于同一个集合 bool inSameSet(e1,e2);
> 		寻找集合根元素 e findRoot(e) ;
> 		合并集合 void merge(e1,e1);
> 		计算集合大小 size_t getCnt();
>
> end 

**如何表示同一个集合的元素：**
	可以借助数组抽象结构的方法对同一集合的元素进行分类，把代表某个集合的元素称为**特征元素**
	把每一个元素都映射成**唯一的编号id**，id同时也作为数组的下标
	规定每一个下标所对应值为集合中其他元素的编号id，如果**数组中某一个id位置的值为-x，代表它是一个特征元素,并且集合中有x个元素**

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/7b55fdb70f754ce6a1d9439a7a8e75f2.png)

## code

```cpp
//忽略了映射关系的构建，数组下标值就是对应的真实元素值
class UnionFindSet
{
private:
	vector<int> ufs;
public:
	UnionFindSet(unsigned int n=0):ufs(n,-1)
	{}  //初始状态所有的元素都各自为一个集合，元素之间没有任何关系
	unsigned int getCnt()
	{
		unsigned int cnt=0;
		for(int x:ufs) if(x==-1) ++cnt;
		return cnt;
	}	//统计ufs数组中-1的个数即可获得集合个数
	unsigned int findRoot(unsigned int e)
	{	
		unsigned int eigenval=e;
		while(ufs[eigenval]>=0) eigenval=ufs[eigenval];
		return eigenval;
	}	//迭代向上寻找，直至数组中的值为-1
	bool inSameSet(unsigned int e1,unsigned int e2)
	{
		return findRoot(e1)==findRoot(e2);
	}
	void merge(unsigned int e1,unsigned int e2)
	{
		unsigned int eigen1=findRoot(e1);
		unsigned int eigen2=findRoot(e2);
		if(eigen1!=eigen2) 
		{
			ufs[eigen1]+=ufs[eigen2];
			ufs[eigen2]=eigen1;
		}//把e2所在集合的特征元素对应的值改为e1所在集合的特征元素即可完成2个集合的合并
	}
};
```
## 合并、查找优化
> **合并优化：小集合合并到大集合中，做到更多的元素能在短时间内找到特征元素**
> ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/f5fa73ac64954e34b572a3d05bdade3c.png)

```cpp
void merge(unsigned int e1,unsigned int e2)
{
	unsigned int eigen1=findRoot(e1);
	unsigned int eigen2=findRoot(e2);
	if(eigen1!=eigen2)
	{
		if(ufs[eigen1]>ufs[eigen2]) swap(eigen1,eigen2); //统一规定集合eigen1的元素个数更大
		ufs[eigen1]+= ufs[eigen2];
		ufs[eigen2]=eigen1;
	}
}
```

--------------

> **查找优化：查找的过程中不断尝试缩小个别节点到特征节点的距离**
> ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/e2181929ac914e6cbbddc32dc8cdde53.png)
> 可以考虑每一次进行findRoot查找时对集合元素关系进行调整，使得每一个元素在集合中更接近特征元素
> ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/24848056def841bbb9057d1d858e33b3.png)

```cpp
unsigned int findRoot(unsigned int e) 
{
	unsigned int eigen = e, prev = -1;
	while (_ufs[eigen] >= 0) 
	{
		int tmp = ufs[eigen];//向上查找
		if (prev>=0) ufs[prev] = tmp;
		prev = eigen;
		eigen = tmp;
	}
	return eigen;
}
```

# LRU 
LRU是Least Recently Used的缩写，意思是最近最少使用，它是一种**Cache替换算法**
淘汰长久未使用的数据，cache的大小是固定有限的，当空间满时如果还需要加入数据就必须淘汰部分先前的数据

普通的LRU链表 数据块通过一个**双向链表**级联，规定**链表尾节点为长时间未使用即将被淘汰的资源**，链表头节点为最近使用的数据，借助**哈希表**实现对各个数据块的快速定位，达到增删查改的时间复杂度均为O（1）
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/c6b81b46de30414ea9e6ff6b7fcc8254.png)

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/cbbb7f3c984c44e286111cc0e0af1ff3.png)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/920c2858b84c4f7081648cb024398705.png)

## ADT

> struct LRUCache <int>
>
> attribute:
>
> ​	双向链表list
>
> ​	散列表map
>
> function:
>
> ​	获取节点 int get(key);
>
> ​	增加节点 int put(key,value);
>
> end

## code

```cpp
class LRUCache {
    size_t _size=0; //链表当前长度
    size_t _capacity; //cache最大容量
    list<pair<int,int>> _list;
    typedef list<pair<int,int>>::iterator iterator;
    unordered_map<int,iterator> _map;
public:
    LRUCache(int capacity) 
        :_capacity(capacity)
    {}
    
    int get(int key) {
        if(!_map.count(key)) return -1;
        iterator it=_map[key];
        int val=it->second;
        _list.push_front(*it);
        _list.erase(it);
        _map[key]=_list.begin();
        return val;
    }
    
    void put(int key, int value) {
        if(_map.count(key)) //更新节点
        {
            iterator it=_map[key];
            it->second=value;
            _list.push_front(*it);
            _list.erase(it);
            _map[key]=_list.begin();
        }
        else
        {
            if(_size<_capacity)
            {
                ++_size;
                _list.push_front({key,value});
                _map[key]=_list.begin();
            }
            else
            {
                auto& last=_list.back(); //获取链表尾节点
                _map.erase(last.first); //删除map中指向尾的映射关系
                _list.pop_back();   //删除链表尾节点
                _list.push_front({key,value}); //头插
                _map[key]=_list.begin(); //更新map
            }
        }
    }
};
```

