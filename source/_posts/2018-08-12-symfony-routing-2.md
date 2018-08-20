---
title: Symfony路由机制
date: 2018-08-12 15:54:23
tags:
- php
- symfony
- 路由
categories:
- [symfony]
---
本文主要讲解路由规则是如何被缓存到PHP文件中的。

## 订阅事件
服务通过配置kernel.event_subscriber标签来订阅事件。

```xml
<service id="router_listener" class="Symfony\Component\HttpKernel\EventListener\RouterListener">
    <tag name="kernel.event_subscriber" />
    <tag name="monolog.logger" channel="request" />
    <argument type="service" id="router" />
    <argument type="service" id="request_stack" />
    <argument type="service" id="router.request_context" on-invalid="ignore" />
    <argument type="service" id="logger" on-invalid="ignore" />
    <argument>%kernel.project_dir%</argument>
    <argument>%kernel.debug%</argument>
</service>
```

> \vendor\symfony\symfony\src\Symfony\Bundle\FrameworkBundle\Resources\config\routing.xml

## 容器编译
在依赖注入容器编译的过程中，会调用RegisterListenersPass.php中的process方法，然后会调用router_listener服务中的getSubscribedEvents方法，

```php
public static function getSubscribedEvents()
{
    return array(
        KernelEvents::REQUEST => array(array('onKernelRequest', 32)),
        KernelEvents::FINISH_REQUEST => array(array('onKernelFinishRequest', 0)),
        KernelEvents::EXCEPTION => array('onKernelException', -64),
    );
}
```

Symfony会为router_listener服务订阅3个事件kernel.request、kernel.finish_request、kernel.exception

> \vendor\symfony\symfony\src\Symfony\Component\EventDispatcher\DependencyInjection\RegisterListenersPass.php
> \vendor\symfony\symfony\src\Symfony\Component\HttpKernel\EventListener\RouterListener.php

## 处理请求
Symfony在处理请求时，会分发kernel.request事件。

```php
$event = new GetResponseEvent($this, $request, $type);
$this->dispatcher->dispatch(KernelEvents::REQUEST, $event);
```

在分发kernel.request事件过程中，会调用router_listener服务的onKernelRequest方法。

> \vendor\symfony\symfony\src\Symfony\Component\HttpKernel\HttpKernel.php

## 匹配请求
获取匹配器来匹配请求。

```php
if ($this->matcher instanceof RequestMatcherInterface) {
    $parameters = $this->matcher->matchRequest($request);
} else {
    $parameters = $this->matcher->match($request->getPathInfo());
}
```

> \vendor\symfony\symfony\src\Symfony\Component\Routing\Router.php

## 缓存路由
在获取匹配器过程中，会缓存路由到PHP文件中。缓存得到的PHP文件才是真正用于匹配请求的匹配器

```php
$cache = $this->getConfigCacheFactory()->cache($this->options['cache_dir'].'/'.$this->options['matcher_cache_class'].'.php',
    function (ConfigCacheInterface $cache) {
        $dumper = $this->getMatcherDumperInstance();
        if (method_exists($dumper, 'addExpressionLanguageProvider')) {
            foreach ($this->expressionLanguageProviders as $provider) {
                $dumper->addExpressionLanguageProvider($provider);
            }
        }

        $options = array(
            'class' => $this->options['matcher_cache_class'],
            'base_class' => $this->options['matcher_base_class'],
        );

        $cache->write($dumper->dump($options), $this->getRouteCollection()->getResources());
    }
);
```

> \vendor\symfony\symfony\src\Symfony\Component\Routing\Router.php