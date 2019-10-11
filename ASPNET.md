# 案例：个人信息管理 #

## 需求分析 ##

1. 使用NVelocity的开发方式重写登录程序，把NVelocity封装成RenderTemplate方法。

2. 这种HttpHandler+ NVelocity的方式非常类似于PHP+smarty的开发方式，也有利于理解asp.net mvc。HttpHandler就是Controller，模板就是View， Controller不操作View的内部细节，只是把数据给View，由View自己控制怎样显示。

3. 字段：Id、姓名、年龄、个人网址（可以为空）。

4. 列表页面和编辑界面：PersonList.aspx、PersonEdit.aspx?Action=AddNew、

   PersonEdit.aspx?Action=Edit&Id=2（在PersonEdit页面判断是否传过来了save按钮来判断是加载还是保存。渲染页面的时候把Action和Id保存在隐藏字段中。保存成功后Redirect回List页面）

5. 进一步案例：有关联字段，比如班级，实现备注

## 代码实现： ##

### PersonList.html ###

```HTML
<head>
    <title>人员列表</title>
</head>
<body>
<a href="PersonEdit.ashx?Action=AddNew">新增人员</a>
<table>
    <thead>
        <tr><td>姓名</td><td>年龄</td><td>邮箱</td></tr>
    </thead>
    <tbody>
        #foreach($person in $Data)
        <tr><td>$person.Name</td><td>$person.Age</td><td>$person.Email</td></tr>
        #end
    </tbody>
</table>
</body>
</html>
```

### PersonEdit.html ###

```html
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
    <title></title>
</head>
<body>
<form action="PersonEdit.ashx" method="post">
    <input type="hidden" name="Action" value="AddNew" />
    <input type="hidden" name="Save" value="true" />
    <table>
        <tr><td>姓名：</td><td><input type="text" name="Name" value="$Data.Name" /></td></tr>
        <tr><td>年龄：</td><td><input type="text" name="Age" value="$Data.Age" /></td></tr>
        <tr><td>邮箱：</td><td><input type="text" name="Email" value="$Data.Email" /></td></tr>
        <tr><td></td><td><input type="submit" value="保存"/></td></tr>
    </table>
</form>
</body>
</html>
```

### PersonList.ashx ###

```C#
        public void ProcessRequest(HttpContext context)
        {
            context.Response.ContentType = "text/html";
            DataTable dt = SqlHelper.ExecuteDataTable("select * from T_Persons");
            //DataTable不是集合，所以无法foreach遍历，DataTable的Rows属性
            //代表表格中的数据行的集合（DataRow的集合），一般传递DataRowCollection
            //给模板方便遍历
            string html = CommonHelper.RenderHtml("PersonList.htm", dt.Rows);       
            context.Response.Write(html);
            //MVC：
        }
```

### PersonEdit.ashx ###

```C#
        public void ProcessRequest(HttpContext context)
        {
            context.Response.ContentType = "text/html";
            //PersonEdit.ashx?action=AddNew
            //PersonEdit.ashx?action=Edit&Id=3
            string action = context.Request["action"];
            if (action == "AddNew")
            {
                //判断是否含有Save并且等于true，如果是的话就说明是点击【保存】按钮请求来的
                bool save = Convert.ToBoolean(context.Request["Save"]);
                if (save)//是保存
                {
                    string name = context.Request["Name"];
                    int age = Convert.ToInt32(context.Request["Age"]);
                    string email = context.Request["Email"];
                    SqlHelper.ExecuteNonQuery("Insert into T_Persons(Name,Age,Email) 		values(@Name,@Age,@Email)",new SqlParameter("@Name",name)
                        , new SqlParameter("@Age", age)
                        , new SqlParameter("@Email", email));
                    context.Response.Redirect("PersonList.ashx");//保存成功返回列表页面
                }
                else
                {
                    //string html = CommonHelper.RenderHtml("PersonEdit.htm", new { Name = "", Age = 20, Email = "@inlett.com" });
                    var data = new { Name = "", Age = 20, Email = "@inlett.com" };
                    string html = CommonHelper.RenderHtml("PersonEdit.htm", data);
                    context.Response.Write(html);
                }
            }
            else if (action == "Edit")
            {
                
            }
            else
            {
                context.Response.Write("Action参数错误！");
            }
        }
```

### CommonHelper.cs ###

```C#
       /// <summary>
        /// 用data数据填充templateName模板，渲染生成html返回
        /// </summary>
        /// <param name="templateName"></param>
        /// <param name="data"></param>
        /// <returns></returns>
        public static string RenderHtml(string templateName, object data)
        {
             VelocityEngine vltEngine = new VelocityEngine();
            vltEngine.SetProperty(RuntimeConstants.RESOURCE_LOADER, "file");
            vltEngine.SetProperty(RuntimeConstants.FILE_RESOURCE_LOADER_PATH, System.Web.Hosting.HostingEnvironment.MapPath("~/templates"));//模板文件所在的文件夹
            vltEngine.Init();

            VelocityContext vltContext = new VelocityContext();
            vltContext.Put("Data", data);//设置参数，在模板中可以通过$data来引用

            Template vltTemplate = vltEngine.GetTemplate(templateName);
            System.IO.StringWriter vltWriter = new System.IO.StringWriter();
            vltTemplate.Merge(vltContext, vltWriter);

            string html = vltWriter.GetStringBuilder().ToString();
            return html;
        }
```

### SqlHelper.cs ###

```C#
        public static readonly string connstr =
            ConfigurationManager.ConnectionStrings["connstr"].ConnectionString;
        public static SqlConnection OpenConnection()
        {
            SqlConnection conn = new SqlConnection(connstr);
            conn.Open();
            return conn;
        }
        public static int ExecuteNonQuery(string cmdText, params SqlParameter[] parameters)
        {
            using (SqlConnection conn = new SqlConnection(connstr))
            {
                conn.Open();
                return ExecuteNonQuery(conn, cmdText, parameters);
            }
        }
        public static object ExecuteScalar(string cmdText,params SqlParameter[] parameters)
        {
            using (SqlConnection conn = new SqlConnection(connstr))
            {
                conn.Open();
                return ExecuteScalar(conn, cmdText, parameters);
            }
        }
        public static DataTable ExecuteDataTable(string cmdText,params SqlParameter[] parameters)
        {
            using (SqlConnection conn = new SqlConnection(connstr))
            {
                conn.Open();
                return ExecuteDataTable(conn, cmdText, parameters);
            }
        }
```

## 练习：学生管理系统 ##

### 数据库分析 ###

学生：姓名、性别、生日、身高、所属班级（select选择）、是否特长生
老师：姓名、电话、邮箱、生日
班级：班级名称、教室号、班主任老师（select选择）

### 功能 ###

每个页面顶部都有统一的菜单，分别进入学生管理、老师管理和班级管理
学生、老师、班级的增删改查。

（*）删除的时候并不是真的删除，而是软删除。

## 练习：留言板 ##

能够发表留言：标题、内容（多行普通文本）、昵称、是否匿名
展示留言列表（标题、内容、昵称、IP地址、日期）
发表留言界面和留言列表界面的头体是统一的。
（*）深入：增加留言的分页。SQLServer2005后增加了Row_Number函数简化实现。
限制结果集。返回第3行到第5行的数据（ ROW_NUMBER 不能用在where子句中，所以将带行号的执行结果作为子查询，就可以将结果当成表一样用了）：
select * from (select *,row_number() over (order by Id asc) as num from student) as s where s.num between 3 and 5

# 状态的传递和保存 #

## 通过Url传递 ##

两个页面之间传递数据最好、后续麻烦最少、最简单的方法就是通过Url传递。
案例：之间的增删改查页面
优点：简单，直接，明确知道发给谁，数据不会乱。缺点：如果多个页面或者不确定页面之间要传那么就需要每次跳转都带着；
不保密。

## 无状态Http ##

1. Http协议是无状态的，不会记得上次和网页“发生了什么”（故事《初恋50次》）。试验：private 字段++。服务器不记的上次给了浏览器什么，否则服务器的压力会太大，浏览器需要记住这些值，下次再提交服务器的时候（请在我的宽度基础上增加10，）就要把上次的值提交给服务器，让他想起来。如果要知道上一次的状态，一个方法是在对浏览器响应结束之前将状态信息保存到页面表单中（实现一下），下次页面再向服务器发出请求的时候带上这些状态信息，这样服务器就能根据这些状态信息还原上次的状态了，类似于去看病的病历本。
2. 尽量不要这么干，客户端的事情让客户端去做。
3. 状态信息保存在隐藏字段中的缺点：加大网站的流量、降低访问速度、机密数据放到表单中会有数据欺骗等安全性问题。故事：自行打印存折，因为余额不是写到存折这个隐藏字段中的，唯一的关联就是卡号。要把机密数据放到服务器，并且区别不同的访问者的私密区域，那么就要一个唯一的标识。

## Cookie ##

1. 如果想自由的传递和读取，用Cookie。Cookie是和站点相关的，并且每次向服务器请求的时候除了发送表单参数外，还会将和站点相关的所有Cookie都提交给服务器，是强制性的。Cookie也是保存在浏览器端的，而且浏览器会在每次请求的时候都会把和这个站点的相关的Cookie提交到服务器，并且将服务端返回的Cookie更新回数据库，因此可以将信息保存在Cookie中，然后在服务器端读取、修改。服务器返回数据除了普通的html数据以外，还会返回修改的Cookie，浏览器把拿到的Cookie值更新本地浏览器的Cookie就可以。看报文

2. 在服务器端控制Cookie案例，实现记住用户名的功能设置值的页面：

   Response.SetCookie(new HttpCookie("UserName",username));

   读取值的页面：username= Request.Cookies["UserName"].Value;

3. 如果不设定Expires那么生命周期则是关闭浏览器则终止，否则“最多”到Expires的时候终止。

4. Cookie的缺点：还不能存储过多信息，机密信息不能存。

5. Cookie无法跨不同的浏览器。

6. Cookie：是可以被清除，不能把不能丢的数据存到Cookie中；Cookie尺寸有限制，一般就是几K，几百K。

## Session原理 ##

1. 需要一种“服务器端的Cookie”：1、医生需要一个私人账本，记录病人编号和身份的对应关系；2、为了防止病人根据分配给他的编号猜测前后人的编号，那么需要一种“很难猜测”
2. Cookie不能存储机密数据。如果想储存据，可以保存一个Guid到Cookie中，然后在服务器中建立一个以Guid为Key，复杂数据为Value全局Dictionary。放到Application中。代码见备注※。这个Guid就是相当于用户的一个“令牌”
3. ASP.Net已经内置了Session机制，把上面的例子用ASP.NetSession重写。普通的HttpHandler要能够操作Session，要实现IRequiresSessionState接口。
4. Cookie是存在客户端，Session是存在服务器端，目的是一样的：保存和当前客户端相关的数据（当前网站的任何一个页面都能取到Session、Cookie）。
5. Session（会话）有自动销毁机制，如果一段时间内浏览器没有和服务器发生任何的交互，则Session会定时销毁。这也就是为什么一段时间不操作，系统就会自动退出。
6. Session实现登陆。

## Application（*） ##

1. Application是应用全局对象，被全体共享。操作之前先Lock，操作完成后UnLock。
2. 添加一个“全局应用程序类” Global.asax，当应用程序第一个页面被访问的时候Application_Start执行。
3. 举被很多书举烂了的例子“统计访问人数”，每次服务器上一个内容被访问的时候Application_BeginRequest会执行就把数量++。这样为什么不好？大并发访问会非常卡！
4. 做网站开发尽量不要用Application，也很少有需要用它的时候。

## 总结 ##

几种数据传递的区别和不同用途：

• Application：全局数据，不要用
• Url：精准传递，用起来麻烦
• 隐藏字段：不保密
• Cookie：保存在客户端，不保密
• Session：保存在服务器端，不能放大数据







