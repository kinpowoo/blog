#ListView嵌套EditText数据混乱和焦点问题完美解决

> 我们在工作中时常有这种需要，一个listview的子item里有多个edittext，edittext之间会抢点，而且listview holder的复用会导致edittext内容的显示混乱，下面，让我们来 fix it!


###首先让我们自定义一个EditText来管理它被添加的TextWatcher

	public class ExtendedEditText extends EditText 
	{   privateArrayList<TextWatcher>mListeners = null;   
	publicExtendedEditText(Context ctx)  
	{  super(ctx);   }  

	publicExtendedEditText(Context ctx,AttributeSetattrs)  
	{  super(ctx, attrs);   }   

	publicExtendedEditText(Context ctx,AttributeSetattrs,intdefStyle)
	{ super(ctx, attrs, defStyle);   }


	@Override
    public boolean onTouchEvent(MotionEvent event) {
        if(event.getAction()==MotionEvent.ACTION_UP){
        //当触摸控件时让edittext获取焦点
            setFocusable(true);
            setFocusableInTouchMode(true);
            requestFocus();
            requestFocusFromTouch();
        }

        return super.onTouchEvent(event);
    }


    @Override
    protected void onTextChanged(CharSequence text, int start, int lengthBefore, int lengthAfter) {
    //在edittext内容改变后将edittext的光标移到最尾端
        setSelection(lengthAfter);
        super.onTextChanged(text, start, lengthBefore, lengthAfter);
    }



	@Override  
	public void addTextChangedListener(TextWatcher watcher)  {    
       if (mListeners == null) { 
          mListeners = newArrayList<TextWatcher>(); 
        }
         mListeners.add(watcher);   
         super.addTextChangedListener(watcher);  
       } 



	@Override  
	public void removeTextChangedListener(TextWatcher watcher)   {
             if (mListeners != null){  
                inti = mListeners.indexOf(watcher); 
                if (i>= 0) { 
                 mListeners.remove(i);
                 } 
              }   
              super.removeTextChangedListener(watcher);  
         } 


	public void clearTextChangedListeners(){ 
		if(mListeners != null) {  
			for(TextWatcher watcher : mListeners){ 
    		super.removeTextChangedListener(watcher); 
    		}   
    	mListeners.clear(); 
    	mListeners = null; 
    	}   
    } 
	}
	
	
	
<br/>
##RecyclerView的效率比ListView要好，这里我用RecyclerView来代替ListView

</br>


	package com.example.kinpowoo.myapplication;

	import android.app.Activity;
	import android.content.Context;
	import android.os.Handler;
	import android.os.Message;
	import android.support.v7.widget.RecyclerView;
	import android.text.Editable;
	import android.text.TextWatcher;
	import android.util.Log;
	import android.view.LayoutInflater;
	import android.view.MotionEvent;
	import android.view.View;
	import android.view.ViewGroup;
	import android.widget.EditText;
	import android.widget.TextView;
	import android.widget.Toast;

	import java.util.List;


	/**
	* Created by kinpowoo on 01/01/18.
	*/

	public class RecycleAdapter extends RecyclerView.Adapter<RecycleAdapter.MyViewHolder> {

    List<Person> personList;
    Context context;
    LayoutInflater mInflate;
    int currentLine;
    Handler handler;

    public static final int NOTIFY_TV = 10086;
    public static final int NOTIFY_ET = 10087;
    public static final int NOTIFY_IV = 10088;

    public ListChanged listener;

    public void setListChanged(ListChanged listChanged){
        this.listener = listChanged;
    }


    public RecycleAdapter(List<Person> persons,Context context,Handler handler){
        this.personList = persons;
        this.context = context;
        this.mInflate = LayoutInflater.from(context);
        this.handler = handler;
    }


    @Override
    public MyViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {

        Log.i("log","viewType是什么，执行了么:"+viewType);
        View v = mInflate.inflate(R.layout.recycle_item,parent,false);
        MyViewHolder holder = new MyViewHolder(v);
        return holder;
    }

	public void updateAdapter(List<Person> list){
       this.personList = list;
       notifyDataSetChanged();
    }



    @Override
    public void onBindViewHolder(final MyViewHolder holder, int position, final List<Object> payloads) {
        Log.i("log","onBindviewHolder，执行了么:"+position);
        final int pos = holder.getAdapterPosition();
		//这里要用holder.getAdapterPosition()来取位置，它得到的位置是list数据中item的下标，比position要准确

        if (payloads.isEmpty()) {//为空，即不是调用notifyItemChanged(position,payloads)后执行的，也即在初始化时执行的
           onBindViewHolder(holder,pos);
            Log.i("log","onBindviewHolder，从不执行了么:"+position);
        } else if(payloads.size() > 0 && payloads.get(0) instanceof Integer){
            Log.i("log","onBindviewHolder，执行了么:"+position);
            //不为空，即调用notifyItemChanged(position,payloads)后执行的，可以在这里获取payloads中的数据进行局部刷新
            int type = (int) payloads.get(0);// 刷新哪个部分 标志位

            Person p = personList.get(pos);
            switch (type) {
                case NOTIFY_TV:
                    holder.name.setText(p.getName());//只刷新tv,
                    break;
                case NOTIFY_ET:
                    Log.i("log","刷新的Notify_ET，执行了么:"+position);
                    holder.age.setText(p.getAge()+"");
                    //只刷新et,这里容易出错，如果p.getAge()是数字型，不把它转成字符型，setText()就会去找这个string的id，
                    //而这个id的string我们没设，就会报错               
                    break;
                case NOTIFY_IV:

                    break;
            }


        }



    }


    @Override
    public void onBindViewHolder(final MyViewHolder holder, final int position) {
        final Person p = personList.get(position);
        holder.name.setText(p.getName());

        holder.gender.setText(p.getSex());

        holder.age.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View view, MotionEvent motionEvent) {
                new OriginKeyUtils((Activity) context,context,holder.age).showKeyboard();
                currentLine = position;
                return false;
            }
        });

		//在设值之前，先把它上面的textWatcher清空
        holder.age.clearTextChangedListeners();
        holder.age.setText(p.getAge()+"");
        //设置之后再加上
        holder.age.addTextChangedListener(new MyTextWatch(holder.age,null));
        
    }


    public interface ListChanged{
        public void listChange(int pos);
    }


    public List<Person> getList(){
        return personList;
    }




    public boolean isEmpty(Editable s){
        if(s.length()==0){
            return true;
        }
        return false;
    }


	//自定义一个TextWatcher类
    class MyTextWatch implements TextWatcher{
        TextView tv;          //传入EditText转为TextView后可以得到id,根据id判断是哪个EditText
        String lastChanged;   //记录editText变化之前的值
        DataChange dataChange;

        public MyTextWatch(EditText editable,DataChange change){
            if(editable instanceof TextView){
                this.tv = (TextView) editable;
            }
            this.dataChange = change;
        }

        @Override
        public void beforeTextChanged(CharSequence charSequence, int i, int i1, int i2) {
            lastChanged = charSequence.toString();
        }

        @Override
        public void onTextChanged(CharSequence charSequence, int i, int i1, int i2) {

        }

        @Override
        public void afterTextChanged(Editable editable) {
            if (editable != null&&!lastChanged.equals(editable.toString())) {
                switch (tv.getId()){
                    case R.id.age:
                Log.i("log",editable.toString());
                int beforeChange = personList.get(currentLine).getAge();
                Log.i("log","beforechange:"+beforeChange);

                int input = isEmpty(editable) ? 0 :Integer.valueOf(editable.toString());
                Log.i("log","input:"+input);
                if (input > 30) {
                    Toast.makeText(context, "年止于30", Toast.LENGTH_LONG).show();
                    //holder.age.setText(beforeChange+"");

                    personList.get(currentLine).setAge(beforeChange);
                    
                    //如何不符合小于30的条件，清空editable,
                    //并为其设改变之前的值
                    editable.clear();
                    editable.append(beforeChange+"");
                } else {
                    //holder.age.setText(editable);
                    personList.get(currentLine).setAge(input);
                }
                
				//值被修改后通知UI线程刷新视图
                 Message msg = new Message();
                 msg.what = 1;
                 msg.arg1 = currentLine;
                 handler.sendMessage(msg);
                 
                //也可以通过回调接口来通知
                if(dataChange!=null){
                    dataChange.dataChange();
                }
                break;
             }
            }
        }



    }

    public interface DataChange{
        public void dataChange();
    }


    @Override
    public int getItemCount() {
        return personList==null?0:personList.size();
    }

    class MyViewHolder extends RecyclerView.ViewHolder{

        public MyViewHolder(View v) {
            super(v);
            name = v.findViewById(R.id.username);
            age = v.findViewById(R.id.age);
            gender = v.findViewById(R.id.gender);
        }

        TextView name;
        ClearFocusEditText age;
        ClearFocusEditText gender;
    }
	}
	
	
	
<br/>


##最后来看看主页面要做些什么

<br/>

	public class EditTextFocus extends Activity{

    List<Person> list;
    RecyclerView listview;
    KeyboardView keyboardView;

    RecycleAdapter adapter;
    Handler handler = new Handler(){
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what){
                case 1:
                    int pos = msg.arg1;
                    //局部更新recyclerView,不用刷新整个list
                    adapter.notifyItemChanged(pos,RecycleAdapter.NOTIFY_ET);
                    break;
            }
        }
    };

    //Button  show;
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.edit_text_focus);

        listview= findViewById(R.id.list_view);

        keyboardView = findViewById(R.id.keyboard_view);

        list = new ArrayList<>();
        for(int i=0;i<30;i++){
            Person p = new Person("wang","male",20);
            list.add(p);
        }

        adapter = new RecycleAdapter(list,this,handler);
        listview.setLayoutManager(new LinearLayoutManager(this));
        adapter.setListChanged(new RecycleAdapter.ListChanged() {
            @Override
            public void listChange(int position) {
				//判断listview是否在计算layout，如果在此时滚动recyclerview,会有异常抛出，程序crash掉
                if(!listview.isComputingLayout()) {
                    List<Person> changed = adapter.getList();

                    adapter.updateAdapter(changed);
                }
            }
        });
        listview.setAdapter(adapter);

    	}
	}