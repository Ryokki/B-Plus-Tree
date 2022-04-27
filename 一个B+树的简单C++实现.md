---
tags:
  - B+树
  - 索引
status: todo
---
***
代码见 https://github.com/Ryokki/B-Plus-Tree
- [ ] todo:证明算法正确性
- [ ] 看这个https://zhuanlan.zhihu.com/p/149287061
- [ ] https://github.com/youkaichao/minidb 这个是清华的大佬写的 https://zhuanlan.zhihu.com/p/67374506
- [ ] https://www.codedump.info/post/20200609-btree-1/
- [ ] https://github.com/enpeizhao/duck_db

***
## 参考文章
- https://zhangshun97.github.io/2020/06/27/b-tree/ B+树循序渐进
- 斯坦福的cs345 https://web.stanford.edu/class/cs346/2015/notes/Blink.pptx [[STANFOR Blink.pdf]]
- cmu的15-445 https://15445.courses.cs.cmu.edu/fall2021/slides/07-trees.pdf [[CMU 07-trees.pdf]] [[CMU 08-indexconcurrency.pdf]]
- Berkley的cs186  https://cs186berkeley.net/sp20/ [[UCB 06 Trees and Indexes FINAL animated.pptx.pdf]] [[UCB 07 Tree-Indexes-final-jmh.pptx.pdf]]
- MIT的6.830  http://dsg.csail.mit.edu/6.830/ [[MIT B+Trees.pdf]]
- https://www.cs.usfca.edu/~galles/visualization/BPlusTree.html B+树可视化

## 定义
M阶B+树：
* B+树，称为M-way search tree，表明每个结点最多有M个孩子
* 设n为key-value的数量，则:`M/2-1 ≤ #keys ≤ M-1` （除根结点外）
	* M=3时，至少1个，最多2个
	* M=4时，至少2个，最多3个
* B+树的叶子节点之间由双向链表连接 (即每个node有个prev_node和next_node)
* ![](https://kkbabe-picgo.oss-cn-hangzhou.aliyuncs.com/img/20220417170212.png)
节点内有N个元素key[]和val[]，那么就有N+1个子结点node[]，并且子节点node[i]的范围是`[key[i-1], key[i])`
比如node[0]<key[0], key[0]<=node[1]<key[1], ...
> 注意是左闭右开

## 实现
## 数据结构设计
一个B+树是由一个个Node组织的，并且我们需要记录B+树的root结点等信息，所以需要设计两个数据结构
### Node
下面是node的图
<img src="https://kkbabe-picgo.oss-cn-hangzhou.aliyuncs.com/img/image-20220413183135124.png" alt="image-20220413183135124" style="zoom: 50%;" />
这里有个地方画错了，internal node是没有vals的，只有nodes

`Node`:
```c++
template <class key_type, class val_type>
class Node  
{
public:
  Node() {
    next_leaf = nullptr;
    prev_leaf = nullptr;
  };

  vector <key_type> keys;  
  vector <val_type> vals;  // 对于叶子节点，有vals数据

  vector <Node<key_type,val_type>*> nodes;  // 对于中间node，有指向下一层的nodes
  
  // 对于叶子，是双向链表
  class Node <key_type, val_type>*next_leaf;
  class Node <key_type, val_type>*prev_leaf;  

};
```

### Tree
```c++
// M阶，node里最大size=M-1，最大孩子数=M
template <class key_type, class val_type, size_t max_children = 3>  
class Tree {
private:
  size_t num_elements;  // 这颗树中存的key-value的数量 <这个可以没有>
  Node<key_type, val_type>*root;  // Tree Root
  size_t max_degree;  // M
}
```

## insert
#### 流程

<img src="https://kkbabe-picgo.oss-cn-hangzhou.aliyuncs.com/img/20220418122406.png" style="zoom: 33%;" />

**insert的流程**：
1. 从root开始向下查找，找到应该插入的叶子节点，并将该key-value插入到这个位置
2. 如果L满了，需要分裂
   1. L分裂成L1和L2
   2. 对于叶子节点，需要将L2的第一个元素`COPY`到parent；对于内部节点，需要将L2的第一个元素`MOVE`到parent
3. 如果parent也满了，那么重复第二步

举个例子：

插入 19，我们会遍历整个树找到整个叶子节点，然后插入:
![](https://kkbabe-picgo.oss-cn-hangzhou.aliyuncs.com/img/20220418130648.png)

我们插入 21，如下图：
![](https://kkbabe-picgo.oss-cn-hangzhou.aliyuncs.com/img/20220418130818.png)
* 先是将21插入[17,19,21]这样的node，发现满了之后分裂。将17作为L1，将19,21作为L2 (19是median key).
* 然后将 L2 的第一个元素 (19) `MOVE` 到 parent，(准确的说，一开始是parent的node[i] (i=0) 指向child,分裂时让L2的第一个元素复制到插入到key[i]和node[i]) 如下图：

![](https://kkbabe-picgo.oss-cn-hangzhou.aliyuncs.com/img/20220418131008.png)

我们再插入36：
![](https://kkbabe-picgo.oss-cn-hangzhou.aliyuncs.com/img/20220418131118.png)
叶子溢出到parent，parent也满了，要分裂成两个：
![](https://kkbabe-picgo.oss-cn-hangzhou.aliyuncs.com/img/20220418131202.png)
然后再将L2的第一个元素`MOVE`到parent ，变成如下图即可：
![](https://kkbabe-picgo.oss-cn-hangzhou.aliyuncs.com/img/20220418131312.png)

回顾一下这个内部节点的分裂过程：
- 一开始是[19,27,32]，有子节点node[0,1,2,3]，需要分裂。
- median_key=27,这本来应该是L2的第一个元素，但由于是`MOVE`，所以L1=19,node1=node[0,1]。L2=32,node2=node[2,3]
- L1是key[0~a] (key[0]),L2是key[a+2~最大] (key[2])
- L1有node[0~a+1]，这个是保留不变的。   L2有node[a+2~最大],这个也是不变的.

#### 正确性证明
用简短的话总结一下insert的流程：
insert，就是先找到对应的叶子节点，在里面插入这个kv。如果没有满的话，ok插入结束。如果满了，会分裂，分裂的流程如下： 
1. 此时的size = M,我们取[0,...,M/2-1]作为L1，取[M/2,M-1]作为L2。假设其parent的node[i]指向我们的L，则让L2的第一个key插入到key[i]中,并将L2插入到node[i]中  。此时parent可能会满，满了也需要分裂。
2. 对于中间节点，parent的node[i]指向L，我们的L满了，让kv[0, ..., M/2-1]为L1，让kv[M/2 + 1, ..., M]为L2，并且让L.node[0, ... , M/2]为L1的node，让剩余的为L2的node，并把key[M/2]插入到parent.key[i]中，并让parent.node[i]分裂.
递归...,直到传到一个节点没有满为止

***
证明1:
问题: 对于叶子节点，parent的node[i]指向L，我们的L满了，让kv[0, ..., M/2-1]为L1，让kv[M/2, ..., M]为L2。 然后我们让L2的第一个元素key[M/2]插入到key[i]，并让node[i]分裂成两个，分别是L1和L2，证明这仍然是一个正确的B+树
解: 我们选中的是node[i]，即L，所以 key[i-1] <= L中key的值 < key[i].
我们将L2的第一个元素插入到key[i]的位置中,就变成了 ...,key[i-1],L2的第一个元素,key[i].
我们将L2的node[i]位置分裂成L1和L2，就变成了       ...,node[i-1],L1,L2,node[i+1],...
所以我们需要证明插入后的 new_key[j-1] <= new_node[j] < new_key[j] 即可:
	首先，对于下标j <= i-1 的 new_node 来说没有影响
	对于j = i, key[i-1] <= L1 < L2的第一个元素,这是正确的，因为:
		L1是原先的L的左边部分，L1当然 < L2的第一个元素 
		L1是原先的L部分, key[i-1] <=L < key[i],故也成立
	对于j = i+1, L2的第一个元素 <= L2 < key[i]，因为：
		左式当然成立
		L2是原先的L部分, key[i-1] <=L < key[i],故也成立 
	对于j > i+1的部分没有影响
综上，得证
***
证明1简述:
其实就是，原来是key[i-1]<node[i]=L<key[i],现在L分裂成L1和L2了，所以要在key[i]插入一个L2的首元素.
![[Drawing 2022-04-22 10.22.05.excalidraw]]
***
证明2:
问题: 对于中间节点，parent的node[i]指向L，我们的L满了，让kv[0, ..., M/2-1]为L1，让kv[M/2 + 1, ..., M]为L2，并且让L.node[0, ... , M/2]为L1的node，让剩余的为L2的node，并把key[M/2]插入到parent.key[i]中，并让parent.node[i]分裂.        证明这仍是一个正确的B+树
解: 

#### insert kv
1. 找到要插入的叶子节点n (插入的时候记录路径中的parent node,以及node的index)
2. 如果key存在了，则update value，返回；如果key不存在，那么叶子节点中插入这个kv，如果要分裂的话执行split 

```c++
  /* find the leaf node */
  while (1) {
    /* 找到应该遍历的子节点node[i]  <注意是小于> */
    for (i = 0; i < n->keys.size(); i++) {
      if (key < n->keys[i]) break;  
    }
    /* 记录路径中的各个parent，并判断为叶子节点时终止，更新n */
    if (n->nodes.size() != 0) {
      traverse_indices.push_back(i);  // 记录遍历路径中node的下标
      parents.push_back(n);   // 记录遍历路径中的node
      n = n->nodes[i];
    } else break; // n的size为0，表示这个node是叶子结点了，ok就找到啦
  }

  /* key exists */
  for (j = 0; j < n->keys.size(); j++) {  
    if (n->keys[j] == key) { // 如果key存在了，那么就直接修改对应的value，return
      n->vals[j] = val;
      return;
    }
  }
  
  /* key not exists */
  num_elements++;
  /* put the val and key in the proper postion */
  // 这个地方挺巧妙的，这里在i的位置插入是因为前面最后一次while循环中，i遍历了keys,使得keys[i-1]<key<keys[i]，所以在i的位置插入
  n->keys.insert(n->keys.begin() + i, key);
  n->vals.insert(n->vals.begin() + i, val);
  
  /* split the node until the bucket(key) is not full any more */
  while (n->keys.size() == max_degree) {
	  // ... split
  }
```

#### split
