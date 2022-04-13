[TOC]

# 参考文章

https://zhangshun97.github.io/2020/06/27/b-tree/ B+树循序渐进，好文

https://web.stanford.edu/class/cs346/2015/notes/Blink.pptx 斯坦福的ppt

https://15445.courses.cs.cmu.edu/fall2021/slides/07-trees.pdf cmu的ppt

https://www.cs.usfca.edu/~galles/visualization/BPlusTree.html B+树可视化

# B+树

M阶B+树：

* B+树，称为M-way search tree，表明每个结点最多有M个孩子

* 设n为key-value的数量，则:`M/2-1 ≤ #keys ≤ M-1` [除根结点外]  

  M=3时，至少1个，最多2个

  M=4时，至少2个，最多3个

* 完美平衡，因为每个叶子结点在同一深度



# 实现

<img src="https://kkbabe-picgo.oss-cn-hangzhou.aliyuncs.com/img/image-20220413183135124.png" alt="image-20220413183135124" style="zoom: 38%;" />





