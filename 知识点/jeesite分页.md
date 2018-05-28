### jeesite分页

BaseEntity包含了Page对象  

page属性：

```Java
private int pageNo = 1; // 当前页码
private int pageSize = Integer.valueOf(Global.getConfig("page.pageSize")); // 页面大小，设置为“-1”表示不进行分页（分页无效）

private long count;// 总记录数，设置为“-1”表示不查询总数

private int first;// 首页索引
private int last;// 尾页索引
private int prev;// 上一页索引
private int next;// 下一页索引

private boolean firstPage;//是否是第一页
private boolean lastPage;//是否是最后一页

private int length = 8;// 显示页面长度
private int slider = 1;// 前后显示页面长度

private List<T> list = new ArrayList<T>();

private String orderBy = ""; // 标准查询有效， 实例： updatedate desc, name asc

private String funcName = "page"; // 设置点击页码调用的js函数名称，默认为page，在一页有多个分页对象时使用。

private String funcParam = ""; // 函数的附加参数，第三个参数值。

private String message = ""; // 设置提示消息，显示在“共n条”之后
```

Page构造方法：

```Java
/**
	 * 构造方法
	 * @param request 传递 repage 参数，来记住页码
	 * @param response 用于设置 Cookie，记住页码
	 * @param defaultPageSize 默认分页大小，如果传递 -1 则为不分页，返回所有数据
	 */
	public Page(HttpServletRequest request, HttpServletResponse response, int defaultPageSize){
		// 设置页码参数（传递repage参数，来记住页码）
		String no = request.getParameter("pageNo");
        //  获得当前页数
		if (StringUtils.isNumeric(no)){
			CookieUtils.setCookie(response, "pageNo", no);
			this.setPageNo(Integer.parseInt(no));
		}else if (request.getParameter("repage")!=null){
			no = CookieUtils.getCookie(request, "pageNo");
			if (StringUtils.isNumeric(no)){
				this.setPageNo(Integer.parseInt(no));
			}
		}
		// 设置页面大小参数（传递repage参数，来记住页码大小）
		String size = request.getParameter("pageSize");
		if (StringUtils.isNumeric(size)){
			CookieUtils.setCookie(response, "pageSize", size);
			this.setPageSize(Integer.parseInt(size));
		}else if (request.getParameter("repage")!=null){
			no = CookieUtils.getCookie(request, "pageSize");
			if (StringUtils.isNumeric(size)){
				this.setPageSize(Integer.parseInt(size));
			}
		}else if (defaultPageSize != -2){
			this.pageSize = defaultPageSize;
		}
		// 设置排序参数
		String orderBy = request.getParameter("orderBy");
		if (StringUtils.isNotBlank(orderBy)){
			this.setOrderBy(orderBy);
		}
	}
```

在创建Page对象后，设置在对象的Page属性，将操作结果返回Page对象，通过toString方法输出html，通过cookie发送页数。