```
import java.util.Iterator;

/**
 * 手动实现的一个ArrayList，封装了数组的一些常用操作**/
public class MyArrayList<T> implements Iterable<T> {
    //数组默认的长度为10
    private static final int DEFAULT_CAPACITY=10;
    //数组的大小
    private int size;
    //泛型T类型所代表的数组
    private T[] theItems;
    //构造方法用clear()来初始化数组
    public MyArrayList(){
        clear();
    }
    //初始化数组，让数组默认长度为10
    public void clear() {
        size=0;
        ensureCapacity(DEFAULT_CAPACITY);
    }
    //返回数组的大小
    public int size(){
        return size;
    }
    public boolean isEmpty(){
        return size()==0;
    }
    public void trimToSize(){
        ensureCapacity(size());
    }
    public T get(int index){
        if (index<0||index>size()){
            throw new ArrayIndexOutOfBoundsException();
        }
        return theItems[index];
    }
    public T set(int index,T newVal){
        if (index<0||index>=size){
            throw new ArrayIndexOutOfBoundsException();
        }
        T old=theItems[index];
        theItems[index]=newVal;
        return old;
    }
    //重新给数组分配长度，for循环是用来拷贝数据的
    private void ensureCapacity(int newCapacity) {
        if (newCapacity<size){
            return;
        }
        T[] old=theItems;
        theItems= (T[]) new Object[newCapacity];
        for (int i=0;i<size();i++){
            theItems[i]=old[i];
        }
    }
    public boolean add(T x){
        add(size(),x);
        return true;
    }
    //添加元素，for循环用于将index下标以及之后的元素
    //都往后移一步。
    public void add(int index,T x){
        if (theItems.length==size()){
            ensureCapacity(size()*2+1);
        }
        for(int i=size;i>index;i--){
            theItems[i]=theItems[i-1];
        }
        theItems[index]=x;

        size++;
    }
    //删除元素，for循环用于将要删除的元素的后一个下标的位置
    //一个一个往前移动，并用index+1下标的元素直接覆盖index下标的元素
    public T remove(int index){
        T removedItem=theItems[index];
        for (int i=index;i<size()-1;i++){
            theItems[i]=theItems[i+1];
        }
        size--;
        return removedItem;
    }

    @Override
    public Iterator<T> iterator() {
        return new ArrayListIterator();
    }

    private class ArrayListIterator implements Iterator<T>{
        private int current=0;

        @Override
        public boolean hasNext() {
            return current<size();
        }

        @Override
        public T next() {
            if (!hasNext()){
                throw new java.util.NoSuchElementException();
            }
            return theItems[current++];
        }

        @Override
        public void remove() {
            MyArrayList.this.remove(--current);
        }
    }
}
```
