---
title: 从模板方法模式到JDBCTemplate
date: 2019-05-22 22:33:24:024
tags: [Java] 
published: true
feature: https://user-gold-cdn.xitu.io/2019/5/21/16ad9448cb69de5e?imageView2/0/w/1280/h/960/ignore-error/1
---
> 本文转载自 [https://juejin.im/post/5cdb7f4af265da037371aa1f](https://juejin.im/post/5cdb7f4af265da037371aa1f) 
将大象装进冰箱需要三步，那么老虎了？如何优雅的将大象装进冰箱？

### 把大象装进冰箱

| Step | 大象 | 老虎 | ... |
|:--:|:--:|:--:|:--: |
| First | 打开冰箱门 | 打开冰箱门 | 打开冰箱门 |
| Second | 把大象放进去 | 把老虎放进去 | ... |
| Third | 关闭冰箱门 | 关闭冰箱门 | 关闭冰箱门 |

***大象类***
```
public class Elephant {
        public void putRefrigerator() {
            openDoor();
            putElephant();
            closeDoor();
        }
        public void openDoor() {
            System.out.println("open the door");
        }
        public void putElephant() {
            System.out.println("put in the Elephant");
        }
        public void closeDoor() {
            System.out.println("close the door");
        }
    }
```

***老虎类***

```
public class Tiger {
        public void putRefrigerator() {
            openDoor();
            putTiger();
            closeDoor();
        }
        public void openDoor() {
            System.out.println("open the door");
        }
        public void putTiger() {
            System.out.println("put in the Tiger");
        }
        public void closeDoor() {
            System.out.println("close the door");
        }
    }
```

可以看出我们将大象和老虎放进冰箱的过程中出现了大量的重复代码，这显然不是一个好的设计，如果我们在以后的系统升级过程中需要再放入长颈鹿怎么办，我们应该如何从我们的设计中删除这些重复代码？通过观察我们发现放大象和放老虎之间有很多共同点，都需要进行开关门的操作，只是放的过程不尽相同，我们是否可以将共同点抽离？我们一起试试看

***抽象超类***
```
public abstract class AbstractPutAnyAnimal {
        //这是一个模板方法，它是一个算法的模板，描述我们将动物放进冰箱的步骤，每一个方法代表了一个步骤
        public void putRefrigerator() {
            openDoor();
            putAnyAnimal();
            closeDoor();
        }
        //在超类中实现共同的方法，由超类来处理
        public void openDoor() {
            System.out.println("open the door");
        }
        public void closeDoor() {
            System.out.println("close the door");
        }
        //每个子类可能有不同的方法,我们定义成抽象方法让子类去实现
        abstract void putAnyAnimal();
    }
```

***大象类***

```
public class Elephant extends AbstractPutAnyAnimal {
        //子类实现自己的业务逻辑
        @Override
        void putAnyAnimal() {
            System.out.println("put in the Elephant");
        }
    }
```

***老虎类***

```
public class Tiger extends AbstractPutAnyAnimal {
        //子类实现自己的业务逻辑
        @Override
        void putAnyAnimal() {
            System.out.println("put in the Tiger");
        }
    }
```

通过将相同的方法抽离到超类中，并定义一个抽象方法供子类提供不同的实现，事实上我们刚刚实现了一个模板方法模式。

### 模板方法模式定义？

**模板方法模式定义了一个算法的步骤，并允许子类为一个或多个步骤提供实现**，putRefrigerator 方法定义了我们将大象装进冰箱的步骤它就是一个模板方法。**模板方法模式在一个方法中定义一个算法的骨架，而将一些步骤延迟到子类中，模板方法使得子类可以不在改变算法结构的情况下，重新定义算法的某些步骤**（子类提供自己的实现）

### 模板方法模式中的钩子

我们可以在超类中定义一个空方法，我们称这种方法为钩子（hook）。子类可以依据情况选择覆盖，钩子的存在可以让子类有能力对算法的不同点进行挂载；**钩子可以让子类实现算法中的可选部分，钩子也可以让子类为抽象类做一些决定**我们将大象装进冰箱后可能会想调整冰箱温度，也可能什么都不做使用默认温度，我们可以通过定义一个钩子，让子类来选择是否调整温度，如下：

***抽象父类***
```
public abstract class AbstractPutAnyAnimal {
        public void putRefrigerator() {
            openDoor();
            putAnyAnimal();
            closeDoor();
            //默认为false,重新这个方法决定是否执行addTemperature();方法
            if (isAdd()) {
                addTemperature();
            }
        }
        public void openDoor() {
            System.out.println("open the door");
        }
        public void closeDoor() {
            System.out.println("close the door");
        }
        abstract void putAnyAnimal();
        void addTemperature(){
            System.out.println("plus one");
        };
        //定义一个空实现，由子类决定是否对其进行实现
        boolean isAdd(){
            return false;
        }
    }
```

***大象类***

```
public class Elephant extends AbstractPutAnyAnimal {
        @Override
        void putAnyAnimal() {
            System.out.println("put in the Elephant");
        }
        //子类实现钩子方法
        @Override
        boolean isAdd() {
            return true;
        }
    }
```

我们通过定义一个钩子方法，子类选择是否实现这个钩子方法，来决定是否调整温度；当然钩子方法的用途不止如此，**它还能让子类有机会对模板中即将发生或刚刚发生的步骤做出反应**，这在JDK中有很多的例子，甚至在前端开发领域也有很多例子，我就不具体展开代码演示了，后面在模板方法模式的更多应用中展开。

JDK以及Spring中使用了很多的设计模式，下面我们通过比较传统JDBC编程和JDBCTemplate来看看模板方法模式是如何帮我们消除样板代码的

### 传统JDBC编程

***JDBC编程之新增***
```
String driver = "com.mysql.jdbc.Driver";
        String url = "jdbc:mysql://localhost:3306/test?characterEncoding=UTF-8&amp;useSSL=true";
        String username = "root";
        String password = "1234";
        Connection connection = null;
        Statement statement = null;
        try {
            Class.forName(driver);
            connection = DriverManager.getConnection(url, username, password);
            String sql = "insert into users(nickname,comment,age) values('小小谭','I love three thousand times', '21')";
            statement = connection.createStatement();
            int i = statement.executeUpdate(sql);
            return i;
        } catch (SQLException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } finally {
            try {
                if (null != statement) {
                    statement.close();
                }
                if (null != connection) {
                    connection.close();
                }
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
        }
        return 0;
```

***JDBC编程之查询***

```
String driver = "com.mysql.jdbc.Driver";
        String url = "jdbc:mysql://localhost:3306/test?characterEncoding=UTF-8&amp;useSSL=true";
        String username = "root";
        String password = "1234";
        Connection connection = null;
        Statement statement = null;
        try{
            Class.forName(driver);
            connection = DriverManager.getConnection(url, username, password);
            String sql = "select nickname,comment,age from users";
            statement = connection.createStatement();
            ResultSet resultSet = statement.executeQuery(sql);
            List<Users> usersList = new ArrayList<>();
            while (resultSet.next()) {
                Users users = new Users();
                users.setNickname(resultSet.getString(1));
                users.setComment(resultSet.getString(2));
                users.setAge(resultSet.getInt(3));
                usersList.add(users);
            }
            return usersList;
        } catch (SQLException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } finally {
            try {
                if (null != statement) {
                    statement.close();
                }
                if (null != connection) {
                    connection.close();
                }
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
        }
        return null;
```

上面给出了我们在传统JDBC编程中的两个案例，可以看到传统JDBC的很多缺点，当然在实际项目中我们可能不会这么原始的进行数据库开发，可能会对JDBC进行一定的封装，方便我们的使用。Spring 官方为了简化JDBC的开发也发布了JDBCTemplate，下面我们就看一下它是如何简化开发的，以及模板方法模式在其中的应用

### JDBCTemplate是个啥，它到底简化了什么？

从JDBCTemplate的名字我们就不难看出，它简化了我们JDBC的开发，而且很可能大量应用了模板方法模式，它到底为我们提供了什么？**它提供了与平台无光的异常处理机制**。使用过原生JDBC开发的同学可能有经历，几乎所有的操作代码都需要我们强制捕获异常，但是在出现异常时我们往往无法通过异常读懂错误。Spring解决了我们的问题它**提供了多个数据访问异常，并且分别描述了他们抛出时对应的问题，同时对异常进行了包装不强制要求我们进行捕获，同时它为我们提供了数据访问的模板化**，从上面的传统JDBC编程我们可以发现，很多操作其实是重复的不变得比如事务控制、资源的获取关闭以及异常处理等，同时结果集的处理实体的绑定，参数的绑定这些东西都是特有的。因此**Spring将数据访问过程中固定部分和可变部分划分为了两个不同的类(Template)和回调(Callback),模板处理过程中不变得部分，回调处理自定义的访问代码**；下面我们具体通过源码来学学习一下

### 模板方法模式在JDBCTemplate中的应用

我所使用的版本是5.1.5.RELEASE

打开JdbcTemplate类(我这里就不截图了，截图可能不清晰我直接将代码copy出来)：

***JdbcTemplate***
```
public class JdbcTemplate extends JdbcAccessor implements JdbcOperations {
        //查询前缀
        private static final String RETURN_RESULT_SET_PREFIX = "#result-set-";
        //计数前缀
        private static final String RETURN_UPDATE_COUNT_PREFIX = "#update-count-";
        //是否跳过警告
        private boolean ignoreWarnings = true;
        //查询大小
        private int fetchSize = -1;
        //最大行
        private int maxRows = -1;
        //查询超时
        private int queryTimeout = -1;
        //是否跳过结果集处理
        private boolean skipResultsProcessing = false;
        //是否跳过非公共结果集处理
        private boolean skipUndeclaredResults = false;
        //map结果集是否大小写敏感
        private boolean resultsMapCaseInsensitive = false;
    
        public JdbcTemplate() {
        }
        //调用父类方法设置数据源和其他参数
        public JdbcTemplate(DataSource dataSource) {
            this.setDataSource(dataSource);
            this.afterPropertiesSet();
        }
        //调用父类方法设置数据源，懒加载策略和其他参数
        public JdbcTemplate(DataSource dataSource, boolean lazyInit) {
            this.setDataSource(dataSource);
            this.setLazyInit(lazyInit);
            this.afterPropertiesSet();
        }
    }
```

JdbcTemplate 继承了JdbcAccessor实现了JdbcOperations，JdbcAccessor主要封装了数据源的操作，JdbcOperations主要定义了一些操作接口。我们一起看一下JdbcOperations类；

```
public abstract class JdbcAccessor implements InitializingBean {
        protected final Log logger = LogFactory.getLog(this.getClass());
        //数据源
        @Nullable
        private DataSource dataSource;
        //异常翻译
        @Nullable
        private volatile SQLExceptionTranslator exceptionTranslator;
        //懒加载策略
        private boolean lazyInit = true;
        public JdbcAccessor() {
        }
        public void setDataSource(@Nullable DataSource dataSource) {
            this.dataSource = dataSource;
        }
        @Nullable
        public DataSource getDataSource() {
            return this.dataSource;
        }
        protected DataSource obtainDataSource() {
            DataSource dataSource = this.getDataSource();
            Assert.state(dataSource != null, "No DataSource set");
            return dataSource;
        }
        public void setDatabaseProductName(String dbName) {
            this.exceptionTranslator = new SQLErrorCodeSQLExceptionTranslator(dbName);
        }
        public void setExceptionTranslator(SQLExceptionTranslator exceptionTranslator) {
            this.exceptionTranslator = exceptionTranslator;
        }
    }
```

之所以**前面提到spring让我们更方便的处理异常就是这里他包装了一个SQLExceptionTranslator**，其他的代码都是做数据源的检查之类的设置数据源，我们看一下其中getExceptionTranslator()方法

```
public SQLExceptionTranslator getExceptionTranslator() {
        SQLExceptionTranslator exceptionTranslator = this.exceptionTranslator;
        if (exceptionTranslator != null) {
            return exceptionTranslator;
        } else {
            synchronized(this) {
                SQLExceptionTranslator exceptionTranslator = this.exceptionTranslator;
                if (exceptionTranslator == null) {
                    DataSource dataSource = this.getDataSource();
                    if (dataSource != null) {
                        exceptionTranslator = new SQLErrorCodeSQLExceptionTranslator(dataSource);
                    } else {
                        exceptionTranslator = new SQLStateSQLExceptionTranslator();
                    }
                    this.exceptionTranslator = (SQLExceptionTranslator)exceptionTranslator;
                }
                return (SQLExceptionTranslator)exceptionTranslator;
            }
        }
    }
```

这是一个标准的单例模式，我们在学习模板方法模式的路途中有捕获了一个野生的单例；我们继续看JdbcOperations接口我们调其中一个接口进行解析；

```
@Nullable
    <T> T execute(StatementCallback<T> var1) throws DataAccessException;
```

***StatementCallback 接口***

```
@FunctionalInterface
    public interface StatementCallback<T> {
        @Nullable
        T doInStatement(Statement var1) throws SQLException, DataAccessException;
    }
```

***execute实现***

```
@Nullable
    public <T> T execute(StatementCallback<T> action) throws DataAccessException {
        //参数检查
        Assert.notNull(action, "Callback object must not be null");
        //获取连接
        Connection con = DataSourceUtils.getConnection(this.obtainDataSource());
        Statement stmt = null;
        Object var11;
        try {
            //创建一个Statement
            stmt = con.createStatement();
            //设置查询超时时间，最大行等参数（就是一开始那些成员变量）
            this.applyStatementSettings(stmt);
            //执行回调方法获取结果集
            T result = action.doInStatement(stmt);
            //处理警告
            this.handleWarnings(stmt);
            var11 = result;
        } catch (SQLException var9) {
            //出现错误优雅退出
            String sql = getSql(action);
            JdbcUtils.closeStatement(stmt);
            stmt = null;
            DataSourceUtils.releaseConnection(con, this.getDataSource());
            con = null;
            throw this.translateException("StatementCallback", sql, var9);
        } finally {
            JdbcUtils.closeStatement(stmt);
            DataSourceUtils.releaseConnection(con, this.getDataSource());
        }
        return var11;
    }
```

这一个方法可谓是展现的淋漓尽致，这是一个典型的模板方法+回调模式，我们不需要再写过多的重复代码只需要实现自己获取result的方法就好（StatementCallback）事实上我们自己也不需要实现这个方法，继续向上看，我们是如何调用execute方法的，以查询为例,我们看他是如何一步步调用的：

***查询方法***
```
public List<Users> findAll() {
        JdbcTemplate jdbcTemplate = DataSourceConfig.getTemplate();
        String sql = "select nickname,comment,age from users";
        return jdbcTemplate.query(sql, new BeanPropertyRowMapper<Users>(Users.class));
    }
```

***query实现***

```
public <T> List<T> query(String sql, RowMapper<T> rowMapper) throws DataAccessException {
        return (List)result(this.query((String)sql, (ResultSetExtractor)(new RowMapperResultSetExtractor(rowMapper))));
    }
```

这里的RowMapper是负责将结果集中一行的数据映射成实体返回，用到了反射技术，这里就不展开了，有兴趣的同学可以自己打开源码阅读，继续向下：

***query实现***
```
@Nullable
    public <T> T query(final String sql, final ResultSetExtractor<T> rse) throws DataAccessException {
        Assert.notNull(sql, "SQL must not be null");
        Assert.notNull(rse, "ResultSetExtractor must not be null");
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Executing SQL query [" + sql + "]");
        }
        //实现回调接口
        class QueryStatementCallback implements StatementCallback<T>, SqlProvider {
            QueryStatementCallback() {
            }
            @Nullable
            public T doInStatement(Statement stmt) throws SQLException {
                ResultSet rs = null;
                Object var3;
                try {
                    //这里真正的执行我们的sql语句
                    rs = stmt.executeQuery(sql);
                    //处理对象映射
                    var3 = rse.extractData(rs);
                } finally {
                    JdbcUtils.closeResultSet(rs);
                }
                return var3;
            }
            public String getSql() {
                return sql;
            }
        }
        //调用execute接口
        return this.execute((StatementCallback)(new QueryStatementCallback()));
    }
```

看到这里相信你也不得拍手称奇，Spring处理的非常巧妙，请继续向下看：

***update详解***
```
protected int update(PreparedStatementCreator psc, @Nullable PreparedStatementSetter pss) throws DataAccessException {
        this.logger.debug("Executing prepared SQL update");
        return updateCount((Integer)this.execute(psc, (ps) -> {
            Integer var4;
            try {
                if (pss != null) {
                    pss.setValues(ps);
                }
                int rows = ps.executeUpdate();
                if (this.logger.isTraceEnabled()) {
                    this.logger.trace("SQL update affected " + rows + " rows");
                }
                var4 = rows;
            } finally {
                if (pss instanceof ParameterDisposer) {
                    ((ParameterDisposer)pss).cleanupParameters();
                }
            }
            return var4;
        }));
    }
```

为什么我要把update函数拎出来讲了，因为update这里使用了lambda函数,回想我们StatementCallback定义只有一个方法的接口，他就是一个函数是接口，所以他是一个函数式接口，所以这里直接使用lambda语法，**lambda函数允许你直接内连，为函数接口的抽象方法提供实现，并且整个表达式作为函数接口的一个实例**。我们在平时学习中可能知道了lambda语法但是可能使用的较少，或者不知道如何用于实战，那么多阅读源码一定可以提升你的实战能力。 我们可以看到JDBCTemplate使用了很多回调。为什么要用回调（Callback)?**如果父类有多个抽象方法，子类需要全部实现这样特别麻烦，而有时候某个子类只需要定制父类中的某一个方法该怎么办呢？这个时候就要用到Callback回调了就可以完美解决这个问题**，可以发现JDBCTemplate并没有完全拘泥于模板方法，非常灵活。我们在实际开发中也可以借鉴这种方法。

### 模板方法模式的更多应用

事实上很多有关生命周期的类都用到了模板方法模式，最典型的也是可能我们最熟悉的莫过于Servlet了，废话不多说上源码
```
public abstract class HttpServlet extends GenericServlet
    {
    }
```

![](https://user-gold-cdn.xitu.io/2019/5/21/16ad9448cb69de5e?imageView2/0/w/1280/h/960/ignore-error/1) HttpServlet的所有方法，我们看到HttpServlet继承了GenericServlet，我们继续看：

```
public abstract class GenericServlet 
    implements Servlet, ServletConfig, java.io.Serializable
{
    private static final String LSTRING_FILE = "javax.servlet.LocalStrings";
    private static ResourceBundle lStrings =
        ResourceBundle.getBundle(LSTRING_FILE);

    private transient ServletConfig config;
    
    public GenericServlet() { }
    
    //没有实现钩子
    public void destroy() {
    }
    
    public String getInitParameter(String name) {
        ServletConfig sc = getServletConfig();
        if (sc == null) {
            throw new IllegalStateException(
                lStrings.getString("err.servlet_config_not_initialized"));
        }

        return sc.getInitParameter(name);
    }
    
    public Enumeration<String> getInitParameterNames() {
        ServletConfig sc = getServletConfig();
        if (sc == null) {
            throw new IllegalStateException(
                lStrings.getString("err.servlet_config_not_initialized"));
        }

        return sc.getInitParameterNames();
    }   
     
    public ServletConfig getServletConfig() {
	return config;
    }
    
    public ServletContext getServletContext() {
        ServletConfig sc = getServletConfig();
        if (sc == null) {
            throw new IllegalStateException(
                lStrings.getString("err.servlet_config_not_initialized"));
        }

        return sc.getServletContext();
    }

    public String getServletInfo() {
	return "";
    }

    public void init(ServletConfig config) throws ServletException {
	this.config = config;
	this.init();
    }
    
    public void init() throws ServletException {

    }
    
    public void log(String msg) {
	getServletContext().log(getServletName() + ": "+ msg);
    }
  
    public void log(String message, Throwable t) {
	getServletContext().log(getServletName() + ": " + message, t);
    }
    
    public abstract void service(ServletRequest req, ServletResponse res)
	throws ServletException, IOException;
    
    public String getServletName() {
        ServletConfig sc = getServletConfig();
        if (sc == null) {
            throw new IllegalStateException(
                lStrings.getString("err.servlet_config_not_initialized"));
        }

        return sc.getServletName();
    }
}
```

可以看到这就是个典型的模板方法类蛮，而且钩子函数也在这里展现的淋漓尽致，如init、destroy方法等，JDK中很多类都是用了模板方法等着你发现哦。

### 模板方法模式在Vue.js中的应用

模板方法模式在其他语言中也有实现比如Vue.js、React中；比如Vue生命周期肯定使用了模板方法，我就不对源码展开分析了。
![](https://user-gold-cdn.xitu.io/2019/5/21/16ad94b7aa8bb518?imageView2/0/w/1280/h/960/ignore-error/1)

### 总结

设计模式在Spring中得到了大量的应用，感兴趣的同学可以看看Spring源码加以学习，如果你觉得我写的还不错的话点个赞吧，如果你发现了错误，或者不好的地方也可以及时告诉我加以改正，谢谢！您的赞赏和批评是进步路上的好伙伴。
