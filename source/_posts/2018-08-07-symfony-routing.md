---
title: Symfony路由机制-一
date: 2018-08-07 17:01:38
tags:
- php
- symfony
- 路由
categories:
- [symfony]
---
路由组件是Symfony框架的核心组件之一，主要有如下功能：

* 根据URL或者Console命令匹配已配置的路由，映射到具体的控制器中
* 在模板（比如Twig）和控制器中生成优雅的URL
* 加载Bundle中路由资源（比如routing.yml）

Symfony在首次请求时会缓存路由，在测试环境下，会在/var/cache/dev下生成appProjectContainerUrlMatcher.php，之后的请求会直接调用该文件中的match方法，加快路由匹配速度。

比如在/app/config/routing.yml中，有如下配置
```yaml
default_page:
    path: /default/page/{page}
    controller: 'DemoBundle:Default:page'
    defaults:
        page: 1
    requirements:
        page: '\d+'
```
会生成下面的代码
```php
// default_page
if (0 === strpos($pathinfo, '/default/page') && preg_match('#^/default/page(?:/(?P<page>\\d+))?$#sD', $pathinfo, $matches)) {
    return $this->mergeDefaults(array_replace($matches, array('_route' => 'default_page')), array (  'page' => 1,  '_controller' => 'DemoBundle\\Controller\\DefaultController::pageAction',));
}
```

后面文章会详细介绍具体细节。