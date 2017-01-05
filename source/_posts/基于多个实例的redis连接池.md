title: 基于多个实例的redis连接池
date: 2015-11-03 23:38:33
tags: [redis连接池,Jedis]
---
redis截至目前为止最新版本为3.0.5稳定版，对于集群的支持进一步稳定，对于低版本的redis，可以通过代理形式和一致性hash算法实现分布式部署。针对各个语言，redis均有很多优秀的客户端程序，在java中有Jedis这个jar，以下实现均以此jar文件进行封装开发。现在下面的项目并不是针对集群而言，而是针对多个redis实例的分库而言，第一次[github上代码～](https://github.com/truechuan/ccredis)  
基于2.7.3版本jedis.jar,写起来还是发现了与之前2.4版本的一些变化。其中发现两点：  
1. 底层连接池版本升级，导致一些参数配置变化。jedis2.1版本连接池配置通过继承org.apache.commons.pool.impl.GenericObjectPool.Config
该类实现：  
``` java
public class JedisPoolConfig extends Config
```  
在2.7.3版本中这样实现 
``` java
public class JedisPoolConfig extends GenericObjectPoolConfig  
public class GenericObjectPoolConfig extends BaseObjectPoolConfig  
public abstract class BaseObjectPoolConfig implements Cloneable
```  
有此可见，JedisPoolConfig是实现了一个接口而非继承，使得更加抽象，现在也仅理解至此。 
2. 对于Redis连接关闭更加方便，在老版本中关闭连接需要使用returnResource(jedis)该方法，在获取连接异常是使用returnBrokenResource(jedis)来关闭连接。  
``` java
	void closeRedis(){
	    JedisPool pool = new JedisPool("host");
        Jedis jedis = null;
        try {
            jedis = pool.getResource();
            pool.returnResource(jedis);
        } catch (Exception e) {
            if (jedis != null) {
                pool.returnBrokenResource(jedis);
            }
        }
	}
``` 
在新版源码中，returnResource(jedis)，returnBrokenResource(jedis)均被废弃，  
``` java
	    /**
	     * @deprecated starting from Jedis 3.0 this method won't exist. 
	     * Resouce cleanup should be done
	     *             using @see {@link redis.clients.jedis.Jedis#close()}
	     */
        @Deprecated
	    public void returnResource(final Jedis resource) {
	    if (resource != null) {
	      try {
	        resource.resetState();
	        returnResourceObject(resource);
	      } catch (Exception e) {
	        returnBrokenResource(resource);
	        throw new JedisException("Could not return the resource to the pool", e);
	      }
	    }
       }  

	  /**
	   * @deprecated starting from Jedis 3.0 this method won't exist.
	   *  Resouce cleanup should be done
	   *             using @see {@link redis.clients.jedis.Jedis#close()}
	   */
	  @Deprecated
	  public void returnBrokenResource(final Jedis resource) {
	    if (resource != null) {
	      returnBrokenResourceObject(resource);
	    }
	  }  
``` 
在新版本的jedis中，对连接关闭更加方便，直接调用close(),  

``` java
	  @Override
	  public void close() {
	    if (dataSource != null) {
	      if (client.isBroken()) {
	        this.dataSource.returnBrokenResource(this);
	      } else {
	        this.dataSource.returnResource(this);
	      }
	    } else {
	      client.close();
	    }
	  }
``` 