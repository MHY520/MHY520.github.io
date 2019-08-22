原文链接：https://www.cnblogs.com/xslzwm/p/10050575.html

**MyBatis框架中Mapper映射配置的使用及原理**  
（Mapper用于映射SQL语句，可以说是MyBatis操作数据库的核心特性之一，这里我们讨论java的MyBatis框架中Mapper映射配置的使用及原理解析，包括对mapper.xml配置文件的读取流程解读）  

*1、Mapper的内置方法*   
  model层就是实体类，对应数据库的表。controller层是Servlet，主要是负责业务模块流程的控制，调用service接口的方法，在struts2就是Action。Service层主要做逻辑判断，Dao层是数据访问层（与数据库进行对接）。至于Mapper是mybatis框架映射用到的，而mapper映射文件在dao层使用。  

*2、解析mapper的xml配置文件*  
  我们来看看mybatis是怎么读取mapper的xml配置文件并解析其中的sql语句。我们还记得是这样配置sqlSessionFactory的：  
```
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">   
  <property name="dataSource" ref="dataSource" />  
  <property name="configLocation" value="classpath:configuration.xml"></property>   
  <property name="mapperLocations" value="classpath:com/xxx/mybatis/mapper/*.xml"/>   
  <property name="typeAliasesPackage" value="com.mybatis.model" />   
</bean>  
```
  这里配置了一个mapperLocation属性，它是一个表达式，sqlSessionFactory会根据这个表达式读取包com.xxx.mybatis.mapper下面的所有xml格式文件，那么具体是怎么根据这个属性来读取配置文件的呢？
  答案就在SqlSessionFactoryBean类中buildSqlSessionFactory方法中：
```
if(!isEmpty(this.mapperLocations)){
   for(Resource mapperLocation:this.mapperLocations){
      continue;
  }
   try{
      XMLMapperBuilder xmlMapperBuilder=new XMLMapperBuilder(mapperLocation.getInputStream(),configuration,mapperLocation.toString(),configuration.getSqlFragments());
      xmlMapperBuilder.parse();
   }catch(Exception e){
      throw new NestedIOException("Failed to parse mapping resource:'"+mapperLocation+"'",e);
   }finally{
      ErrorContext.instance().reset();
   }
   if(logger.isDebugEnablec()){
      logger.debug("Parsed mapper file:'"+mapperLocation+"'");
   }
}
```
  mybatis使用XMLMapperBuilder类的实例来解析mapper配置文件。
```
public XMLMapperBuilder(Reader reader,configuration configuration,String resource,Map<String,XNode> sqlFragments){
    this(new XPathParser(reader,true,configuration.getVariables(),new XMLMapperEntityResolver()),configuration,resource,sqlFragments);
}
private XMLMapperBuilder(XPathParser parser, Configuration configuration, String resource, Map<String, XNode> sqlFragments) { 
  super(configuration); 
  this.builderAssistant = new MapperBuilderAssistant(configuration, resource); 
  this.parser = parser; 
  this.sqlFragments = sqlFragments; 
  this.resource = resource; 
}
```
  接着系统调用xmlMapperBuilder的parse方法解析mapper
```
public void parse(){
   //如果configuration对象还没加载xml配置文件（避免重复加载，实际上是确认是否解析了mapper节点的属性以及内容，
   //为解析它的子节点如cache、sql、select、resultMap、parameterMap等做准备），
   //则从输入流中解析mapper节点，然后再将resource的状态置为已加载
   if(!configuration.isResourceloaded(resource)){
      configurationElement(parser.evalNode("/mapper"));
      configuration.addCloadedResource(respurce);
      bindMapperForNamespace();
   }
   //解析在configurationElement函数中处理resultMap时其extends属性指向的父对象还没被处理的<resultMap>节点 
  parsePendingResultMaps(); 
  //解析在configurationElement函数中处理cache-ref时其指向的对象不存在的<cache>节点(如果cache-ref先于其指向的cache节点加载就会出现这种情况) 
  parsePendingChacheRefs(); 
  //同上，如果cache没加载的话处理statement时也会抛出异常 
  parsePendingStatements(); 
}
```
  mybatis解析mapper的xml文件的过程已经很明显了，接下来我们看看它是怎么解析mapper的：
```
private void configurationElement(XNode context) { 
  try{
    //获取mapper节点的namespace属性
    String namespace=context.getStringAttribute("namespace");
     if(namespace.equals("")){
         throw new BuilderException("Mapper's namespace cannot be empty");
     //设置当前的namespace
     builderAssistant.setCuttentNamespace(namespace);
     //解析mapper的<cache-ref>节点
     cacheRefElement(context.evalNode("cache-ref"));
      //解析mapper的<cache>节点 
     cacheElement(context.evalNode("cache")); 
     //解析mapper的<parameterMap>节点 
     parameterMapElement(context.evalNodes("/mapper/parameterMap")); 
     //解析mapper的<resultMap>节点 
     resultMapElements(context.evalNodes("/mapper/resultMap")); 
     //解析mapper的<sql>节点 
     sqlElement(context.evalNodes("/mapper/sql")); 
     //使用XMLStatementBuilder的对象解析mapper的<select>、<insert>、<update>、<delete>节点, 
     //mybaits会使用MappedStatement.Builder类build一个MappedStatement对象, 
     //所以mybaits中一个sql对应一个MappedStatement 
     buildStatementFromContext(context.evalNodes("select|insert|update|delete")); 
     }
  }catch(Exception e){
     throw new BuilderException("Error parsing Mapper XML. Cause: " + e, e); 
  }
}
```
  configurationElement函数几乎解析了mapper节点下所有子节点，至此mybatis解析了mapper中的所有节点，并将其加入到了Configuration对象中提供sqlSessionFactory对象随时使用。这里我们需要补充讲一下mybatis是怎么使用XMLStatementBuilder类的对象的parseStatementNode函数借用MapperBuilderAssistant类对象builderAssistant的addMappedStatement解析MappedStatement并将其关联到Configuration类对象的
```
public void parseStatementNode() { 
  //ID属性 
  String id = context.getStringAttribute("id"); 
  //databaseId属性 
  String databaseId = context.getStringAttribute("databaseId"); 
  
  if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) { 
   return; 
  } 
  //fetchSize属性 
  Integer fetchSize = context.getIntAttribute("fetchSize"); 
  //timeout属性 
  Integer timeout = context.getIntAttribute("timeout"); 
  //parameterMap属性 
  String parameterMap = context.getStringAttribute("parameterMap"); 
  //parameterType属性 
  String parameterType = context.getStringAttribute("parameterType"); 
  Class<?> parameterTypeClass = resolveClass(parameterType); 
  //resultMap属性 
  String resultMap = context.getStringAttribute("resultMap"); 
  //resultType属性 
  String resultType = context.getStringAttribute("resultType"); 
  //lang属性 
  String lang = context.getStringAttribute("lang"); 
  LanguageDriver langDriver = getLanguageDriver(lang); 
  
  Class<?> resultTypeClass = resolveClass(resultType); 
  //resultSetType属性 
  String resultSetType = context.getStringAttribute("resultSetType"); 
  StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString())); 
  ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType); 
  
  String nodeName = context.getNode().getNodeName(); 
  SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH)); 
  //是否是<select>节点 
  boolean isSelect = sqlCommandType == SqlCommandType.SELECT; 
  //flushCache属性 
  boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect); 
  //useCache属性 
  boolean useCache = context.getBooleanAttribute("useCache", isSelect); 
  //resultOrdered属性 
  boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false); 
  
  // Include Fragments before parsing 
  XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant); 
  includeParser.applyIncludes(context.getNode()); 
  
  // Parse selectKey after includes and remove them. 
  processSelectKeyNodes(id, parameterTypeClass, langDriver); 
    
  // Parse the SQL (pre: <selectKey> and <include> were parsed and removed) 
  SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass); 
  //resultSets属性 
  String resultSets = context.getStringAttribute("resultSets"); 
  //keyProperty属性 
  String keyProperty = context.getStringAttribute("keyProperty"); 
  //keyColumn属性 
  String keyColumn = context.getStringAttribute("keyColumn"); 
  KeyGenerator keyGenerator; 
  String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX; 
  keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true); 
  if (configuration.hasKeyGenerator(keyStatementId)) { 
   keyGenerator = configuration.getKeyGenerator(keyStatementId); 
  } else { 
   //useGeneratedKeys属性 
   keyGenerator = context.getBooleanAttribute("useGeneratedKeys", 
     configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType)) 
     ? new Jdbc3KeyGenerator() : new NoKeyGenerator(); 
  } 
  
  builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType, 
    fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass, 
    resultSetTypeEnum, flushCache, useCache, resultOrdered,  
    keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets); 
 }
```
  由以上代码可以看出mybatis使用XPath解析mapper的配置文件后将其中的resultMap、parameterMap、cache、statement等节点使用关联的builder创建并将得到的对象关联到configuration对象中，而这个configuration对象可以从sqlSession中获取的，这就解释了我们在使用sqlSession对数据库进行操作时mybatis怎么获取到mapper并执行其中的sql语句的问题。
