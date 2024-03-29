---
author: ixiaohei
pubDatetime: 2023-09-05T21:13:52.737Z
title: spring mybatis starter源码分析
postSlug: mybatis
featured: true
ogImage: https://user-images.githubusercontent.com/53733092/215771435-25408246-2309-4f8b-a781-1f3d93bdf0ec.png
tags:
  - mybatis
  - java
description: mybatis和spring boot集成源码分析
---

## spring boot集成
由于mybatis官方已经提供了对spring boot集成，引入spring boot推荐的starter风格包即可。

## MybatisAutoConfiguration
mybatis自动化配置全部在此类`org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration`中实现

## @Mapper注解扫描实现
在MybatisAutoConfiguration引入了AutoConfiguredMapperScannerRegistrar；其会往spring容器中注册Bean定义。核心逻辑在`registerBeanDefinitions`方法中
```java
@Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

      logger.debug("Searching for mappers annotated with @Mapper");

      ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);

      try {
        if (this.resourceLoader != null) {
          scanner.setResourceLoader(this.resourceLoader);
        }

        List<String> packages = AutoConfigurationPackages.get(this.beanFactory);
        if (logger.isDebugEnabled()) {
          for (String pkg : packages) {
            logger.debug("Using auto-configuration base package '{}'", pkg);
          }
        }

        scanner.setAnnotationClass(Mapper.class);
        scanner.registerFilters();
        scanner.doScan(StringUtils.toStringArray(packages));
      } catch (IllegalStateException ex) {
        logger.debug("Could not determine auto-configuration package, automatic mapper scanning disabled.", ex);
      }
    }
```
### ClassPathMapperScanner
ClassPathMapperScanner是一个类路径Mapper扫描器，对指定类路径下的类扫描，会将有`@Mapper`注解的接口注册成BeanDefinition；注册逻辑在方法`processBeanDefinitions`中。
```java
private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    for (BeanDefinitionHolder holder : beanDefinitions) {
      definition = (GenericBeanDefinition) holder.getBeanDefinition();

      if (logger.isDebugEnabled()) {
        logger.debug("Creating MapperFactoryBean with name '" + holder.getBeanName() 
          + "' and '" + definition.getBeanClassName() + "' mapperInterface");
      }

      // the mapper interface is the original class of the bean
      // but, the actual class of the bean is MapperFactoryBean
      definition.getConstructorArgumentValues().addGenericArgumentValue(definition.getBeanClassName()); // issue #59
      definition.setBeanClass(this.mapperFactoryBean.getClass());

      definition.getPropertyValues().add("addToConfig", this.addToConfig);

      boolean explicitFactoryUsed = false;
      if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
        definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionFactory != null) {
        definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
        explicitFactoryUsed = true;
      }

      if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
        if (explicitFactoryUsed) {
          logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionTemplate != null) {
        if (explicitFactoryUsed) {
          logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
        explicitFactoryUsed = true;
      }

      if (!explicitFactoryUsed) {
        if (logger.isDebugEnabled()) {
          logger.debug("Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
        }
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
      }
    }
  }
```
从以上代码可以看出`Mapper`都是由MapperFactoryBean工厂Bean产生。它使用`Mapper`的接口class作为构造器参数。另外由代码上下文可知道最终`Mapper`的`BeanDefinition`被设置了`AbstractBeanDefinition.AUTOWIRE_BY_TYPE`的autowrieMode；此autowrieMode是按Bean属性的Type注入依赖Bean

### MapperFactoryBean
在初始化这个Bean`org.mybatis.spring.mapper.MapperFactoryBean`时候会添加`Mapper`,核心逻辑在方法`checkDaoConfig`中
```java
@Override
  protected void checkDaoConfig() {
    super.checkDaoConfig();

    notNull(this.mapperInterface, "Property 'mapperInterface' is required");

    Configuration configuration = getSqlSession().getConfiguration();
    if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
      try {
        configuration.addMapper(this.mapperInterface);
      } catch (Exception e) {
        logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", e);
        throw new IllegalArgumentException(e);
      } finally {
        ErrorContext.instance().reset();
      }
    }
  }
```
可见其`configuration.addMapper(this.mapperInterface);`代码，对其深入理解，发现它是生成`Mapper`逻辑

## 生成Mapper
### ## org.apache.ibatis.session.Configuration
此类代表整个mybatis的配置信息。上文中的`addMapper`是它的方法
```java
public <T> void addMapper(Class<T> type) {
    mapperRegistry.addMapper(type);
}
```
可见其中委托给MapperRegistry处理

### org.apache.ibatis.binding.MapperRegistry
Mapper的仓库。负责管理Mapper和生产Mapper代理。Mapper生产核心逻辑如下
```java
  public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
        knownMappers.put(type, new MapperProxyFactory<T>(type));
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }
```
由以上代码可知，MapperRegistry里面都是存放的Mapper对应的MapperProxyFactory；而由`getMapper`方法可以看出，Mapper实例是由MapperProxyFactory新建出来的
```java
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```

### org.apache.ibatis.binding.MapperProxyFactory
Mapper代理工厂类，负责生产Mapper的代理实例；其`newInstance`方法如下
```java
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
```
通过上述代码可知，Mapper都是jdk动态代理出来的实例，而处理的InvocationHandler为`MapperProxy`;至此Mapper的生成逻辑已经完成。


## Mapper调用
### org.apache.ibatis.binding.MapperProxy
mapper的代理类

### 调用关系
mapper.xxx()
|
MapperProxy.invoke()
|
MapperMethod.execute()
|
SqlSessionTemplate.curd()
|
SqlSessionInterceptor.invoke()
|
DefaultSqlSession.curd()

