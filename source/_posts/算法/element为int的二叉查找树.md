参照着《数据结构java语言描述》的伪码，自己实现的方法细节，最复杂的一个方法是`remove()`
``` java
package DataStructure.Tree;

public class IntTreeBag implements Cloneable {

    private Node root;

    public class Node {
        Node(int element, Node left, Node right) {
            this.element = element;
            this.left = left;
            this.right = right;
        }
        public Node getLeft() {
            return left;
        }
        public Node getRight() {
            return right;
        }
        private int element;
        private Node left;
        private Node right;

        @Override
        public String toString() {
            return String.valueOf(element);
        }
    }

    public IntTreeBag() {
        this(0);
    }

    public IntTreeBag(int rootValue) {
        root = new Node(rootValue, null, null);
    }

    public int size() {
        return size(root);
    }

    private int size(Node node) {
        if (node == null) {
            return 0;
        } else {
            return 1 + size(node.left) + size(node.right);
        }
    }
    public Node getRoot() {
        return root;
    }


    public void add(int target) {
        boolean done = false;
        Node cursor = root;

        while (!done) {
            if (target < cursor.element) {
                if (cursor.left == null) {
                    cursor.left = new Node(target, null, null);
                    done = true;
                } else {
                    cursor = cursor.left;
                }
            } else if (target > cursor.element) {
                if (cursor.right == null) {
                    cursor.right = new Node(target, null, null);
                    done = true;
                } else {
                    cursor = cursor.right;
                }
            } else {//target==cursor.element
                //查找二叉树还是暂时不做有相同结点的处理。
//                if (cursor.left == null) {
//                    cursor.left = new Node(target, null, null);
//                    done = true;
//                } else {
//                    cursor = cursor.left;
//                }
                done=true;
            }
        }
    }

    /**
     * 删除值为target的目标结点，巧妙地记录下游标结点的父结点，并判断
     * 当前游标在左子树还是右子树，然后操作游标结点的父结点即可容易删除目标。
     *
     * 删除操作将目标分成5种类型来操作，分别是：
     * 目标是根节点，目标左右结点为空，目标仅左结点为空，
     * 目标仅右结点为空，目标左右结点非空
     *
     * @param target 目标
     * @return 如果成功删除返回true，否则返回false
     */
    public boolean remove(int target) {
        if (isEmpty()) {
            return true;
        }

        Node parentOfCursor = null;//cursor的父结点
        Node cursor = root;
        boolean inLeft=false;//记录cursor是在左结点还是在右结点
        boolean done = false;

        while (!done) {
            if (cursor==null){//没有找到target
                return false;
            }
            if (target < cursor.element) {
                parentOfCursor = cursor;
                cursor = cursor.left;
                inLeft=true;
            } else if (target > cursor.element) {
                parentOfCursor = cursor;
                cursor = cursor.right;
                inLeft=false;
            } else {//target == cursor.element
                if (cursor == root) {//cursor就是根结点, cursor没有向下搜索过, inLeft和parentOfCursor都还是初始值
                    if (cursor.left == null) {
                        root = root.right;
                        done=true;
                    } else {//root.left!=null
                        if (root.left.right == null) {
                            root.left.right = root.right;
                            root = root.left;
                            done=true;
                        } else {
                            root.element = getRightMostNode(root.left).element;
                            removeRightMostNode(root.left);
                            done=true;
                        }
                    }
                } else if (cursor.left == null && cursor.right == null) {//左右结点都为空
                    if (inLeft){
                        parentOfCursor.left=null;
                        done=true;
                    }else {
                        parentOfCursor.right=null;
                        done=true;
                    }
                }else if (cursor.left == null){//左结点为空，右结点不为空
                    if (inLeft){
                        parentOfCursor.left=cursor.right;
                        done=true;
                    }else {
                        parentOfCursor.right=cursor.right;
                        done=true;
                    }
                }else if (cursor.right == null){//右结点为空，左结点不为空
                    if (inLeft){
                        parentOfCursor.left=cursor.left;
                        done=true;
                    }else {
                        parentOfCursor.right=cursor.left;
                        done=true;
                    }
                }else {//左右两个子结点都不为空
                    if (cursor.left.right==null){
                        cursor.left.right=cursor.right;
                        if (inLeft){
                            parentOfCursor.left=cursor.left;
                            done=true;
                        }else {
                            parentOfCursor.right=cursor.left;
                            done=true;
                        }
                    }else {
                        cursor.element=getRightMostNode(cursor.left).element;
                        removeRightMostNode(cursor.left);
                        done=true;
                    }
                }
            }
        }
        return true;//能跳出while循环就是done=true，成功删除了。

    }

    public boolean isEmpty() {
        return root == null;
    }

    /**
     * remove the most left node of the rootNode
     *
     * @param rootNode .
     * @return return true if remove target. return
     * false if the left of rootNode is null.
     */
    public boolean removeLeftMostNode(Node rootNode) {
        Node c = rootNode;
        Node parentOfc = null;
        while (c.left != null) {
            parentOfc = c;
            c = c.left;
        }
        if (parentOfc != null) {//rootNode.left!=null
            parentOfc.left = null;
            return true;
        } else {
            //rootNode.left==null.
            return false;
        }
    }

    public boolean removeRightMostNode(Node rootNode) {
        Node c = rootNode;
        Node parentOfc = null;
        while (c.left != null) {
            parentOfc = c;
            c = c.right;
        }
        if (parentOfc != null) {
            parentOfc.right = null;
            return true;
        } else {
            return false;
        }
    }

    public Node getLeftMostNode(Node rootNode) {
        Node cursor = rootNode;
        while (cursor.left != null) {
            cursor = cursor.left;
        }
        return cursor;
    }

    public Node getRightMostNode(Node rootNode) {
        Node cursor = rootNode;
        while (cursor.right != null) {
            cursor = cursor.right;
        }
        return cursor;
    }

    public int countOccurences(int target) {
        int count = 0;
        Node cursor = root;
        while (cursor != null) {
            if (target < cursor.element) {
                cursor = cursor.left;
            } else if (target > cursor.element) {
                cursor = cursor.right;
            } else if (target == cursor.element) {
                cursor = cursor.left;
                count++;
            }
        }
        return count;
    }

    @Override
    protected Object clone() throws CloneNotSupportedException {
        super.clone();
        return treeCopy(root);
    }

    //深度拷贝一个树，返回拷贝的全新的树的根节点。
    private Node treeCopy(Node node) {
        Node left, right;
        if (node == null) {
            return null;
        } else {
            left = treeCopy(node.left);
            right = treeCopy(node.right);
            return new Node(node.element, left, right);
        }
    }

    /**
     * 先序遍历打印
     * @param node 根节点
     */
    public void preorderPrint(Node node){
        if (node!=null){
            System.out.println(node.element);
            if (node.left!=null){
                preorderPrint(node.left);
            }
            if (node.right!=null){
                preorderPrint(node.right);
            }
        }
    }

    /**
     * 添加一棵树，传入根结点即可
     * 因为内部调用了add()，因此会过滤掉相同element的结点。
     * @param node root of the tree to be added
     */
    public void addAll(Node node){
        if (node!=null){
            this.add(node.element);
            if (node.left!=null){
                addAll(node.left);
            }
            if (node.right!=null){
                addAll(node.right);
            }
        }
    }
    public void addMany(int...elements){
        for (int i=0;i<elements.length;i++){
            add(elements[i]);
        }
    }

    /**
     * 按从小到大的顺序打印，其实是中序遍历
     * @param node 根节点
     */
    public void printInSequence(Node node){
        if (node!=null){
            if (node.left!=null){
                printInSequence(node.left);
            }
            System.out.print(node.element+" ");
            if (node.right!=null){
                printInSequence(node.right);
            }
        }
    }

    /**
     * 倒序打印，即从大到小打印树，也是中序遍历，
     * 不过是先右结点，根节点，再左结点
     * @param node
     */
    public void printInReverseSequence(Node node){
        if (node!=null){
            if (node.right!=null){
                printInReverseSequence(node.right);
            }
            System.out.print(node.element+" ");
            if (node.left!=null){
                printInReverseSequence(node.left);
            }
        }
    }



}

```
