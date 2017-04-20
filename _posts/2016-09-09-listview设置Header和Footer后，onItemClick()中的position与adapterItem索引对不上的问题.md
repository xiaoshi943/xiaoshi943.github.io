### 2016-09-09-listview设置Header和Footer后，onItemClick()中的position与adapterItem索引对不上的问题

​	当我们给ListView加了一个HeaderView后，onItemClick()方法里的`position`参数的值和我们列表中的Item下标对不上。比如我们给ListView添加了一个headerView，当点击列表中的第一条item时，我们期望的`position`是0，可是实际上却是1；再比如，如我们给ListView添加了两个headerView，`position`却是是2。也就是说，Adapter把HeaderView和FooterView也当成了Item，它是从Header开始计数的。

​	这就引发了一个问题：我们平时用ListView的时候，一般在Actiivty中存放一个数据源List<T> datas，将该datas传给adapter展示数据，当点击Item时，通过onItemClick()方法里的`position`取出datas中的相应数据。如果ListView设置了HeaderView，所点击的Item和position是对不上的，从datas取出的数据自然不对。

​	之所以会有这样的问题，是因为这样的用法本身就不对。想取到被点击的Item的相应数据不应该通过源数据datas来获得，而是要通过getAdapter().getItem(position)来获得。如下代码：

```java
public class ListViewTestActivity extends Activity implements OnItemClickListener{

	ListView listview;
	List<String> datas = new ArrayList<String>();
	ListViewAdapter adapter;
	
	@Override
	public void onCreate(Bundle savedInstanceState){
		super.onCreate(savedInstanceState);
		setContentView(R.layout.hotelbook_list);
		
		adapter = new ListViewAdapter(this , datas);
		listview = (ListView) findViewById(R.id.hotelbook_listview);
        //View view = getLayoutInflater().inflate(R.layout.display_test,null);
		//listview.addHeaderView(view,null,false);
		listview.setAdapter(adapter);
	}
	
	@Override
	public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
		//【1】不推荐的用法
		String item = datas.get(position);
		Toast.makeText(this, item, Toast.LENGTH_LONG).show();
		
		//【2】正确用法
		String item2 = (String) parent.getAdapter().getItem(position) ;
		Toast.makeText(this, item2, Toast.LENGTH_LONG).show();
	}
}
```

​	onItemClick()中的第一种用法我们不推荐，因为当ListView设置了HeaderView之后，onItemClick()中的`position`就和datas中的索引对不上了。当然，我们也可以计算一下有多少个HeaderView，然后用`position`减去HeaderView的个数来“更正”position的值，如下：

```java
	@Override
	public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
		//【1】错误的用法
        int hCount = listview.getHeaderViewsCount(); //获取HeaderView的个数
		String item = datas.get(position - hCount); //减去HeaderView的个数
		Toast.makeText(this, item, Toast.LENGTH_LONG).show();
		
	}
```

​	上面的做法看没有问题，但是如果datas中的数据被更改之后，而没有通知adapter更新，此时datas和adapter的数据也是对不上的。

​	如果是使用`parent.getAdapter().getItem(position)` 就不会出现上面的问题了，我们看一下ListView的源码。

```java
>> ListView.java >>

     public void addHeaderView(View v, Object data, boolean isSelectable) {
        final FixedViewInfo info = new FixedViewInfo();
        info.view = v;
        info.data = data;
        info.isSelectable = isSelectable;
        mHeaderViewInfos.add(info);
        mAreAllItemsSelectable &= isSelectable;

        // Wrap the adapter if it wasn't already wrapped.
        if (mAdapter != null) {
            if (!(mAdapter instanceof HeaderViewListAdapter)) {
                mAdapter = new HeaderViewListAdapter(mHeaderViewInfos, mFooterViewInfos, mAdapter);
            }
          
            if (mDataSetObserver != null) {
                mDataSetObserver.onChanged();
            }
        }
    }

    @Override
    public void setAdapter(ListAdapter adapter) {
        if (mAdapter != null && mDataSetObserver != null) {
            mAdapter.unregisterDataSetObserver(mDataSetObserver);
        }

        resetList();
        mRecycler.clear();
		
      	//如果ListView有HeaderView或者FooterView的话，则使用HeaderViewListAdapter对象来替代原来的			  dataper
        if (mHeaderViewInfos.size() > 0|| mFooterViewInfos.size() > 0) {
            mAdapter = new HeaderViewListAdapter(mHeaderViewInfos, mFooterViewInfos, adapter);
        } else {
            mAdapter = adapter;
        }
        ......
    }
```

​	我们看到在setAdapter()中做了一些处理：如果有HeaderView或者FooterView的话，则创建一个HeaderViewListAdapter来替代原来我们为ListView创建的adapter；如果没有，则使用原来的adapter。

**这里还有一点值得注意：**

> 添加HeaderView和FooterView必须在setAdapter()方法之前，否则setAdapter()中的mHeaderViewInfos和mFooterViewInfos的size是为0的，即认为没有HeaderView和FooterView。



​	下面我们看一下HeaderViewListAdapter有什么特别之处。

```java
public class HeaderViewListAdapter implements WrapperListAdapter, Filterable {

    private final ListAdapter mAdapter;

    ArrayList<ListView.FixedViewInfo> mHeaderViewInfos;
    ArrayList<ListView.FixedViewInfo> mFooterViewInfos;
  
  public HeaderViewListAdapter(ArrayList<ListView.FixedViewInfo> headerViewInfos,
                                 ArrayList<ListView.FixedViewInfo> footerViewInfos,
                                 ListAdapter adapter) {
    	//保存原来的adapter
        mAdapter = adapter;
        mIsFilterable = adapter instanceof Filterable;

    	//保存Header
        if (headerViewInfos == null) {
            mHeaderViewInfos = EMPTY_INFO_LIST;
        } else {
            mHeaderViewInfos = headerViewInfos;
        }

    	//保存Footer
        if (footerViewInfos == null) {
            mFooterViewInfos = EMPTY_INFO_LIST;
        } else {
            mFooterViewInfos = footerViewInfos;
        }

        mAreAllFixedViewsSelectable =
                areAllListInfosSelectable(mHeaderViewInfos)
                && areAllListInfosSelectable(mFooterViewInfos);
    }

    public int getHeadersCount() {
        return mHeaderViewInfos.size();
    }

    public int getFootersCount() {
        return mFooterViewInfos.size();
    }
  	......
    //item的总数包括Header和Footer的个数
    public int getCount() {
        if (mAdapter != null) {
            return getFootersCount() + getHeadersCount() + mAdapter.getCount();
        } else {
            return getFootersCount() + getHeadersCount();
        }
    }
  	......
    public Object getItem(int position) {
        // Header (negative positions will throw an IndexOutOfBoundsException)
        int numHeaders = getHeadersCount();
        if (position < numHeaders) { //如果position小于HeaderView的个数，说明需要获取的item是											Headerview
            return mHeaderViewInfos.get(position).data;
        }

        // Adapter
        final int adjPosition = position - numHeaders; //"矫正"position，即减去HeaderView的个数
        int adapterCount = 0;
        if (mAdapter != null) {
            adapterCount = mAdapter.getCount();
            if (adjPosition < adapterCount) { //如果"矫正"的position在初始dapter的item个数（即源数												  据datas个数）范围内，说明需要获取的Item是初始													adapter中的数据
                return mAdapter.getItem(adjPosition);
            }
        }

        // Footer (off-limits positions will throw an IndexOutOfBoundsException)
        return mFooterViewInfos.get(adjPosition - adapterCount).data; //如果既不是HeaderView也不																		是"正常"的Item，则剩下的就																	   是footerView了
    }
  	......
}
```

​	通过分析上面的源码，我们可以解答两个问题：

**（1）为什么添加HeaderView和FooterView之后`position`会变？**

​	当我们为ListView添加HeaderView或者FooterView后，ListView的setAdater()方法中会将一个HeaderViewListAdapter对象替换传进来的adapter对象。而HeaderViewListAdapter的getCount()方法中加上了HeaderView和FooterView的数目。

**（2）为什么通过`parent.getAdapter().getItem(position)`就能准确取到对应的Item？**

​	在HeaderViewListAdapter的getItem()方法中我们看到对position做了判断。

* 如果position的小于HeaderView的数目，则返回对应的Header条目的data。
* 如果第一个条件不满足，则有可能是命中原来adapter的条目。所以首先对position进行“矫正”，即position减去HeaderView的的数目，这样，position就和adapter的Item索引对应上了。然后判断矫正后的position是不是在adapter条目数的范围内，如果是，则返回相应的Item。
* 如果以上两个条件都不满足，说明position命中的是Footer。

