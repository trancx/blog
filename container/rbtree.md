---
description: 之前的笔记
---

# RB Tree - 红黑树

## 序

红黑树是一种非常常见的结构，但是网络上的文章很少将它的原理解释清楚，尤其是添加删除的情况的分类，这里将以前的笔记一起放上，正好方便读者了解，进程调度用了非常多的红黑树结构。

## 为什么能平衡

![&#x6781;&#x7AEF;&#x60C5;&#x51B5;](../.gitbook/assets/image%20%2831%29.png)

红黑树的定义这里不赘述，同时我也不会讲解旋转的操作，因为这些有的是文章讲解。首先，我解释一点，红黑树的平衡来自哪里。

{% hint style="info" %}
从根结点出发，任意一条的路径上的黑色节点数相等
{% endhint %}

那么极端情况就容易理解了，即一边全黑，另外一边红黑交替，也就是说，最短路径的长（高）度为 n，那么另外最长（高）的长（高）度为 2n，红黑树的容忍的树高差就是树高最小的值，所以它可以不那么平衡，这是跟 AVL树 的区别。

## 旋转如何能维持平衡

首先，旋转的本质只是节点的左移和右移，为的就是维护子树的树高平衡。

![](../.gitbook/assets/image%20%2885%29.png)

考虑上图的左旋，其实就是黑色节点跑到了左子树，原来在右子树的节点，跑到了根节点。宏观上理解，就是**右子树的节点变少了，左子树的节点变多**。非要理解，就是旋转点变为左节点，而旋转点的右节点变为父母，其左子树附加到旋转点右子树。多复杂，就用宏观理解，其实就是平衡树，而子树的移动是理所当然的。

![](../.gitbook/assets/image%20%28108%29.png)

这个也是如此，可能比较抽象，但最终的结果都是，旋转点到了子树，减少了一端的子树的高度，这就是旋转的实质。可能上面这幅图没有平衡的感觉，是的，实际操作，还要左旋一次，也就是左旋的图那样～

{% hint style="info" %}
左旋即 旋转点到左子树，右子树树高减少，右旋亦同理。
{% endhint %}

## 红黑树的插入与删除

插入节点的步骤最开始和搜索二叉树相同，然后插入的节点是作为红色节点，然后如果树高不平衡，就需要调整（ rotate ），删除也类似，如果左右都有节点，则先找到它的后继（successor），两者替换位置后再删除，这样做的目的是选择一个只有单子树的位置，容易操作。

### Insertion

第一步，按搜索树插入新节点，并标记新节点为红色

![&#x56FD;&#x5916;&#x6559;&#x6388;&#x7684;&#x8BFE;&#x4EF6;](../.gitbook/assets/image%20%2828%29.png)

这个课件使用的是 edge 的说法，而不是我们平常说的节点颜色，其实是一样的。

![](../.gitbook/assets/image%20%286%29.png)

一直忘了说，null 也算是黑色节点，其实不影响判断，因为每条路径下，最后必然是叶子节点，所以也必然都有 null，那么大家都会抵消了。

这里说明了一点，就是需要调整的时机，即出现了连续俩个红色的节点，也就是说，我们插入之后发现父母节点是红色，那么就得调整了。

{% hint style="info" %}
如果父母是黑色，不用调整，看了之前的分析，其实决定“树高”的一直都是黑色节点，红黑树只需要保证，没有连续的红色节点，每条路径的黑色节点相同，所以这里不考虑父母是黑色结点的情况了。
{% endhint %}

#### 插入新节点要考虑三代节点

g 指代 grandparent，p 指代 parent，还有一个节点，即 uncle，这是插入的时候要考虑的三个结点，当然，插入的节点本身一定是红色了。

#### 一次旋转即可的情况

![](../.gitbook/assets/image%20%28114%29.png)

上面说明的是一个情况，即 uncle 是黑色，p 是红色，可以得到的一点，g 一定是**黑色**，因为只有黑色节点后面才可以跟红色节点。现在说明，这里右旋的原因，现在明显是 g-&gt;p-&gt;n 这一子树的树高会比 g-&gt;uncle 那一子树多（大多数正常情况下），首先，**这俩条路径的黑色节点一定相同**！这是RB tree 存在的意义，然后现在出现了连续俩个红色节点，违反了规则，其实就是 **树高** 已经不平衡了～

我们能做的一件事，就是把一个红色节点，移动到另外一条子树，能这样的做的理由就是，g 是黑色，u 是黑色，**它们之间，起码还有一个红色节点的位置**！

也说了旋转的实质，所以这里，g 直接右旋到右子树，然后发现，右旋导致了一点，看下图

![](../.gitbook/assets/image%20%2866%29.png)

这里的右旋，导致左子树的黑色节点，少了，右子树的黑色节点，多了。所以我们重新染色。

![](../.gitbook/assets/image%20%2829%29.png)

其实大多数情况下，uncle 应该是空节点，因为插入的节点的位置肯定是叶子，那么可以发现原来二叉树已经不平衡了，所以可以判断 U 是空节点，不过这里更加形象，所以画出。当然，最后一种情况可能导致这个结果，也就是，**染色**导致上层的不平衡（看 Uncle 是红色节点的部分 ）。

再给出，另外一个方向的情况，不在讨论了。

![&#x6B64;&#x5904;&#x9ED8;&#x8BA4; U &#x662F;&#x9ED1;&#x8272;&#x8282;&#x70B9;](../.gitbook/assets/image%20%2844%29.png)

#### 复杂的情况，不连成直线

当 N 是 P 的右节点，而 P 是 G 的左节点，或者反过来，即 G P N 不再是一条直线的时候，需要二次旋转，看下图。

![](../.gitbook/assets/image%20%2870%29.png)

发现了吗，经过一次旋转（以P为基准）之后，我们得到了前一种情况，只是现在 N 变成了 “P“，P 变成了 “N“，所以虽然说复杂，其实就是多了一步转换而已。给出转换图。

![&#x84DD;&#x8272; &#x8868;&#x793A;&#x65B0;&#x63D2;&#x5165;&#x8282;&#x70B9;&#xFF0C;&#x662F;&#x7EA2;&#x8272;](../.gitbook/assets/image%20%284%29.png)

看看教授的课件如何讲解的。

![](../.gitbook/assets/image%20%2871%29.png)

上图的是 left-right rotation，还有一个 right-left 其实我的图片已经有了，本质就是经过一次旋转后，转换为最早我们说的那个情况而已。

{% hint style="info" %}
实际编写代码的时候，可以先处理这个情况，转换成上面的GPN连成直线的情况，这是代码编写的技巧。
{% endhint %}

#### 稍微简单的情况，Uncle is Red

最简单的情况，当然就是父亲节点是黑色，什么都不处理，这里考虑 Uncle 不再是黑色，而是红色。

![](../.gitbook/assets/image%20%2821%29.png)

思考一会，现在 G U 之间，可没有位置安放一个红色节点了，如何是好？

答案是，交给上层处理，这其实说明，这一区域的树其实已经比较平衡了，没有较大的树高差，至于在往上，我们也不知道，所以传递上去，那么如何传递，且看。

![](../.gitbook/assets/image%20%2824%29.png)

实质是，左右得到一个黑色节点，然后把 G变为黑色，最后的结果是不多不少，但是 G 可能其父母是红色，所以，又转换为了之前的情况～ 看教授的课件是如何说明的。

![](../.gitbook/assets/image%20%2890%29.png)

![](../.gitbook/assets/image%20%2840%29.png)

所以，代码处理的过程中，为 G 变成了新的节点，重新再过函数处理流程一遍（即循环的 continue）

下面给出我画的一张图

![N G P U&#x6CA1;&#x6709;&#x6807;&#x51FA;&#xFF0C;&#x84DD;&#x8272;&#x662F;&#x65B0;&#x63D2;&#x5165;&#x8282;&#x70B9;](../.gitbook/assets/image%20%2810%29.png)

#### 总结

![](../.gitbook/assets/image%20%28122%29.png)

注意，旋转能解决的，必然是瞬间完成，只有 Uncle 是红色的时候，有可能一直向上传递，这样就有多次处理了。思考一个问题，黑色节点是如何增多的？

之前的处理，都是保证了黑色节点没有改变，其实是当旋转之后，发现旋转点是根结点，而现在的根结点又是红色，于是将根结点变为黑色，或者，向上传递之后，发现 G 就是根结点，而 G 现在是红色，所以变为黑色，就这样，黑色节点增加了。～

#### Kernel rbtree 注释

很多地方都用到了，红黑树结构，这里给出一些注释，其实红黑树的插入比较简单一些。

{% code-tabs %}
{% code-tabs-item title="rbtree.c" %}
```c
void rb_insert_color(struct rb_node *node, struct rb_root *root)
{
	struct rb_node *parent, *gparent;

	while ((parent = rb_parent(node)) && rb_is_red(parent))
	{ /* 循环结束的标志， 父母存在（非根结点），父母是红色（黑色不用处理 ）*/
		gparent = rb_parent(parent);

		if (parent == gparent->rb_left)
		{
			{
				/* uncle is red， 直接染色，然后 continue */
				register struct rb_node *uncle = gparent->rb_right;
				if (uncle && rb_is_red(uncle))
				{
					rb_set_black(uncle);
					rb_set_black(parent);
					rb_set_red(gparent);
					node = gparent;
					continue;
				}
			}

			if (parent->rb_right == node)
			{
				/* G P N 不在一条直线的情况， 旋转之后，N G 互换 */
				register struct rb_node *tmp;
				__rb_rotate_left(parent, root);
				tmp = parent;
				parent = node;
				node = tmp;
			}
			/* 一条直线的情况， 旋转之后染色，重新处理  */
			rb_set_black(parent);
			rb_set_red(gparent);
			__rb_rotate_right(gparent, root);
		} else {
			/* 镜面操作，不在解释 */
			{
				register struct rb_node *uncle = gparent->rb_left;
				if (uncle && rb_is_red(uncle))
				{
					rb_set_black(uncle);
					rb_set_black(parent);
					rb_set_red(gparent);
					node = gparent;
					continue;
				}
			}

			if (parent->rb_left == node)
			{
				register struct rb_node *tmp;
				__rb_rotate_right(parent, root);
				tmp = parent;
				parent = node;
				node = tmp;
			}

			rb_set_black(parent);
			rb_set_red(gparent);
			__rb_rotate_left(gparent, root);
		}
	}

	/* 根结点变为黑色，防止之前的操作导致根结点变为红色 */
	rb_set_black(root->rb_node);
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

{% hint style="info" %}
代码处理显然是经过分析的，即多种情况的转化图，这种编写习惯非常值得学习
{% endhint %}

### Deletion

对于红黑树来说，删除的操作可能不那么复杂，但是情况却有点多，这对于计算机来说不是难事，但是人理解起来不太简单，这也是计算机的优势了，善于无错的处理复杂情况。

删除的步骤，就是选择合适的位置（跟 BST 一样），删除之后再来调整。

![](../.gitbook/assets/image%20%2856%29.png)

![](../.gitbook/assets/image%20%2891%29.png)

上面说的很清楚拉，真心感激这个课件，省了我好多精力~

#### 最简单的情况，删除的节点是红色节点

之前也提到了，红色节点不会影响 RB trees 的平衡，所以删除它不会有任何影响

{% hint style="info" %}
注意，当与前继（或者后继节点）替换位置之后，考虑的颜色是替换之后的节点，原来的位置已经不再考虑范围了，下文在没有强调的时候，都是指替换之后的位置
{% endhint %}

#### 删除的节点是黑色节点，Double Black

![](../.gitbook/assets/image%20%28106%29.png)

![](../.gitbook/assets/image%20%28112%29.png)

当删除的节点是黑色的时候，就会出现 D-B 的情况，那么思想是什么呢，就是补偿，之前我们新插入一个节点，其实思想也是，只是补偿的是另外一个子树，想办法把节点移动到另外一个子树，来减少树高差，现在补偿的就是自己所在的子树了。

所以，整个思想都是，**从另外一条子树偷一个红色节点，然后转换为黑色**，这样总的黑色节点就没有少，而自己的子树也多了一个结点，从而维持红黑树的“平衡“。

![&#x603B;&#x7ED3;&#x7684;&#x597D;&#x68D2;](../.gitbook/assets/image%20%2876%29.png)

现在我们来考虑几种情况把。

#### Case 1， 兄弟是黑色，它的孩子有红色

为了简化描述，删除节点之后，接替在它的节点的那个位置我们称为 Child（C），其父母我们称为 P，然后 P 的另外一个孩子 Sibling（S），它的孩子我们就成为 Nephew（N）侄子。

![](../.gitbook/assets/image%20%2837%29.png)

对于这种，右侧是一条直线的，一个左旋即可解决，但是这里给的例子是 P为黑节点，其实它不一定是黑色，往往很有可能是红色。 看我画的示意图比较清楚

![&#x52A0;&#x7C97;&#x9ED1;&#x8272;&#xFF0C;&#x8868;&#x793A; D-B](../.gitbook/assets/image%20%2823%29.png)

由于菱形那一块 以及 P有可能是红色，所以旋转之后的必须要重新染色，怎么染色呢？答案是 S 变为 P 原来的颜色（即 S-&gt;color = P-&gt;color），然后 P 变为黑色，然后 S 的孩子，即 Nephew（N）变为黑色。

![&#x5DE6;&#x5B50;&#x6811; &#x767D;&#x8272; +2\*&#x9ED1; &#x53F3;&#x5B50;&#x6811; &#x767D;&#x8272; + 1\*&#x9ED1;](../.gitbook/assets/image%20%2842%29.png)

这样无论 P 究竟是红色 还有 黑色，都可以解决两边子树的黑色节点的问题，D-B 表示俩个黑节点，最后面，它所在的那一段，也仍然是 白色（可黑可红）+ 两个黑色节点，另外一个子树也一样。菱形呢？可以看出它上层原是  白色 + 黑，现在也是，所以这个调整非常正确。

{% hint style="info" %}
删除的难点，就在于删除之后这个重新染色，其他操作都是很自然的，值得一提的是，菱形那一块如果是红色的，其实包括了 俩个N都是红色的情况。 
{% endhint %}

#### Case 1.2 兄弟是黑色，它的孩子是红色，但不在一条直线

接着上面的例子，也就是说，右 N 侄子 是黑色，左 N 是红色。

![&#x867D;&#x7136;&#x8BFE;&#x4EF6;&#x4E0A;&#x662F;&#x7C89;&#x8272;&#xFF08;&#x53EF;&#x53D8;&#xFF09;&#xFF0C;&#x6211;&#x4EEC;&#x4E3E;&#x7684;&#x4F8B;&#x5B50;&#x5C31;&#x662F;&#x9ED1;&#x8272;](../.gitbook/assets/image%20%2873%29.png)

因为俩个 N 都是红色 刚才已经涉及了，现在讨论的是 P S N（Nephew）不在一条直线的情况。以课件的例子，就是， P S Z 不是在一条直线。

![&#x540C;&#x7406;&#xFF0C;&#x767D;&#x8272;&#x662F;&#x53EF;&#x9ED1;&#x53EF;&#x7EA2;](../.gitbook/assets/image%20%2881%29.png)

是否记得，我们在说插入的算法的时候，也谈到了不在一条直线的情况，解决方案就是旋转一次，转换为在一条直线的情况\( Case 1 \)。～

![right rotate](../.gitbook/assets/image%20%2884%29.png)

考虑，菱形所在的那条直线，是不是少了一个黑色节点，而 S 的右孩子（即 右 N）一定是黑色，否则我们也不会来到这一步了，所以，我们直接把红色变为黑色，S 变为红色，即可保持平衡，按之前的说法就是，S 和 右 N 之间还有一个位置可以放红色节点。

![](../.gitbook/assets/image%20%2868%29.png)

如此一来，转换为了在一条直线的情况了，只是有一点值得注意，菱形一定是黑色，这里就有读者自己思考把～

![](../.gitbook/assets/image%20%2872%29.png)

#### Case 2 兄弟为黑色节点，侄子也全为黑色

![](../.gitbook/assets/image%20%2817%29.png)

![](../.gitbook/assets/image%20%2888%29.png)

这种情况，课件上说的很明白了，另外一侧已经没有红色节点可以拿了，只能够向上寻找，如果 P 正好是红色，那我们就不需要往上，否则 P 变为 D-B。这种情况，在实际代码编写的时候，是要先于 Case 1 判断，因为向上传递之后，可能就又是 Case 1（包括  Case 1.2 ）。

![](../.gitbook/assets/image%20%28127%29.png)

#### Case 3 兄弟节点为红色

这也是最后一种情况了，但也是最先要判断的，如果我的兄弟都是红色，我自然不需要看我的侄子了，我的兄弟的红色就可以拿来使用。（这里隐含着 P 一定是黑色 ，N一定是黑色）

![](../.gitbook/assets/image%20%2875%29.png)

![](../.gitbook/assets/image%20%289%29.png)

先旋转，再重新着色，因为拿了另外一侧树的一个黑色节点，所以 现在 S 已经黑色，转换成了之前的 Cases，如果下一步发现新的 S（原来的侄子变为 S 了）的孩子都是黑色，那么只需要重新染色就好了，因为现在的 P 一定是红色。

{% hint style="info" %}
实际的转换是  Case 3 -&gt;{ Case 1 1.2, Case 2}，而 Case 2 -&gt; { 直接解决， Case 1 1.2  Case 2 Case 3 }，然后 Case 1.2 -&gt; Case 1
{% endhint %}

这给了实际编写代码的思路，流程图我就不画了，就是上面所说的过程，下面的 node 指的就是我所画的图的 Child（C），其实就是现在为 D-B 的那个节点，这个函数是找到了删除的位置，并且删除的节点是 D-B 这个函数才被调用的，只负责处理前面提到的 Cases。

```c
static void __rb_erase_color(struct rb_node *node, struct rb_node *parent,
			     struct rb_root *root)
{
	struct rb_node *other;

	while ((!node || rb_is_black(node)) && node != root->rb_node)
	{  /* Case 2 will propagate upward~ */
		if (parent->rb_left == node)
		{
			other = parent->rb_right;

			 /* sibling is red~  Case 3 */
			if (rb_is_red(other))
			{
				rb_set_black(other);
				rb_set_red(parent);
				__rb_rotate_left(parent, root);
				other = parent->rb_right; /* update sibling */
			}

			/* --- now sibling is black  ---*/

			/* two children are black( null included )  Case 2 */
			if ((!other->rb_left || rb_is_black(other->rb_left)) &&
			    (!other->rb_right || rb_is_black(other->rb_right)))
			{
				rb_set_red(other);
				node = parent;
				parent = rb_parent(node);
				/* Case 2 will propagate upward~ */
			}
			else
			{	/* anyone of them is red～ || both are red, Case 1 */
				if (!other->rb_right || rb_is_black(other->rb_right))
				{
					/* node 是左节点，so 处理 左侄子是 红色的情况 Case 1.2，*/
					rb_set_black(other->rb_left);
					rb_set_red(other);
					__rb_rotate_right(other, root);
					other = parent->rb_right; /* 更新 update  */
				}

				/* 三种情况都回到了这儿~, 右侄子是红色，parent不确定，other是黑色，node也是黑色 */
				rb_set_color(other, rb_color(parent));
				rb_set_black(parent);
				rb_set_black(other->rb_right);
				/* parent 之所以是黑色，是为了防止，左右侄子同时都为红色的情况，这个要死记 XXX */
				__rb_rotate_left(parent, root);
				/* 至此。 other 便为局部的 根结点 */


				node = root->rb_node; /* set black */
				break;
			}
		}
		else
		{ /* 以下都是镜面操作，上面是 left 这里就是 righ ， vice versus*/
			other = parent->rb_left;
			if (rb_is_red(other))
			{
				rb_set_black(other);
				rb_set_red(parent);
				__rb_rotate_right(parent, root);
				other = parent->rb_left;
			}
			if ((!other->rb_left || rb_is_black(other->rb_left)) &&
			    (!other->rb_right || rb_is_black(other->rb_right)))
			{
				rb_set_red(other);
				node = parent;
				parent = rb_parent(node);
			}
			else
			{
				if (!other->rb_left || rb_is_black(other->rb_left))
				{
					rb_set_black(other->rb_right);
					rb_set_red(other);
					__rb_rotate_left(other, root);
					other = parent->rb_left;
				}
				rb_set_color(other, rb_color(parent));
				rb_set_black(parent);
				rb_set_black(other->rb_left);
				__rb_rotate_right(parent, root);
				node = root->rb_node;
				break;
			}
		}
	}
	if (node)
		rb_set_black(node); /* 传递上去的 父母节点 设置为 黑色 */
}

```

上面的代码是处理若干 cases 的代码，下面就是我们之前说的，BST 删除节点的步骤了，很简单。

```c
void rb_erase(struct rb_node *node, struct rb_root *root)
{
	struct rb_node *child, *parent;
	int color;

	if (!node->rb_left)
		child = node->rb_right;
	else if (!node->rb_right)
		child = node->rb_left;
	else
	{
	/* find its successor（predecessor is also fine）*/
		struct rb_node *old = node, *left;

		node = node->rb_right;
		while ((left = node->rb_left) != NULL)
			node = left;

		if (rb_parent(old)) {
			/* 连接 node 与 删除结点的 parent */
			if (rb_parent(old)->rb_left == old)
				rb_parent(old)->rb_left = node;
			else
				rb_parent(old)->rb_right = node;
		} else
			root->rb_node = node; /* 被删除的节点是根结点 */

		/* node是successor 一定没有左 child */
		child = node->rb_right;

		parent = rb_parent(node);
		/* 此处正是开始删除节点的位置（转移后），记录它的孩子和父母 */
		color = rb_color(node);

		if (parent == old) {
			/* 意味着，被删除的节点的 successor 就是其 右节点（没有左节点 ） */
			parent = node;
		} else {
			if (child)
				rb_set_parent(child, parent);
			/* 删除的不是叶子节点，连接孙子和爷爷，此处被删除节点准备被抽出 */

			/* 被抽出后， 父母的左方就是孩子应该在的位置 前提是 successor不是其右节点*/
			/*
			 *      O (parent
			 *     /                           O
			 *    O （ successor） ->          /
			 *     \                         O
			 *      O (child, maybe null )
			 *
			 */
			parent->rb_left = child;
			node->rb_right = old->rb_right; /* 此处 assignment */
			rb_set_parent(old->rb_right, node);
		}

		/* 此处，将node的位置替换到原来要删除的位置 */
		node->rb_parent_color = old->rb_parent_color;
		node->rb_left = old->rb_left;
		/* 此处的情况可能是 successor是右节点，所以不用给右赋值 */
		rb_set_parent(old->rb_left, node);

		goto color;
	}

	/* 有一个是 null，或者都为 null 的情况*/
	parent = rb_parent(node);
	color = rb_color(node);

	if (child)
		rb_set_parent(child, parent);
	if (parent)
	{
		if (parent->rb_left == node)
			parent->rb_left = child;
		else
			parent->rb_right = child;
	}
	else
		root->rb_node = child; /* child 作为根节点，因为没有父母 */

 color:	/* 如果删除的节点是 黑色，否则不需要处理 */
	if (color == RB_BLACK)
		__rb_erase_color(child, parent, root);
}
```

## 末

红黑树的提出以及处理方案，处处体现着“补偿”的思想，多了就从少的子树拿，少了就从多的子树拿，再不济，就交给上层处理，大部分的 R-B 处理都是 Locally，不需要惊动整棵树，所以它非常的适合处理动态性强的结构，比如进程的调度队列，进程的内存空间管理以及set，map的实现等等。

实际代码的编写使得流程图的重要性得以体现，非常棒！

![](../.gitbook/assets/image%20%28117%29.png)

![](../.gitbook/assets/image%20%28124%29.png)

