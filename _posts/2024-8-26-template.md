---
layout: post
title: 部分算法板子
tags: algorithm
excerpt: ~~~
---


## 线段树
``` c++
// 是平衡二叉树
// 父节点代表整个区间的和，根节点最长，叶子节点为真实数据
struct Node
{
int l, r;
int sum;
}tr[4*N];

// 对于x节点，父节点为 x >> 1; 左子节点为 x << 1; 右子节点为 x << 1 | 1;

// 用子节点消息更新当前节点信息（可能可以省略，将这句代码移到其他函数中）
void pushup(int u)
{
    tr[u].sum = tr[u << 1].sum + tr[u << 1 | 1].sum;
}

// 在一段区间上初始化线段树，(根节点下标，区间)
void build(int u, int l, int r)
{
    // 如果是叶子节点
    // a[r]是真实数据
    if(l == r) tr[u] = {l, r, a[r]};
    else
    {
        tr[u] = {l, r};
        int mid = l + r >>1;
        build(u << 1, l, mid);
        build(u << 1 | 1, mid +1, r);
        pushup(u);
    }
}

// 查询
int query(int u, int l, int r)
{
    // 如果当前区间包含要查的部分，直接返回当前总和
    if(tr[u].l >= l && tr[u].r <= r) return tr[u].sum;
    int mid = tr[u].l + tr[u].r >> 1;
    int sum = 0;
    // 如果mid大于等于查询区间左端点，则拿mid的sum
    if(l <= mid) sum = query(u << 1, l, r);
    // 如果mid小于查询区间右端点，则拿mid的sum
    if(r > mid) sum += query(u << 1 | 1, l, r);
    return sum;
}

// 修改
void modify(int u, int x, int v)
{
    if(tr[u].l == tr[u].r) tr[u].sum += v;
    else
    {
        int mid = tr[u].l +tr[u].r >> 1;
        // 小于等于找左子节点
        if(x <= mid) modify(u << 1, x, v);
        else modify(u << 1 | 1, x, v);
        pushup(u);
    }
}

// 构建结构
build(1, 1, n);
```

## 树状数组
```c++
int a[100010];// 数据
int q[100010];// 树状数组

// 查看这个x在第几层
int lowbit(int x)
{
    return x&-x;
}

// v为加的数
int add(int x, int v)
{
    for(int i=x; i<=n; i+=lowbit(i)) q[i]+=v;
}

// 求和
void query(int x)
{
    int res=0;
    for(int i=x; i; i-=lowbit(i)) res+=q[i];
    return res;
}

// 构建结构
for(int i=1; i<=n; i++)
    add(i, a[i]);

void add(int a, int b)// 这里采用数组实现邻接表来存储图，也就是将多个单链表h[i]拼起来
{
    e[idx] = b, ne[idx] = h[a], h[a] = idx ++ ;// 头插法创建单链表，新节点指向已有节点
    // h[a]是单链表a的起点，最后一个插入元素的地址，也是idx区间的终点
}
```

## 二分
右边界，目标值需要严格 大于 被二分的数组的值
```c++
// b[k]是目标值
while (l < r)
{
    long long mid = (l + r + 1) / 2;
    if (b[k] > a[mid])// 判断check正确
    {
        l = mid;
    }
    else
    {
        r = mid-1;
    }
}
```

l为边界
```c++
// b[k]是目标值
while (l < r)
{
    long long mid = (l + r) / 2;
    if (b[k] < c[mid])
    {
        r = mid;
    }
    else
    {
        l = mid + 1;
    }
}
// r是边界
```

## 并查集
```c++
vector<int> father(...);

for(int i=1; i<=n; i++) father[i] = i;

int find(int u)
{
    // return u==father[u]?u:father[u]=find(father[u]);
    if(father[u] != u) father[u] = find(father[u]);
    return father[u];
}

void join(int u, int v)
{
    u=find(u);
    v=find(v);
    if(u==v) return;
    father[v]=u;
}

bool IsSame(int u, int v)
{
    u=find(u, father);
    v=find(v, father);
    return u==v;
}
```

## 求最大公约数
```c++
int gcd(int a, int b)
{
    return b ? gcd(b, a % b) : a;
}
```

## 求约数
```c++
vector<int> get_divisors(int n)
{
    vector<int> res;

    for(int i=1; i<=n/i; i++)
        if(n % i == 0)
        {
            res.push_back(i);
            if( i!= n/i) res.push_back(n / i);
        }

    sort(res.begin(), res.end());
    return res;
}
```

## 快速幂
```c++
int n, m, k, x;// n个人，移动m，进行1e轮，求x位置

long long qmi(int a, int b, int p)
{
    long long res=1%p;
    while(b)
    {
        if(b&1) res=res*a%p;
        a=a*(long long)a%p;
        b>>=1;
    }
    return res;
}

int main()
{
    cin>>n>>m>>k>>x;
    
    cout<<(x+(long long)qmi(10, k, n) * m)%n<<endl;
    return 0;
}
```