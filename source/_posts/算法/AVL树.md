参考：https://www.cnblogs.com/skywang12345/p/3577479.html

## AVL树本质上还是一棵二叉搜索树，它的特点是：
1.本身首先是一棵二叉搜索树。
2.带有平衡条件：每个结点的左右子树的高度之差的绝对值（平衡因子）最多为1。
也就是说，AVL树，本质上是带了平衡功能的二叉查找树（二叉排序树，二叉搜索树）。

以下为四种不平衡的临界条件，分别为LL（leftleft）,LR（leftright）,RL,RR。
![](https://upload-images.jianshu.io/upload_images/7177220-83255f2429ce767e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
当每次到达这四种不平衡的临界条件时，都进行**旋转树**操作，那么树就能维持平衡：
```
private AVLTreeNode leftLeftRotation(AVLTreeNode node) {
        checkNull(node);

        AVLTreeNode  leftChild = node.left;
        node.left =  leftChild.right;
         leftChild.right = node;
        //reset the height
        node.height = max(height(node.left), height(node.right)) + 1;
         leftChild.height = max(height( leftChild.left), height( leftChild.right)) + 1;
        return  leftChild;
    }

    private AVLTreeNode rightRightRotation(AVLTreeNode node) {
        checkNull(node);

        AVLTreeNode  rightChild = node.right;
        node.right =  rightChild.left;
         rightChild.left = node;

        node.height = max(height(node.left), height(node.right)) + 1;
         rightChild.height = max(height( rightChild.left), height( rightChild.right)) + 1;
        return  rightChild;
    }

    private AVLTreeNode leftRightRotation(AVLTreeNode node) {
        node.left = rightRightRotation(node.left);//先对node.left旋转
        return leftLeftRotation(node);//后对自己旋转
    }

    private AVLTreeNode rightLeftRotation(AVLTreeNode node) {
        node.right = leftLeftRotation(node.right);
        return rightRightRotation(node);
    }
```

结合插入和删除，AVL树的实现：
```
package DataStructureAndAlgor;
/**
 * AVL树
 * 回顾的时候，最好把LL,RR,LR,RL旋转的四张图画出来，好理解。
 *
 * @param <T>
 */
public class AVLTree<T extends Comparable<T>> {

    class AVLTreeNode {
        T key;
        int height;
        AVLTreeNode left;
        AVLTreeNode right;

        public AVLTreeNode(T key, AVLTreeNode left, AVLTreeNode right) {
            this.key = key;
            this.left = left;
            this.right = right;
            this.height = 0;//初始化的结点的height都是0，因为都是作为叶子结点添加到树中的。
        }
    }

    private AVLTreeNode mRoot;


    private int height(AVLTreeNode tree) {
        if (tree != null) {
            return tree.height;
        }
        return 0;
    }

    public int height() {
        return height(mRoot);
    }

    public int height(T key) {
        AVLTreeNode node = search(mRoot, key);
        return height(node);
    }

    private int max(int a, int b) {
        return a > b ? a : b;
    }

    private AVLTreeNode leftLeftRotation(AVLTreeNode node) {
        checkNull(node);

        AVLTreeNode  leftChild = node.left;
        node.left =  leftChild.right;
         leftChild.right = node;
        //reset the height
        node.height = max(height(node.left), height(node.right)) + 1;
         leftChild.height = max(height( leftChild.left), height( leftChild.right)) + 1;
        return  leftChild;
    }

    private AVLTreeNode rightRightRotation(AVLTreeNode node) {
        checkNull(node);

        AVLTreeNode  rightChild = node.right;
        node.right =  rightChild.left;
         rightChild.left = node;

        node.height = max(height(node.left), height(node.right)) + 1;
         rightChild.height = max(height( rightChild.left), height( rightChild.right)) + 1;
        return  rightChild;
    }

    private AVLTreeNode leftRightRotation(AVLTreeNode node) {
        node.left = rightRightRotation(node.left);//先对node.left旋转
        return leftLeftRotation(node);//后对自己旋转
    }

    private AVLTreeNode rightLeftRotation(AVLTreeNode node) {
        node.right = leftLeftRotation(node.right);
        return rightRightRotation(node);
    }

    private void checkNull(AVLTreeNode node) {
        if (node == null) {
            throw new NullPointerException();
        }
    }

    /**
     * 将结点插入到AVL树种，并返回根节点。
     *
     * @param tree AVL树的根节点
     * @param key  待插入的结点的值。
     * @return 根节点
     */
    private AVLTreeNode insert(AVLTreeNode tree, T key) {
        if (tree == null) {//递归终点
            tree = new AVLTreeNode(key, null, null);
        } else {
            int cmp = key.compareTo(tree.key);//compare result

            if (cmp < 0) {
                tree.left = insert(tree.left, key);//递归插入
                //插入结点后，若AVLTree失去平衡，旋转树。
                if (height(tree.left) - height(tree.right) == 2) {//说明是tree结点出问题(必须是left-right，否则得负值)
                    if (key.compareTo(tree.left.key) < 0) {//这里如果写成tree.right.key会报空指针异常
                        tree = leftLeftRotation(tree);
                    } else {
                        tree = leftRightRotation(tree);
                    }
                }
            } else if (cmp > 0) {
                tree.right = insert(tree.right, key);//递归插入
                if (height(tree.right) - height(tree.left) == 2) {//说明是tree结点出问题(必须是left-right，否则得负值)
                    if (key.compareTo(tree.right.key) < 0) {//这里如果写成tree.left.key会报空指针异常
                        tree = rightLeftRotation(tree);
                    } else {
                        tree = rightRightRotation(tree);
                    }
                }
            } else {  //cmp==0;
                System.out.println("添加失败！不允许添加相同的结点！");
            }

        }

        tree.height = max(height(tree.left), height(tree.right)) + 1;

        return tree;
    }

    public void insert(T key) {
        mRoot = insert(mRoot, key);
    }

    /**
     * 删除树中的结点z, 返回根节点
     *
     * @param tree 从tree结点开始找寻要删除的结点z
     * @param z    待删除的根节点
     * @return 根节点
     */
    private AVLTreeNode remove(AVLTreeNode tree, AVLTreeNode z) {
        if (tree == null || z == null) {
            return null;
        }

        int cmp = z.key.compareTo(tree.key);
        if (cmp < 0) {//待删除的结点在tree的左子树
            tree.left = remove(tree.left, z);
            if (height(tree.right) - height(tree.left) == 2) {//如果删除后tree失去平衡, 进行调节
                AVLTreeNode r = tree.right;
                if (height(r.left) > height(r.right)) {
                    tree = rightLeftRotation(tree);
                } else {
                    tree = rightRightRotation(tree);
                }
            }
        } else if (cmp > 0) {//待删除的结点在tree的右子树
            tree.right = remove(tree.right, z);
            if (height(tree.left) - height(tree.right) == 2) {//失去平衡
                AVLTreeNode l = tree.left;
                if (height(l.left) < height(l.right)) {
                    tree = leftRightRotation(tree);
                } else {
                    tree = leftLeftRotation(tree);
                }
            }
        } else {//cmp==0,tree是要删除的结点
            if (tree.left != null && tree.right != null) {//tree的左右孩子非空
                if (height(tree.left) > height(tree.right)) {
                    /*
                    如果tree的左子树比右子树高，则
                    （1）找出tree的左子树中的最大结点
                    （2）将该最大结点的值赋值给tree
                     (3) 删除该最大结点 。
                     采用该方法的好处是：删除结点之后，AVL树仍然是平衡的。
                     */
                    AVLTreeNode max = maximum(tree.left);
                    tree.key = max.key;
                    tree.left = remove(tree.left, max);
                } else {
                    /*
                    如果tree的右子树比左子树高或相等，则
                    （1） 找出tree的右子树中的最小结点
                    （2） 将该最小结点的值赋值给tree
                     (3) 删除该最大结点
                    采用该方法的好处是：删除结点之后，AVL树仍然是平衡的。
                     */
                    AVLTreeNode min = minimum(tree.right);
                    tree.key = min.key;
                    tree.right = remove(tree.right, min);
                }
            } else {
                tree = (tree.left != null) ? tree.left : tree.right;
            }
        }

        if (tree != null) {//删除叶子结点的时候，这个tree是会返回null的。
            tree.height = max(height(tree.left), height(tree.right)) + 1;
        }


        return tree;
    }

    public void remove(T key) {
        AVLTreeNode z = search(mRoot, key);
        if (z != null) {
            mRoot = remove(mRoot, z);
        }
    }

    private void preOrderPrint(AVLTreeNode root, int depth) {
        if (root != null) {
            System.out.println();//换行
            for (int i = 0; i < depth; i++) {//for循环来打印value前的空格
                System.out.print("--");//结点深度越大，空格越多
            }
            System.out.print(root.key);
            depth++;
            preOrderPrint(root.left, depth);
            preOrderPrint(root.right, depth);
        }
    }

    public void print() {
        preOrderPrint(mRoot, 0);
    }


    private AVLTreeNode search(AVLTreeNode node, T key) {
        if (node == null) {
            return null;
        }
        int cmp = key.compareTo(node.key);
        if (cmp < 0) {
            return search(node.left, key);
        } else if (cmp > 0) {
            return search(node.right, key);
        } else {
            return node;
        }

    }

    private AVLTreeNode maximum(AVLTreeNode tree) {
        if (tree == null) {
            return null;
        }
        while (tree.right != null) {
            tree = tree.right;
        }
        return tree;
    }

    private AVLTreeNode minimum(AVLTreeNode tree) {
        if (tree == null) {
            return null;
        }
        while (tree.left != null) {
            tree = tree.left;
        }
        return tree;
    }

    public T maximum() {
        AVLTreeNode p = maximum(mRoot);
        if (p != null) {
            return p.key;
        }
        return null;
    }

    public T minimum() {
        AVLTreeNode p = minimum(mRoot);
        if (p != null) {
            return p.key;
        }
        return null;
    }
}
```
测试：
```
public class Main {
    public static void main(String[] args) {
        AVLTree<Integer> tree = new AVLTree<>();
        int arr[] = {33, 22, 4, 7, 0, 60, 13, 32, 21, 99, 6, 15, 27, 2};

        for (int i = 0; i < arr.length; i++) {
            tree.insert(arr[i]);
        }

        tree.print();
        System.out.println();
        System.out.println("树的根结点高度：" + tree.height());

        System.out.println("删除结点33,32,27,60,99");

        tree.remove(33);
        tree.remove(32);
        tree.remove(27);
        tree.remove(60);
        tree.remove(99);

        tree.print();
        System.out.println();
        System.out.println("树的根结点高度：" + tree.height());
    }
}
```
打印：
```
22
--7
----4
------0
--------2
------6
----15
------13
------21
--33
----32
------27
----60
------99
树的根结点高度：5
99
0
删除结点33,32,27,60,99

7
--4
----0
------2
----6
--15
----13
----22
------21
树的根结点高度：4
```


