
1. **setClickable**:  设置为true时，表明控件可以点击，如果为false，就不能点击；“点击”适用于鼠标、键盘按键、遥控器等；
注意，setOnClickListener方法会默认把控件的setClickable设置为true。


2. **setEnabled**:  使能控件，如果设置为false，该控件永远不会活动，不管设置为什么属性，都无效；
设置为true，表明激活该控件，控件处于活动状态，处于活动状态，就能响应事件了，比如触摸、点击、按键事件等；
*setEnabled就相当于总开关一样，只有总开关打开了，才能使用其他事件。*

3. **setFocusable** 使能控件获得焦点，设置为true时，并不是说立刻获得焦点，要想立刻获得焦点，得用requestFocus；
使能获得焦点，就是说具备获得焦点的机会、能力，当有焦点在控件之间移动时，控件就有这个机会、能力得到焦点。

转载自：http://www.bubuko.com/infodetail-650092.html
