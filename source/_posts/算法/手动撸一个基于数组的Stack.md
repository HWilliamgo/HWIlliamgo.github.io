```

public class MyStack<T> {
    private T t;
    private int size;
    private T[] items;
    private int current;
    private static final int DEFAUL_CAPACITY = 10;
    //构造函数清除并确认stack初始容量为10
    public MyStack() {
        clear();
        ensureCapacity(DEFAUL_CAPACITY);
    }
    //将stack的容量剪裁到size大小
    public void trimToSize() {
        ensureCapacity(size());
    }

    public int size() {
        return size;
    }

    //清空栈，并让指针指到-1的地方。
    public void clear() {
        this.size = 0;
        current = -1;
    }

    /**
     * 重新确认stack的容量，并将stack中数据拷贝。
     * @param newCapacity 新的容量
     */
    public void ensureCapacity(int newCapacity) {
        if (newCapacity < size) {
            System.out.println("新的容量小于已存在的数组的容量");
            return;
        }
        T[] oldItems = items;
        items = (T[]) new Object[newCapacity];
        for (int i = 0; i < size; i++) {
            items[i] = oldItems[i];
        }
    }

    public boolean isEmpty() throws Exception {
        if (current == 0) {
            return true;
        } else if (current > 0) {
            return false;
        } else {
            throw new Exception("the pointer of the stack is below 0");
        }
    }

    public boolean isFull() throws Exception {
        if (current > 0) {
            return current == size() - 1;
        } else {
            return false;
        }
    }

    public T pop() throws Exception {
        if (isEmpty()) {
            throw new Exception("the stack is empty now");
        }
        size--;
        return items[current--];
    }

    public T top() throws Exception {
        if(isEmpty()){
            throw new Exception("the stack is empty now");
        }
        return items[current];
    }
    public void push(T x) throws Exception {
        if (isFull()) {
            ensureCapacity(size() * 2);
        }
        items[++current] = x;
        size++;
    }
}
```
---
入口函数测试：
```
public class Main {
    public static void main(String []args) throws Exception {
        MyStack<Integer> stack=new MyStack<>();
        stack.push(10);
        stack.push(9);
        stack.push(8);
        stack.push(7);
        stack.push(6);
        stack.push(5);
        stack.push(4);
        stack.push(3);
        stack.push(2);
        stack.push(1);
        stack.push(0);
        stack.push(-1);
        stack.push(-2);
        System.out.println("The size of the stack is "+stack.size());
        int a=stack.pop();
        System.out.println("The size of the stack is "+stack.size());
        int b=stack.top();
        System.out.println("a ="+a);
        System.out.println("The top of the stack is "+b);
    }
}
```
打印：
```
The size of the stack is 13
The size of the stack is 12
a =-2
The top of the stack is -1
```
