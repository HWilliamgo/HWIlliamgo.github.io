```
public class MyLinkedList<T> implements Iterable<T>{
    //链表大小
    private int size;
    //修改次数
    private int modCount=0;
    //链表头结点
    private Node<T> beginMarker;
    //链表尾结点
    private Node<T> endMarker;
    //构造方法中清除链表
    public MyLinkedList(){
        clear();
    }

    @Override
    public Iterator<T> iterator() {
        return new LinkedListIterator();
    }

    /**
     * Node就是结点
     * @param <T>传入的泛型参数
     */
    private static class Node<T>{
        public T data;
        public Node<T> previous;
        public Node<T> next;
        public Node(T data,Node<T> previous,Node<T> next){
            this.data=data;
            this.previous=previous;
            this.next=next;
        }
    }

    /**
     * clear(),清除链表，重新申请头尾结点，并让头尾结点相连。
     */
    public void clear(){
        beginMarker=new Node<>(null,null,null);
        endMarker=new Node<>(null,beginMarker,null);
        beginMarker.next=endMarker;

        size=0;
        modCount++;
    }
    //直接返回链表大小
    public int size(){
        return size;
    }
    //判断链表是否是空，注意！只要链表长度为0就是空链表，链表的头结点和尾结点不算。
    public boolean isEmpty(){
        return size==0;
    }
    public boolean add(T x){
        add(size(),x);
        return true;
    }
    public void add(int index,T x){
        addBefore(getNode(index),x);
    }
    public T get(int index){
        return getNode(index).data;
    }

    /**
     * 在index处找到对应的Node,将新值赋值给Node的data。
     * @param index 要重新赋值的位置
     * @param newVal 传入的新的数据
     * @return 返回旧的数据
     */
    public T set(int index,T newVal){
        Node<T> p=getNode(index);
        T oldVal=p.data;
        p.data=newVal;
        return oldVal;
    }
    public T remove(int index){
        return remove(getNode(index));
    }

    /**
     * 添加一个数据x到结点p的前面，即在结点p前面生成一个新的结点newNode
     * newNode的头指针指向p结点原先头指针指向的结点的尾指针：p.previous.next
     * newNode的尾指针指向p结点的头指针：p.previous
     * @param p 将数据装入结点Node中，并放到结点p的前面
     * @param x 数据
     */
    private void addBefore(Node<T> p,T x){
        Node<T> newNode=new Node<>(x,p.previous,p);
        newNode.previous.next=newNode;
        p.previous=newNode;
        size++;
        modCount++;
    }

    /**
     *将p结点的头指针指向的结点赋值给p结点的尾指针指向的结点的头指针，
     *将p结点的尾指针指向的结点赋值给p结点的头指针指向的结点的尾指针
     *
     *即：将p结点的前结点的尾和p结点的后结点的头相连。
     * @param p 要删除的结点
     */
    private T remove(Node<T> p){
        p.next.previous=p.previous;
        p.previous.next=p.next;
        size--;
        modCount++;

        return p.data;
    }

    /**
     * 判断index的位置是在链表前半部分还是后半部分。
     * 若在前半部分，就从头结点开始往后遍历，直到遍历到index的位置的时候，返回该处的Node；
     * 若在后半部分，就从尾结点开始往前遍历，直到遍历到index的位置的时候，返回该处的Node。
     * @param index 要取得结点的位置
     * @return
     */
    private Node<T> getNode(int index){
        Node<T> p;
        if (index<0||index>size()){
            throw new IndexOutOfBoundsException();
        }
        if (index<size()/2){
            p=beginMarker.next;
            for (int i=0;i<index;i++){
                p=p.next;
            }
        }else {
            p=endMarker;
            for (int i=size();i>index;i--){
                p=p.previous;
            }
        }
        return p;
    }
    private class LinkedListIterator implements Iterator<T>{
        private Node<T> current=beginMarker.next;
        private int expectedModCount=modCount;
        private boolean okToRemove=false;

        @Override
        public boolean hasNext() {
            return current!=endMarker;
        }

        @Override
        public T next() {
            if (modCount!=expectedModCount){
                throw new ConcurrentModificationException();
            }
            if (!hasNext()){
                throw new NoSuchElementException();
            }
            T nextItem=current.data;
            current=current.next;
            okToRemove=true;
            return nextItem;
        }

        @Override
        public void remove() {
            if(modCount!=expectedModCount){
                throw new ConcurrentModificationException();
            }
            if (!okToRemove){
                throw new IllegalStateException();
            }
            MyLinkedList.this.remove(current.previous);
            okToRemove=false;
            expectedModCount++;
        }
    }

}
```
