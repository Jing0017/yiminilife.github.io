---
title: zuul 根据标签进行路由
categories: 技术
tags:
  - zuul
  - 网关
  - Netflix
abbrlink: 6d250c8e
date: 2019-10-23 10:18:54
---

## 背景

笔者所在部门为了响应公司的号召，需要建立灰度环境。那么既然有了灰度环境的概念，必然需要实现灰度发布。

而目前的网关zuul并不能实现我们的需求，需要对路由策略进行扩展。

## 目标

网关根据指定的标签，转发请求到包含指定标签的服务节点上。

## 方案

方案设计图如下：

![1571798323362](1571798323362.png)

## 实现

### 实现步骤

#### 给服务打标签

修改项目的配置文件，每台服务器都要配置。笔者所在公司使用consul做注册中心，故配置如下：

```yaml
spring:
 cloud:
  consul:
   port: 8500
    host: xx.xx.xx.xx
     discovery:
      tags:
      - pro
```

启动成功后观察consul，出现相应标签，表示服务注册成功。

![1571798927307](1571798927307.png)

#### 扩展路由策略

##### 负载均衡原理简析

首先，我们查看zuul源码，发现其使用的是netfix提供的ribbon-loadbalancer来实现负载均衡。程序中使用的版本为2.2.5。阅读源码发现ribbon主要通过ILoadBalancer接口定义了一系列行为：

![](image2019-10-10_14-30-33.png)

观察ILoadBalancer最基础的实现类com.netflix.loadbalancer.BaseLoadBalancer可以了解主要行为。LB的主要实现都在里面，DynamicServerListLoadBalancer和ZoneAwareLoadBalancer不过是扩展了他的功能而已。

可以看到一个LB中包含的几个重要的属性（工具），正是这几个工具为LB的功能提供支持。 他们是：

| 属性              | 描述                                                         |
| ----------------- | ------------------------------------------------------------ |
| Iping             | 判断目标服务是否存活 对应不同的协议不同的方式去探测，得到后端服务是否存活。如有http的，还有对于微服务框架内的服务存活的NIWSDiscoveryPing是通过eureka client来获取的instanceinfo中的信息来获取。 |
| IRule             | 负载均衡策略,可以插件化的为LB提供各种适用的负载均衡算法      |
| LoadBalancerStats | LB运行信息 记录LB的实时运行信息，这些运行信息可以被用来作为LB策略的输入。 |

观察下几个主要方法。首先是最重要的的服务选择方法：

![](image2019-10-10_14-38-4.png)

可以看到将该功能委托给包含的负载均衡策略rule来实现。ok，下面我们来看一下Ribbon负载均衡策略定义：

IRule其实就只做了一件事情Server choose(Object key)，可以看到这个功能是在LB中定义（要求）的，LB把这个功能委托给IRule来实现。不同的IRule可以向LB提供不同的负载均衡算法。

![](image2019-10-10_14-39-57.png)

com.netflix.loadbalancer包下面的提供了常用的几种策略。有RoundRobinRule、RandomRule这样的不依赖于Server运行状况的策略，也有AvailabilityFilteringRule、WeightedResponseTimeRule等多种基于收集到的Server运行状况决策的策略。判断运行状况时有，判断单个server的，也有判断整个zone的，适用于各种不同场景需求。

实现上有些策略可以继承一个既存的简单策略用于某些启动时候，也可以包含一个简单策略。甚至有ZoneAvoidanceRule这样的可以包含复合谓词的条件判断。

详见ZoneAvoidanceRule源码：

![](image2019-10-10_14-41-59.png)

其中组合了ZoneAvoidancePredicate和AvailabilityPredicate俩个谓词。前一个，以一个区域为单位考察可用性，对于不可用的区域整个丢弃，从剩下区域中选可用的server。判断出最差的区域，排除掉最差区域。在剩下的区域中，将按照服务器实例数的概率抽样法选择，从而判断判定一个zone的运行性能是否可用，剔除不可用的zone（的所有server），AvailabilityPredicate用于过滤掉连接数过多的Server。

我们看到ZoneAvoidanceRule中并没有重写choose方法，查看一下继承图：

![](image2019-10-10_14-57-34.png)

在PredicateBasedRule中我们发现：

![](image2019-10-10_14-58-19.png)

此处choose方法从lb中获取所有服务节点，然后针对具体实现类中的谓词（XXXPredicate）对服务节点进行过滤。继续跟踪最后发现执行至AbstractServerPredicate类中的代码：

![](image2019-10-10_15-13-44.png)

此处对server列表进行了过滤，过滤谓词是后面的this.getServerOnlyPredicate该方法只是简单返回了serverOnlyPredicate：

![](image2019-10-10_15-15-40.png)

此处可以发现，最终执行的谓词是具体子类定义的谓词，也就是ZoneAvoidanceRule的compositePredicate。

由此可见，我们只要按照ribbon的框架结构，提供自定义的负载均衡策略，实现具体的组合谓词，就可以实现根据标签来过滤服务节点。

##### 实现自定义的负载均衡策略（ServiceTagsAwareRule）

要实现自定义的负载均衡策略ServiceTagsAwareRule，我们先要实现它的谓词，首先建立一个抽象谓词类（BaseDiscoveryEnabledPredicate）：

```java
package com.shein.supply.wms.gateway.predicate;
 
import com.netflix.loadbalancer.AbstractServerPredicate;
import com.netflix.loadbalancer.PredicateKey;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cloud.consul.discovery.ConsulServer;
 
import java.util.Objects;
 
/**
 * A template method predicate to be applied to service discovered server instances. The concrete implementation of
 * this class need to implement the {@link #apply(ConsulServer)} method.
 *
 * @author yanjing
 * date: 2019/9/30
 * description:
 */
public abstract class BaseDiscoveryEnabledPredicate extends AbstractServerPredicate {
 
    protected Logger logger = LoggerFactory.getLogger(this.getClass());
 
//此处做一下前置判断，不为空且是ConsulServer的实例，spring-cloud框架会自动将候选的服务节点从Server向下转型为ConsulServer
    @Override
    public boolean apply(PredicateKey input) {
        return Objects.nonNull(input)
                && input.getServer() instanceof ConsulServer
                && apply((ConsulServer) input.getServer());
    }
 
 
    /**
     * Returns whether the specific {@link ConsulServer} matches this predicate.
     *
     * @param server the discovered server
     * @return whether the server matches the predicate
     */
    protected abstract boolean apply(ConsulServer server);
}
```

再实现具体的谓词逻辑（ServiceTagsAwarePredicate）：

```java
package com.shein.supply.wms.gateway.predicate;
 
import cn.dotfashion.soa.framework.util.JsonTools;
import com.shein.supply.wms.gateway.api.RibbonFilterContext;
import com.shein.supply.wms.gateway.support.RibbonFilterContextHolder;
import org.apache.commons.collections4.CollectionUtils;
import org.springframework.cloud.consul.discovery.ConsulServer;
 
import java.util.List;
 
/**
 * A default implementation of {@link ConsulServer} that matches the instance against the attributes
 * registered through
 *
 * @author yanjing
 * date: 2019/9/30
 * description:
 */
public class ServiceTagsAwarePredicate extends BaseDiscoveryEnabledPredicate {
 
    @Override
    protected boolean apply(ConsulServer server) {
        final RibbonFilterContext context = RibbonFilterContextHolder.getCurrentContext();
        final List<String> contextTags = context.getServiceTags(); //自定义的prefilter中指定的生产环境或灰度环境标签
        final List<String> serviceTags = server.getHealthService().getService().getTags(); //候选服务节点的标签
        boolean hit = CollectionUtils.isSubCollection(contextTags, serviceTags); //候选服务节点的标签列表中是否包含指定的环境标签
        logger.info("ServiceTagsAwarePredicate contextTags:{}, serviceTags:{},result: {},server:{}",
                contextTags, serviceTags, hit, JsonTools.defaultMapper().toJson(server));
        return hit;
    }
}
```

接下来我们来实现负载均衡策略（ServiceTagsAwareRule）：

```java
package com.shein.supply.wms.gateway.rule;
 
import com.netflix.loadbalancer.AbstractServerPredicate;
import com.netflix.loadbalancer.AvailabilityPredicate;
import com.netflix.loadbalancer.CompositePredicate;
import com.netflix.loadbalancer.PredicateBasedRule;
import com.shein.supply.wms.gateway.predicate.BaseDiscoveryEnabledPredicate;
import com.shein.supply.wms.gateway.predicate.ServiceTagsAwarePredicate;
 
/**
 * @author yanjing
 * date: 2019/9/30
 * description:
 */
public class ServiceTagsAwareRule extends PredicateBasedRule {
 
    private CompositePredicate predicate;
 
    /**
     * Creates new instance of {@link ServiceTagsAwareRule}.
     */
    public ServiceTagsAwareRule() {
        super();
        predicate = createCompositePredicate(new ServiceTagsAwarePredicate(), new AvailabilityPredicate(this, null));
    }
 
    @Override
    public AbstractServerPredicate getPredicate() {
        return predicate;
    }
 
    /**
     * Creates the composite predicate with fallback strategies.
     *
     * @param discoveryEnabledPredicate the discovery service predicate
     * @param availabilityPredicate     the availability predicate
     * @return the composite predicate
     */
    private CompositePredicate createCompositePredicate(BaseDiscoveryEnabledPredicate discoveryEnabledPredicate, AvailabilityPredicate availabilityPredicate) {
        return CompositePredicate
                .withPredicates(discoveryEnabledPredicate, availabilityPredicate)
                .addFallbackPredicate(availabilityPredicate)
                .build();
    }
}
```

将自定义的策略配置交给spring管理：

```java
package com.shein.supply.wms.gateway.support;
 
import com.shein.supply.wms.gateway.rule.ServiceTagsAwareRule;
import org.springframework.beans.factory.config.ConfigurableBeanFactory;
import org.springframework.boot.autoconfigure.AutoConfigureBefore;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.cloud.netflix.ribbon.RibbonClientConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Scope;
 
/**
 * The Ribbon discovery filter auto configuration.
 *
 * @author yanjing
 * date: 2019/9/30
 * description:
 */
@Configuration
@AutoConfigureBefore(RibbonClientConfiguration.class)
@ConditionalOnProperty(value = "ribbon.filter.tags.enabled", matchIfMissing = true)
public class RibbonDiscoveryRuleAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
    public ServiceTagsAwareRule serviceTagsAwareRule() {
        return new ServiceTagsAwareRule();
    }
}
```

自定义RibbonFilterContext:

```java
package com.shein.supply.wms.gateway.api;
 
import java.util.List;
 
/**
 * Ribbon discovery filter context,stores the tags based on which the server matching will be performed.
 *
 * @author yanjing
 * date: 2019/9/30
 * description:
 */
public interface RibbonFilterContext {
 
    /**
     * 给上下文增加标签
     *
     * @param tag 标签
     * @return 上下文实例
     */
    RibbonFilterContext add(String tag);
 
    /**
     * 删除标签
     *
     * @param tag 标签
     * @return 上下文实例
     */
    RibbonFilterContext remove(String tag);
 
 
    /**
     * 获取所有服务标签
     *
     * @return 服务标签list
     */
    List<String> getServiceTags();
}
```

RibbonFilterContextHolder:

```java
package com.shein.supply.wms.gateway.support;
 
import com.shein.supply.wms.gateway.api.RibbonFilterContext;
 
/**
 * The Ribbon filter context holder.
 * @author yanjing
 * date: 2019/9/30
 * description:
 */
public class RibbonFilterContextHolder {
 
    /**
     * Stores the {@link RibbonFilterContext} for current thread.
     */
    private static final ThreadLocal<RibbonFilterContext> CONTEXT_HOLDER = new InheritableThreadLocal<RibbonFilterContext>() {
        @Override
        protected RibbonFilterContext initialValue() {
            return new DefaultRibbonFilterContext();
        }
    };
 
    /**
     * Retrieves the current thread bound instance of {@link RibbonFilterContext}.
     *
     * @return the current context
     */
    public static RibbonFilterContext getCurrentContext() {
        return CONTEXT_HOLDER.get();
    }
 
    /**
     * Clears the current context.
     */
    public static void clearCurrentContext() {
        CONTEXT_HOLDER.remove();
    }
}
```

DefaultRibbonFilterContext:

```java
package com.shein.supply.wms.gateway.support;
 
import com.google.common.collect.Lists;
import com.shein.supply.wms.gateway.api.RibbonFilterContext;
 
import java.util.List;
 
/**
 * Ribbon discovery filter context, stores the attributes based on which the server matching will be performed.
 *
 * @author yanjing
 * date: 2019/9/30
 * description:
 */
public class DefaultRibbonFilterContext implements RibbonFilterContext {
    /**
     * Filter attributes.
     */
    private final List<String> serviceTags = Lists.newArrayList();
 
 
    @Override
    public RibbonFilterContext add(String tag) {
        serviceTags.add(tag);
        return this;
    }
 
    @Override
    public RibbonFilterContext remove(String tag) {
        serviceTags.remove(tag);
        return this;
    }
 
    @Override
    public List<String> getServiceTags() {
        return serviceTags;
    }
}
```

最后在配置文件中加入以下配置：

![](image2019-10-11_10-30-31.png)

指定ribbon加载我们自定义的策略，此处参考了[spring-cloud官方文档](https://cloud.spring.io/spring-cloud-netflix/multi/multi_spring-cloud-ribbon.html)：

#### gateway根据标签路由

在wms-gateway的AuthUrlRsp类中新增access2Grey字段

在wms-gateway的RoleFilter中做如下修改：

![](image2019-10-10_16-19-49.png)

其中RibbonFilterConstants：

![](image2019-10-10_16-20-20.png)

## 遇到的问题以及解决方案

### 请求没有路由到指定标签的服务

开发环境自测的时候发现一个问题，请求没有按照指定的标签路由，排查发现自定义的RibbonFilterContext中存储的标签列表中存在重复的标签，日志如下：

![](image2019-10-10_16-35-44.png)

由此可见，contextTags中出现了俩个grey标签，导致谓词判断执行结果为false。而理论上contextTags中应该不会出现重复的标签。最终排查发现，虽然我们自定义了一套threadlocal变量，但是框架底层使用的线程池会对线程进行复用：

![](image2019-10-10_16-53-55.png)

解决方案：

1. 复用zuul维护的RequestContext

在每个请求执行完成时候，将相应的threadlocal变量remove。但是应该在什么时间点remove呢？没有思路，直接查询下zuul的生命周期，ZuulServlet:

![](image2019-10-10_16-56-31.png)
  我们发现，zuul自身也维护了一套上下文环境变量，并且在请求结束时清空了。因此，我们可以直接复用zuul维护的RequestContext即可。

2. 在每个请求结束时清空自定义上下文

   ![](image2019-10-11_12-22-31.png)

由图可知，无论*routing filters*成功与否，*post filters*都会被执行，其实通过刚刚的源码我们也可以验证这个执行流程。所以新增一个*post filter*,在其中执行清空自定义上下文。

```java
package com.shein.supply.wms.gateway.filter;
 
import com.netflix.zuul.ZuulFilter;
import com.shein.supply.wms.gateway.support.RibbonFilterContextHolder;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cloud.netflix.zuul.filters.support.FilterConstants;
import org.springframework.stereotype.Component;
 
/**
 * @author yanjing
 * date: 2019/10/11
 * description:
 */
@Component
public class ClearFilter extends ZuulFilter {
 
    private Logger logger = LoggerFactory.getLogger(this.getClass());
 
    @Override
    public String filterType() {
        return FilterConstants.POST_TYPE;
    }
 
    @Override
    public int filterOrder() {
        return 0;
    }
 
    @Override
    public boolean shouldFilter() {
        return true;
    }
 
    @Override
    public Object run() {
        //清除自定义的上下文
        logger.info("开始清除自定义的上下文，{}", RibbonFilterContextHolder.getCurrentContext());
        RibbonFilterContextHolder.clearCurrentContext();
        logger.info("清除自定义的上下文成功");
        return null;
    }
}
```

### 开发环境没有在配置文件指定ribbon执行自定义的ServiceTagsAwareRule，为何程序还能正常执行？

项目在本地运行时，我们在本地的default配置文件中加入了以下代码：

![](image2019-10-11_10-30-31.png)

然而在发布开发环境的时候，由于我个人的疏忽，没有将开发环境的配置文件加上上述配置代码，可是令人惊讶的是程序运行正常，和预期结果一致。那么框架又是如何自动加载我们自定义的负载均衡策略的呢？

原因：

在经过本地调试后，我们仔细看下RibbonDiscoveryRuleAutoConfiguration和RibbonClientConfiguration这俩个文件：

```java
package com.shein.supply.wms.gateway.support;
 
import com.shein.supply.wms.gateway.rule.ServiceTagsAwareRule;
import org.springframework.beans.factory.config.ConfigurableBeanFactory;
import org.springframework.boot.autoconfigure.AutoConfigureBefore;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.cloud.netflix.ribbon.RibbonClientConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Scope;
 
/**
 * The Ribbon discovery filter auto configuration.
 *
 * @author yanjing
 * date: 2019/9/30
 * description:
 */
@Configuration
@AutoConfigureBefore(RibbonClientConfiguration.class)
@ConditionalOnProperty(value = "ribbon.filter.tags.enabled", matchIfMissing = true)
public class RibbonDiscoveryRuleAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
    public ServiceTagsAwareRule serviceTagsAwareRule() {
        return new ServiceTagsAwareRule();
    }
}
```

这里主要注意一下@AutoConfigureBefore(RibbonClientConfiguration.class)这句注解，表示该配置会在RibbonClientConfiguration之前进行加载。然后我们再来看下RibbonClientConfiguration：

```java
/*
 * Copyright 2013-2014 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
 
package org.springframework.cloud.netflix.ribbon;
 
import java.net.URI;
 
import javax.annotation.PostConstruct;
 
import org.apache.http.client.params.ClientPNames;
import org.apache.http.client.params.CookiePolicy;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.cloud.commons.httpclient.HttpClientConfiguration;
import org.springframework.cloud.netflix.ribbon.apache.HttpClientRibbonConfiguration;
import org.springframework.cloud.netflix.ribbon.okhttp.OkHttpRibbonConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
 
import com.netflix.client.DefaultLoadBalancerRetryHandler;
import com.netflix.client.RetryHandler;
import com.netflix.client.config.CommonClientConfigKey;
import com.netflix.client.config.DefaultClientConfigImpl;
import com.netflix.client.config.IClientConfig;
import com.netflix.loadbalancer.ConfigurationBasedServerList;
import com.netflix.loadbalancer.DummyPing;
import com.netflix.loadbalancer.ILoadBalancer;
import com.netflix.loadbalancer.IPing;
import com.netflix.loadbalancer.IRule;
import com.netflix.loadbalancer.PollingServerListUpdater;
import com.netflix.loadbalancer.Server;
import com.netflix.loadbalancer.ServerList;
import com.netflix.loadbalancer.ServerListFilter;
import com.netflix.loadbalancer.ServerListUpdater;
import com.netflix.loadbalancer.ZoneAvoidanceRule;
import com.netflix.loadbalancer.ZoneAwareLoadBalancer;
import com.netflix.niws.client.http.RestClient;
import com.sun.jersey.api.client.Client;
import com.sun.jersey.client.apache4.ApacheHttpClient4;
 
import static com.netflix.client.config.CommonClientConfigKey.DeploymentContextBasedVipAddresses;
import static org.springframework.cloud.netflix.ribbon.RibbonUtils.setRibbonProperty;
import static org.springframework.cloud.netflix.ribbon.RibbonUtils.updateToHttpsIfNeeded;
 
/**
 * @author Dave Syer
 */
@SuppressWarnings("deprecation")
@Configuration
@EnableConfigurationProperties
//Order is important here, last should be the default, first should be optional
// see https://github.com/spring-cloud/spring-cloud-netflix/issues/2086#issuecomment-316281653
@Import({HttpClientConfiguration.class, OkHttpRibbonConfiguration.class, RestClientRibbonConfiguration.class, HttpClientRibbonConfiguration.class})
public class RibbonClientConfiguration {
 
    public static final int DEFAULT_CONNECT_TIMEOUT = 1000;
    public static final int DEFAULT_READ_TIMEOUT = 1000;
 
    @Value("${ribbon.client.name}")
    private String name = "client";
 
    // TODO: maybe re-instate autowired load balancers: identified by name they could be
    // associated with ribbon clients
 
    @Autowired
    private PropertiesFactory propertiesFactory;
 
    @Bean
    @ConditionalOnMissingBean
    public IClientConfig ribbonClientConfig() {
        DefaultClientConfigImpl config = new DefaultClientConfigImpl();
        config.loadProperties(this.name);
        config.set(CommonClientConfigKey.ConnectTimeout, DEFAULT_CONNECT_TIMEOUT);
        config.set(CommonClientConfigKey.ReadTimeout, DEFAULT_READ_TIMEOUT);
        return config;
    }
 
    @Bean
    @ConditionalOnMissingBean //不会进行加载，因为我们之前已经在RibbonDiscoveryRuleAutoConfiguration中加载了IRule的子类，也就是我们自定义的ServiceTagsAwareRule
    public IRule ribbonRule(IClientConfig config) {
        if (this.propertiesFactory.isSet(IRule.class, name)) {
            return this.propertiesFactory.get(IRule.class, config, name);
        }
        ZoneAvoidanceRule rule = new ZoneAvoidanceRule();
        rule.initWithNiwsConfig(config);
        return rule;
    }
 
    @Bean
    @ConditionalOnMissingBean
    public IPing ribbonPing(IClientConfig config) {
        if (this.propertiesFactory.isSet(IPing.class, name)) {
            return this.propertiesFactory.get(IPing.class, config, name);
        }
        return new DummyPing();
    }
 
    @Bean
    @ConditionalOnMissingBean
    @SuppressWarnings("unchecked")
    public ServerList<Server> ribbonServerList(IClientConfig config) {
        if (this.propertiesFactory.isSet(ServerList.class, name)) {
            return this.propertiesFactory.get(ServerList.class, config, name);
        }
        ConfigurationBasedServerList serverList = new ConfigurationBasedServerList();
        serverList.initWithNiwsConfig(config);
        return serverList;
    }
 
    @Bean
    @ConditionalOnMissingBean
    public ServerListUpdater ribbonServerListUpdater(IClientConfig config) {
        return new PollingServerListUpdater(config);
    }
 
 
    @Bean
    @ConditionalOnMissingBean //此处加载ribbonLoadBalancer，其中的IRule框架会自行匹配到我们之前已经加载好的ServiceTagsAwareRule
    public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
            ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
            IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
        if (this.propertiesFactory.isSet(ILoadBalancer.class, name)) {
            return this.propertiesFactory.get(ILoadBalancer.class, config, name);
        }
        return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList,
                serverListFilter, serverListUpdater);
    }
 
    @Bean
    @ConditionalOnMissingBean
    @SuppressWarnings("unchecked")
    public ServerListFilter<Server> ribbonServerListFilter(IClientConfig config) {
        if (this.propertiesFactory.isSet(ServerListFilter.class, name)) {
            return this.propertiesFactory.get(ServerListFilter.class, config, name);
        }
        ZonePreferenceServerListFilter filter = new ZonePreferenceServerListFilter();
        filter.initWithNiwsConfig(config);
        return filter;
    }
 
    @Bean
    @ConditionalOnMissingBean
    public RibbonLoadBalancerContext ribbonLoadBalancerContext(ILoadBalancer loadBalancer,
            IClientConfig config, RetryHandler retryHandler) {
        return new RibbonLoadBalancerContext(loadBalancer, config, retryHandler);
    }
 
    @Bean
    @ConditionalOnMissingBean
    public RetryHandler retryHandler(IClientConfig config) {
        return new DefaultLoadBalancerRetryHandler(config);
    }
 
    @Bean
    @ConditionalOnMissingBean
    public ServerIntrospector serverIntrospector() {
        return new DefaultServerIntrospector();
    }
 
    @PostConstruct
    public void preprocess() {
        setRibbonProperty(name, DeploymentContextBasedVipAddresses.key(), name);
    }
 
    static class OverrideRestClient extends RestClient {
 
        private IClientConfig config;
        private ServerIntrospector serverIntrospector;
 
        protected OverrideRestClient(IClientConfig config,
                ServerIntrospector serverIntrospector) {
            super();
            this.config = config;
            this.serverIntrospector = serverIntrospector;
            initWithNiwsConfig(this.config);
        }
 
        @Override
        public URI reconstructURIWithServer(Server server, URI original) {
            URI uri = updateToHttpsIfNeeded(original, this.config,
                    this.serverIntrospector, server);
            return super.reconstructURIWithServer(server, uri);
        }
 
        @Override
        protected Client apacheHttpClientSpecificInitialization() {
            ApacheHttpClient4 apache = (ApacheHttpClient4) super.apacheHttpClientSpecificInitialization();
            apache.getClientHandler().getHttpClient().getParams().setParameter(
                    ClientPNames.COOKIE_POLICY, CookiePolicy.IGNORE_COOKIES);
            return apache;
        }
 
    }
 
}
```

至此，谜题解开，由于我们优先加载了自定义的ServiceTagsAwareRule，所以RibbonClientConfiguration中默认的ribbonRule不会被加载，最后在加载ribbonLoadBalancer的时候，框架只能匹配到我们自定义的ServiceTagsAwareRule。

最后，我们可以开心的将指向自定义策略的配置省略了！

## 参考链接：

http://juke.outofmemory.cn/entry/253610

http://juke.outofmemory.cn/entry/253843

http://ju.outofmemory.cn/entry/253845

https://github.com/jmnarloch/ribbon-discovery-filter-spring-cloud-starter

https://cloud.spring.io/spring-cloud-netflix/multi/multi_spring-cloud-ribbon.html

https://github.com/Netflix/zuul/wiki/How-it-Works