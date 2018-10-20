---
title: Symfony路由机制-二
date: 2018-08-12 15:54:23
tags:
- php
- symfony
- 路由
- 正则表达式
categories:
- [symfony]
---
本文主要讲解Symfony3.4加载路由配置文件并将它解析为RouteCollection对象。

## 订阅事件
router_listener服务通过配置kernel.event_subscriber标签来订阅事件。

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

下面简单介绍每个节点的含义：
- 第一个tag节点表示事件订阅。
- 第二个tag节点表示日志级别。
- 其余argument节点表示服务实例化时所需的参数。

详细分析会在**依赖注入组件**的文章中。

> \vendor\symfony\symfony\src\Symfony\Bundle\FrameworkBundle\Resources\config\routing.xml

## 容器编译
在容器编译的过程中，会调用所有Bundle的DependencyInjection\Compiler目录下以Pass结尾的类中的process函数，其中不同类具有不同的职责。RegisterListenersPass.php负责事件订阅部分，它的process函数会获取所有配置了kernel.event_listener和kernel.event_subscriber标签的服务，具有kernel.event_subscriber标签的服务会调用该服务的getSubscribedEvents方法获取服务订阅的事件。

> **注意：**自定义服务，这里指事件订阅器，在配置了kernel.event_subscriber标签的情况下，必须实现EventSubscriberInterface接口的getSubscribedEvents方法。

RegisterListenersPass类的process函数：

```php
public function process(ContainerBuilder $container)
{
    // 检查是否有事件分发服务，即event_dispatcher服务
    if (!$container->hasDefinition($this->dispatcherService) && !$container->hasAlias($this->dispatcherService)) {
        return;
    }
	
    // 获取事件分发服务
    $definition = $container->findDefinition($this->dispatcherService);

    // 找到配置kernel.event_listener标签的所有服务
    foreach ($container->findTaggedServiceIds($this->listenerTag, true) as $id => $events) {
        foreach ($events as $event) {
            // 优先级属性（可选），默认值为0
            $priority = isset($event['priority']) ? $event['priority'] : 0;

            // 事件属性（必选），表示订阅的事件
            if (!isset($event['event'])) {
                throw new InvalidArgumentException(sprintf('Service "%s" must define the "event" attribute on "%s" tags.', $id, $this->listenerTag));
            }

            // 方法属性（可选），回调函数，对于kernel.request事件，默认值为onKernelRequest
            if (!isset($event['method'])) {
                // 单词首字母大写
                $event['method'] = 'on'.preg_replace_callback(array(
                    '/(?<=\b)[a-z]/i',          // 匹配单词的首字母
                    '/[^a-z0-9]/i',             // 匹配字符串中除字母和数字之外的字符 
                ), function ($matches) { return strtoupper($matches[0]); }, $event['event']);
                // 替换字符串中非字母和数字的字符为空
                $event['method'] = preg_replace('/[^a-z0-9]/i', '', $event['method']);
            }

            // 事件分发服务添加回调函数
            $definition->addMethodCall('addListener', array($event['event'], array(new ServiceClosureArgument(new Reference($id)), $event['method']), $priority));

            // FrameworkBundle在build的时候添加了5个hotPathEvents，分别是kernel.request，
            // kernel.controller，kernel.controller_arguments，kernel_response，kernel.finish_request
            if (isset($this->hotPathEvents[$event['event']])) {
                // 为服务添加container.hot_path标签，由ResolveHotPathPass.php的process函数处理
                $container->getDefinition($id)->addTag($this->hotPathTagName);
            }
        }
    }

    $extractingDispatcher = new ExtractingEventDispatcher();

    // 找到配置kernel.event_subscriber标签的所有服务
    foreach ($container->findTaggedServiceIds($this->subscriberTag, true) as $id => $attributes) {
        // 获取服务定义
        $def = $container->getDefinition($id);

        // 获取类名
        $class = $def->getClass();

        // 获取反射类
        if (!$r = $container->getReflectionClass($class)) {
            throw new InvalidArgumentException(sprintf('Class "%s" used for service "%s" cannot be found.', $class, $id));
        }

        // 事件订阅器必须实现EventSubscriberInterface接口
        if (!$r->isSubclassOf(EventSubscriberInterface::class)) {
            throw new InvalidArgumentException(sprintf('Service "%s" must implement interface "%s".', $id, EventSubscriberInterface::class));
        }
        // 获取类名
        $class = $r->name;

        ExtractingEventDispatcher::$subscriber = $class;
        // 添加订阅者，调用getSubscribedEvents方法返回订阅的事件
        $extractingDispatcher->addSubscriber($extractingDispatcher);
        // 遍历监听器
        foreach ($extractingDispatcher->listeners as $args) {
            // 和上面类似
            $args[1] = array(new ServiceClosureArgument(new Reference($id)), $args[1]);
            // 添加回调
            $definition->addMethodCall('addListener', $args);

            // 和上面类似
            if (isset($this->hotPathEvents[$args[0]])) {
                $container->getDefinition($id)->addTag('container.hot_path');
            }
        }
        $extractingDispatcher->listeners = array();
    }
}
```

> **/(?<=\b)[a-z]/i**用到了正则表达式中的**零宽断言**，可以参考这篇教程：[正则表达式30分钟入门教程](https://deerchao.net/tutorials/regex/regex.htm "正则表达式30分钟入门教程")。
> 
> \vendor\symfony\symfony\src\Symfony\Component\EventDispatcher\DependencyInjection\RegisterListenersPass.php

RouterListener类的getSubscribedEvents函数：

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

显然，router_listener服务通过getSubscribedEvents函数订阅了kernel.request、kernel.finish_request、kernel.exception这3个事件。也就是说,配置
```xml
<tag name="kernel.event_subscriber" />
```
与配置
```xml
<tag name="kernel.event_listener" priority="32" event="kernel.request" method="onKernelRequest" />
<tag name="kernel.event_listener" event="kernel.finish_request" method="onKernelFinishRequest" />
<tag name="kernel.event_listener" priority="-64" event="kernel.exception" method="onKernelException" />
```
效果一样，可见kernel.event_subscriber配置方法比kernel.event_listener灵活的多。

详细分析会在之后**事件分发组件**的文章中。

> 服务订阅事件可以配置kernel.event_listener标签和kernel.event_subscriber标签，只需要定义一个标签即可。
>> **注意：**当两个标签配置同一个事件的时候，后者并不会覆盖前者，会在事件分发的时候，执行两遍。
> 
> \vendor\symfony\symfony\src\Symfony\Component\HttpKernel\EventListener\RouterListener.php

## 处理请求
Symfony3.4在处理请求时，会分发kernel.request事件。在分发过程中，router_listener服务会调用回调函数onKernelRequest。

```php
$event = new GetResponseEvent($this, $request, $type);
// 分发kernel.request事件
$this->dispatcher->dispatch(KernelEvents::REQUEST, $event);
```

> \vendor\symfony\symfony\src\Symfony\Component\HttpKernel\HttpKernel.php

RouterListener类的onKernelRequest函数：

```php
public function onKernelRequest(GetResponseEvent $event)
{
    // 获取请求对象
    $request = $event->getRequest();

    // 设置当前请求
    $this->setCurrentRequest($request);

    if ($request->attributes->has('_controller')) {
        // 路由匹配已经完成。如果一个事件订阅两次，这里会提前返回，避免无效工作
        return;
    }

    try {
        // 调用匹配器来匹配请求
        if ($this->matcher instanceof RequestMatcherInterface) {
            $parameters = $this->matcher->matchRequest($request);
        } else {
            $parameters = $this->matcher->match($request->getPathInfo());
        }

        if (null !== $this->logger) {
            $this->logger->info('Matched route "{route}".', array(
                'route' => isset($parameters['_route']) ? $parameters['_route'] : 'n/a',
                'route_parameters' => $parameters,
                'request_uri' => $request->getUri(),
                'method' => $request->getMethod(),
            ));
        }

        // 参数添加到请求对象中
        $request->attributes->add($parameters);
        unset($parameters['_route'], $parameters['_controller']);
        // 请求对象设置路由参数
        $request->attributes->set('_route_params', $parameters);
    } catch (ResourceNotFoundException $e) {
        $message = sprintf('No route found for "%s %s"', $request->getMethod(), $request->getPathInfo());

        if ($referer = $request->headers->get('referer')) {
            $message .= sprintf(' (from "%s")', $referer);
        }

        throw new NotFoundHttpException($message, $e);
    } catch (MethodNotAllowedException $e) {
        $message = sprintf('No route found for "%s %s": Method Not Allowed (Allow: %s)', $request->getMethod(), $request->getPathInfo(), implode(', ', $e->getAllowedMethods()));

        throw new MethodNotAllowedHttpException($e->getAllowedMethods(), $message, $e);
    }
}
```

> \vendor\symfony\symfony\src\Symfony\Component\HttpKernel\EventListener\RouterListener.php

## 匹配请求
Router类的matchRequest函数：

```php
public function matchRequest(Request $request)
{
    // 通过缓存路由后得到的匹配器，才是真正用于匹配请求的匹配器
    $matcher = $this->getMatcher();
    if (!$matcher instanceof RequestMatcherInterface) {
        // fallback to the default UrlMatcherInterface
        return $matcher->match($request->getPathInfo());
    }

    // 匹配请求
    return $matcher->matchRequest($request);
}
```

该函数获取真正匹配请求的匹配器。

> \vendor\symfony\symfony\src\Symfony\Component\Routing\Router.php

## 匹配器
Router类的getMatcher函数：

```php
public function getMatcher()
{
    if (null !== $this->matcher) {
        return $this->matcher;
    }

    if (null === $this->options['cache_dir'] || null === $this->options['matcher_cache_class']) {
        $this->matcher = new $this->options['matcher_class']($this->getRouteCollection(), $this->context);
        if (method_exists($this->matcher, 'addExpressionLanguageProvider')) {
            foreach ($this->expressionLanguageProviders as $provider) {
                $this->matcher->addExpressionLanguageProvider($provider);
            }
        }

        return $this->matcher;
    }

    $cache = $this->getConfigCacheFactory()->cache($this->options['cache_dir'].'/'.$this->options['matcher_cache_class'].'.php',
        function (ConfigCacheInterface $cache) {
            // 获取匹配器Dumper实例
            $dumper = $this->getMatcherDumperInstance();
            if (method_exists($dumper, 'addExpressionLanguageProvider')) {
                foreach ($this->expressionLanguageProviders as $provider) {
                    $dumper->addExpressionLanguageProvider($provider);
                }
            }

            $options = array(
                // 匹配器类名
                'class' => $this->options['matcher_cache_class'],
                // 匹配器父类名
                'base_class' => $this->options['matcher_base_class'],
            );

            // 缓存路由
            $cache->write($dumper->dump($options), $this->getRouteCollection()->getResources());
        }
    );

    if (!class_exists($this->options['matcher_cache_class'], false)) {
        require_once $cache->getPath();
    }

    // 真正匹配器
    return $this->matcher = new $this->options['matcher_cache_class']($this->context);
}
```

> \vendor\symfony\symfony\src\Symfony\Component\Routing\Router.php

## 匹配器Dumper
Router类的getMatcherDumperInstance函数：

```php
protected function getMatcherDumperInstance()
{
    return new $this->options['matcher_dumper_class']($this->getRouteCollection());
}
```

> matcher_dumper_class：Symfony\Component\Routing\Matcher\Dumper\PhpMatcherDumper
> 
> \vendor\symfony\symfony\src\Symfony\Component\Routing\Router.php

该函数使用解析路由配置获得的RouteCollection对象实例化PhpMatcherDumper类。见下面配置。

```xml
<parameters>PhpMatcherDumper
    <parameter key="router.options.generator_class">Symfony\Component\Routing\Generator\UrlGenerator</parameter>
    <parameter key="router.options.generator_base_class">Symfony\Component\Routing\Generator\UrlGenerator</parameter>
    <parameter key="router.options.generator_dumper_class">Symfony\Component\Routing\Generator\Dumper\PhpGeneratorDumper</parameter>
    <parameter key="router.options.matcher_class">Symfony\Bundle\FrameworkBundle\Routing\RedirectableUrlMatcher</parameter>
    <parameter key="router.options.matcher_base_class">Symfony\Bundle\FrameworkBundle\Routing\RedirectableUrlMatcher</parameter>
    <parameter key="router.options.matcher_dumper_class">Symfony\Component\Routing\Matcher\Dumper\PhpMatcherDumper</parameter>
    <parameter key="router.options.matcher.cache_class">%router.cache_class_prefix%UrlMatcher</parameter>
    <parameter key="router.options.generator.cache_class">%router.cache_class_prefix%UrlGenerator</parameter>
    <parameter key="router.request_context.host">localhost</parameter>
    <parameter key="router.request_context.scheme">http</parameter>
    <parameter key="router.request_context.base_url"></parameter>
</parameters>
```

> \vendor\symfony\symfony\src\Symfony\Bundle\FrameworkBundle\Resources\config\routing.xml

## 路由集合
Router类的getRouteCollection函数：

```php
public function getRouteCollection()
{
    if (null === $this->collection) {
        // 调用routing.loader服务加载路由资源
        $this->collection = $this->container->get('routing.loader')->load($this->resource, $this->options['resource_type']);
        // 解析参数，解析配置中%包围的参数
        $this->resolveParameters($this->collection);
        // 添加资源
        $this->collection->addResource(new ContainerParametersResource($this->collectedParameters));
    }

    return $this->collection;
}
```

> \vendor\symfony\symfony\src\Symfony\Bundle\FrameworkBundle\Routing\Router.php

$this->resource即routing_dev.yml。见下面配置。

```yaml
framework:
    router:
        resource: '%kernel.project_dir%/app/config/routing_dev.yml'
        strict_requirements: true
    profiler: { only_exceptions: false }
```

> \app\config\config_dev.yml

## router.loader服务
router.loader服务会在容器编译过程中生成文件getRouting_LoaderService.php。

```
<?php

use Symfony\Component\DependencyInjection\Argument\RewindableGenerator;

include_once $this->targetDirs[3].'\\vendor\\symfony\\symfony\\src\\Symfony\\Component\\Config\\Loader\\LoaderInterface.php';
include_once $this->targetDirs[3].'\\vendor\\symfony\\symfony\\src\\Symfony\\Component\\Config\\Loader\\Loader.php';
include_once $this->targetDirs[3].'\\vendor\\symfony\\symfony\\src\\Symfony\\Component\\Config\\Loader\\FileLoader.php';
include_once $this->targetDirs[3].'\\vendor\\symfony\\symfony\\src\\Symfony\\Component\\Routing\\Loader\\XmlFileLoader.php';
include_once $this->targetDirs[3].'\\vendor\\symfony\\symfony\\src\\Symfony\\Component\\Routing\\Loader\\YamlFileLoader.php';
include_once $this->targetDirs[3].'\\vendor\\symfony\\symfony\\src\\Symfony\\Component\\Routing\\Loader\\PhpFileLoader.php';
include_once $this->targetDirs[3].'\\vendor\\symfony\\symfony\\src\\Symfony\\Component\\Routing\\Loader\\GlobFileLoader.php';
include_once $this->targetDirs[3].'\\vendor\\symfony\\symfony\\src\\Symfony\\Component\\Routing\\Loader\\DirectoryLoader.php';
include_once $this->targetDirs[3].'\\vendor\\symfony\\symfony\\src\\Symfony\\Component\\Routing\\Loader\\ObjectRouteLoader.php';
include_once $this->targetDirs[3].'\\vendor\\symfony\\symfony\\src\\Symfony\\Component\\Routing\\Loader\\DependencyInjection\\ServiceRouterLoader.php';
include_once $this->targetDirs[3].'\\vendor\\symfony\\symfony\\src\\Symfony\\Component\\Routing\\Loader\\AnnotationClassLoader.php';
include_once $this->targetDirs[3].'\\vendor\\symfony\\symfony\\src\\Symfony\\Bundle\\FrameworkBundle\\Routing\\AnnotatedRouteControllerLoader.php';
include_once $this->targetDirs[3].'\\vendor\\symfony\\symfony\\src\\Symfony\\Component\\Routing\\Loader\\AnnotationFileLoader.php';
include_once $this->targetDirs[3].'\\vendor\\symfony\\symfony\\src\\Symfony\\Component\\Routing\\Loader\\AnnotationDirectoryLoader.php';
include_once $this->targetDirs[3].'\\vendor\\symfony\\symfony\\src\\Symfony\\Component\\Config\\Loader\\LoaderResolverInterface.php';
include_once $this->targetDirs[3].'\\vendor\\symfony\\symfony\\src\\Symfony\\Component\\Config\\Loader\\LoaderResolver.php';
include_once $this->targetDirs[3].'\\vendor\\symfony\\symfony\\src\\Symfony\\Component\\Config\\Loader\\DelegatingLoader.php';
include_once $this->targetDirs[3].'\\vendor\\symfony\\symfony\\src\\Symfony\\Bundle\\FrameworkBundle\\Routing\\DelegatingLoader.php';
include_once $this->targetDirs[3].'\\vendor\\symfony\\symfony\\src\\Symfony\\Component\\Config\\FileLocatorInterface.php';
include_once $this->targetDirs[3].'\\vendor\\symfony\\symfony\\src\\Symfony\\Component\\Config\\FileLocator.php';
include_once $this->targetDirs[3].'\\vendor\\symfony\\symfony\\src\\Symfony\\Component\\HttpKernel\\Config\\FileLocator.php';

// file_locator服务实例化
$a = ${($_ = isset($this->services['file_locator']) ? $this->services['file_locator'] : $this->services['file_locator'] = new \Symfony\Component\HttpKernel\Config\FileLocator(${($_ = isset($this->services['kernel']) ? $this->services['kernel'] : $this->get('kernel', 1)) && false ?: '_'}, ($this->targetDirs[3].'\\app/Resources'), array(0 => ($this->targetDirs[3].'\\app')))) && false ?: '_'};
// annotation_reader服务实例化
$b = ${($_ = isset($this->services['annotation_reader']) ? $this->services['annotation_reader'] : $this->getAnnotationReaderService()) && false ?: '_'};

$c = new \Symfony\Bundle\FrameworkBundle\Routing\AnnotatedRouteControllerLoader($b);

// 加载器解析器
$d = new \Symfony\Component\Config\Loader\LoaderResolver();
// xml文件加载器
$d->addLoader(new \Symfony\Component\Routing\Loader\XmlFileLoader($a));
// yaml文件加载器
$d->addLoader(new \Symfony\Component\Routing\Loader\YamlFileLoader($a));
// php文件加载器
$d->addLoader(new \Symfony\Component\Routing\Loader\PhpFileLoader($a));
// glob文件加载器
$d->addLoader(new \Symfony\Component\Routing\Loader\GlobFileLoader($a));
// 目录加载器
$d->addLoader(new \Symfony\Component\Routing\Loader\DirectoryLoader($a));
$d->addLoader(new \Symfony\Component\Routing\Loader\DependencyInjection\ServiceRouterLoader($this));
$d->addLoader($c);
// annotation目录加载器
$d->addLoader(new \Symfony\Component\Routing\Loader\AnnotationDirectoryLoader($a, $c));
// annotation文件加载器
$d->addLoader(new \Symfony\Component\Routing\Loader\AnnotationFileLoader($a, $c));

return $this->services['routing.loader'] = new \Symfony\Bundle\FrameworkBundle\Routing\DelegatingLoader(${($_ = isset($this->services['controller_name_converter']) ? $this->services['controller_name_converter'] : $this->services['controller_name_converter'] = new \Symfony\Bundle\FrameworkBundle\Controller\ControllerNameParser(${($_ = isset($this->services['kernel']) ? $this->services['kernel'] : $this->get('kernel', 1)) && false ?: '_'})) && false ?: '_'}, $d);
```

routing.loader即DelegatingLoader，DelegatingLoader字面意思为**委托加载器**，它不具有加载资源的功能，但会利用LoaderResolver来获取加载资源的加载器。routing.loader实例化过程中LoaderResolver会添加9种加载器，因此Symfony3.4支持多种格式的路由配置，比如yaml、xml、php、annotation。

> **注意：**有些加载器实例化的时候需要file_locator服务，这个服务会定位资源的位置。

## 加载路由资源
DelegatingLoader类的load函数：

```php
public function load($resource, $type = null)
{
    if ($this->loading) {
        throw new FileLoaderLoadException($resource, null, null, null, $type);
    }
    // 资源正在加载
    $this->loading = true;

    try {
        // 调用父类load函数
        $collection = parent::load($resource, $type);
    } finally {
        $this->loading = false;
    }

    foreach ($collection->all() as $route) {
        // 不解析无效的值
        if (!\is_string($controller = $route->getDefault('_controller')) || !$controller) {
            continue;
        }

        try {
            // 解析控制器，EmployeesBundle:Default:employees将被解析为EmployeesBundle\Controller\DefaultController::employeesAction
            $controller = $this->parser->parse($controller);
        } catch (\InvalidArgumentException $e) {
            // 注意这里当controller的值不合理时，Symfony虽然会抛出异常，但没处理
        }

        // 替换为完整的控制器
        $route->setDefault('_controller', $controller);
    }

    return $collection;
}
```

> \vendor\symfony\symfony\src\Symfony\Bundle\FrameworkBundle\Routing\DelegatingLoader.php

DelegatingLoader类的load函数：

```php
public function load($resource, $type = null)
{
    // 调用LoaderResolver解析资源，确定加载器
    if (false === $loader = $this->resolver->resolve($resource, $type)) {
        throw new FileLoaderLoadException($resource, null, null, null, $type);
    }

    // 加载资源
    return $loader->load($resource, $type);
}
```

> \vendor\symfony\symfony\src\Symfony\Component\Config\Loader\DelegatingLoader.php

LoaderResolver类的resolve函数：

```php
public function resolve($resource, $type = null)
{
    foreach ($this->loaders as $loader) {
        if ($loader->supports($resource, $type)) {
            return $loader;
        }
    }

    return false;
}
```

LoaderResolver会遍历9种加载器，通过调用supports方法来判断将会使用的加载器。

> \vendor\symfony\symfony\src\Symfony\Component\Config\Loader\LoaderResolver.php

比如YamlFileLoader类的supports函数：

```php
public function supports($resource, $type = null)
{
    return \is_string($resource) && \in_array(pathinfo($resource, PATHINFO_EXTENSION), array('yml', 'yaml'), true) && (!$type || 'yaml' === $type);
}
```

> \vendor\symfony\symfony\src\Symfony\Component\Routing\Loader\YamlFileLoader.php

routing_dev.yml配置如下。

```yaml
_wdt:
    resource: '@WebProfilerBundle/Resources/config/routing/wdt.xml'
    prefix: /_wdt

_profiler:
    resource: '@WebProfilerBundle/Resources/config/routing/profiler.xml'
    prefix: /_profiler

_errors:
    resource: '@TwigBundle/Resources/config/routing/errors.xml'
    prefix: /_error

_main:
    resource: routing.yml
```

routing.yml一般为主路由配置，Symfony3.4通过加载它去加载自定义Bundle中的路由资源。FileLocator会在以下3个路径中寻找routing.yml文件。
1. routing_dev.yml所在目录，一般为\app\config
2. \app\Resources（全局资源存放位置）
3. \app（前面都找不到的话，在这个目录下找）

> **注意：**按顺序解析，只会解析一个。
> 
> \app\config\routing_dev.yml

上面路径的配置见下面。

```xml
<service id="file_locator" class="Symfony\Component\HttpKernel\Config\FileLocator">
    <argument type="service" id="kernel" />
    <argument>%kernel.root_dir%/Resources</argument>
    <argument type="collection">
        <argument>%kernel.root_dir%</argument>
    </argument>
</service>
<service id="Symfony\Component\HttpKernel\Config\FileLocator" alias="file_locator" />
```

> \vendor\symfony\symfony\src\Symfony\Bundle\FrameworkBundle\Resources\config\services.xml

这里只分析Yaml文件加载器，其余加载器类似。YamlFileLoader类的load函数：

```php
public function load($file, $type = null)
{
    // 定位器检查文件是否存在，将别名地址（以@开头）转换为绝对地址，在路径中寻找相对地址
    $path = $this->locator->locate($file);

    // 必须为本地文件
    if (!stream_is_local($path)) {
        throw new \InvalidArgumentException(sprintf('This is not a local file "%s".', $path));
    }

    if (!file_exists($path)) {
        throw new \InvalidArgumentException(sprintf('File "%s" not found.', $path));
    }

    if (null === $this->yamlParser) {
        // Yaml解析器
        $this->yamlParser = new YamlParser();
    }

    // 设置自定义错误处理函数，解析yaml文件出错时会用到这个错误处理函数
    $prevErrorHandler = set_error_handler(function ($level, $message, $script, $line) use ($file, &$prevErrorHandler) {
        $message = E_USER_DEPRECATED === $level ? preg_replace('/ on line \d+/', ' in "'.$file.'"$0', $message) : $message;

        return $prevErrorHandler ? $prevErrorHandler($level, $message, $script, $line) : false;
    });

    try {
        // 解析yaml文件，返回PHP数组
        $parsedConfig = $this->yamlParser->parseFile($path);
    } catch (ParseException $e) {
        throw new \InvalidArgumentException(sprintf('The file "%s" does not contain valid YAML.', $path), 0, $e);
    } finally {
        // 还原错误处理函数
        restore_error_handler();
    }

    // 初始化RouteCollection类
    $collection = new RouteCollection();
    // 添加资源
    $collection->addResource(new FileResource($path));

    // yaml文件为空
    if (null === $parsedConfig) {
        return $collection;
    }

    // 非数组
    if (!\is_array($parsedConfig)) {
        throw new \InvalidArgumentException(sprintf('The file "%s" must contain a YAML array.', $path));
    }

    foreach ($parsedConfig as $name => $config) {
        // 验证路由配置是否合法
        $this->validate($config, $name, $path);

        if (isset($config['resource'])) {
            // 解析导入
            $this->parseImport($collection, $config, $path, $file);
        } else {
            // 解析路由
            $this->parseRoute($collection, $name, $config, $path);
        }
    }

    return $collection;
}
```

> \vendor\symfony\symfony\src\Symfony\Component\Routing\Loader\YamlFileLoader.php

YamlFileLoader类中有个私有静态属性$availableKeys，定义了路由配置的所有选项。

```php
private static $availableKeys = array(
    'resource', 'type', 'prefix', 'path', 'host', 'schemes', 'methods', 
    'defaults', 'requirements', 'options', 'condition', 'controller',
);
```

> \vendor\symfony\symfony\src\Symfony\Component\Routing\Loader\YamlFileLoader.php

配置项具体含义如下：
- resource：资源路径，相对地址或者绝对地址或者别名地址（以@开头）
- type：类型，比如annotation、glob
- prefix：前缀，与resource搭配使用
- path：路由路径
- host：主机，比如blog.pangpang.fun
- schemes：表示URI schemes，常见的比如http、https
- methods：请求类型，比如get、post
- defaults：默认值，与path项搭配使用，其中_controller非常重要
- requirements：一般为正则表达式
- options：选项，比如{'utf8': true}表示支持utf8
- condition：条件，使用[**The Expression Syntax**](https://symfony.com/doc/3.4/components/expression_language/syntax.html "The Expression Syntax")
- controller：映射的控制器，比如EmployeesBundle:Default:employees

> **注意：**schemes表示URI schemes，具体见[https://en.wikipedia.org/wiki/Uniform_Resource_Identifier#Generic_syntax](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier#Generic_syntax "URI")。常见schemes有：http、https、ftp、mailto，对于web应用来说，scheme一般表示http和https。

validate函数用来验证路由配置是否合法，可以看到路由配置需要遵守的约定。YamlFileLoader类的validate函数：

```php
protected function validate($config, $name, $path)
{
    // 配置必须为数组 
    if (!\is_array($config)) {
        throw new \InvalidArgumentException(sprintf('The definition of "%s" in "%s" must be a YAML array.', $name, $path));
    }
    // 配置项必须在$availableKeys数组中
    if ($extraKeys = array_diff(array_keys($config), self::$availableKeys)) {
        throw new \InvalidArgumentException(sprintf(
            'The routing file "%s" contains unsupported keys for "%s": "%s". Expected one of: "%s".',
            $path, $name, implode('", "', $extraKeys), implode('", "', self::$availableKeys)
        ));
    }
    // 配置项resource和path不能同时出现
    if (isset($config['resource']) && isset($config['path'])) {
        throw new \InvalidArgumentException(sprintf(
            'The routing file "%s" must not specify both the "resource" key and the "path" key for "%s". Choose between an import and a route definition.',
            $path, $name
        ));
    }
    // 配置项type必须与resource同时出现
    if (!isset($config['resource']) && isset($config['type'])) {
        throw new \InvalidArgumentException(sprintf(
            'The "type" key for the route definition "%s" in "%s" is unsupported. It is only available for imports in combination with the "resource" key.',
            $name, $path
        ));
    }
    // 配置项resource和path至少出现一个
    if (!isset($config['resource']) && !isset($config['path'])) {
        throw new \InvalidArgumentException(sprintf(
            'You must define a "path" for the route "%s" in file "%s".',
            $name, $path
        ));
    }
    // 配置项controller和defaults中的_controller不能同时出现
    if (isset($config['controller']) && isset($config['defaults']['_controller'])) {
        throw new \InvalidArgumentException(sprintf('The routing file "%s" must not specify both the "controller" key and the defaults key "_controller" for "%s".', $path, $name));
    }
}
```

> \vendor\symfony\symfony\src\Symfony\Component\Routing\Loader\YamlFileLoader.php

YamlFileLoader类的parseRoute函数：

```php
protected function parseRoute(RouteCollection $collection, $name, array $config, $path)
{
    // 获取各个配置项的值，没设置的话，设置默认值
    $defaults = isset($config['defaults']) ? $config['defaults'] : array();
    $requirements = isset($config['requirements']) ? $config['requirements'] : array();
    $options = isset($config['options']) ? $config['options'] : array();
    $host = isset($config['host']) ? $config['host'] : '';
    $schemes = isset($config['schemes']) ? $config['schemes'] : array();
    $methods = isset($config['methods']) ? $config['methods'] : array();
    $condition = isset($config['condition']) ? $config['condition'] : null;

    // controller配置项的值会保存到defaults配置项中的_controller
    if (isset($config['controller'])) {
        $defaults['_controller'] = $config['controller'];
    }

    // 实例化路由
    $route = new Route($config['path'], $defaults, $requirements, $options, $host, $schemes, $methods, $condition);

    // 添加路由，重复定义的情况，会按先后顺序只保留后者
    $collection->add($name, $route);
}
```

> **注意：**在配置了path的情况下，path、defaults、requirements、options、host、schemes、methods、condition、controller这9个配置项有效。
> **注意：**实例化Route过程中，会为options添加默认值{'compiler_class': 'Symfony\\Component\\Routing\\RouteCompiler'}。
> 
> \vendor\symfony\symfony\src\Symfony\Component\Routing\Loader\YamlFileLoader.php

YamlFileLoader类的parseImport函数：

```php
protected function parseImport(RouteCollection $collection, array $config, $path, $file)
{
    // 获取各个配置项的值，没设置的话，设置默认值
    $type = isset($config['type']) ? $config['type'] : null;
    $prefix = isset($config['prefix']) ? $config['prefix'] : '';
    $defaults = isset($config['defaults']) ? $config['defaults'] : array();
    $requirements = isset($config['requirements']) ? $config['requirements'] : array();
    $options = isset($config['options']) ? $config['options'] : array();
    $host = isset($config['host']) ? $config['host'] : null;
    $condition = isset($config['condition']) ? $config['condition'] : null;
    $schemes = isset($config['schemes']) ? $config['schemes'] : null;
    $methods = isset($config['methods']) ? $config['methods'] : null;

    // 同parseRoute函数
    if (isset($config['controller'])) {
        $defaults['_controller'] = $config['controller'];
    }

    // 设置当前目录，主要用于FileLocator中
    $this->setCurrentDir(\dirname($path));

    // 导入路由资源，返回的是子路由集合
    $imported = $this->import($config['resource'], $type, false, $file);

    if (!\is_array($imported)) {
        $imported = array($imported);
    }

    foreach ($imported as $subCollection) {
        // 添加前缀，作用于path
        $subCollection->addPrefix($prefix);
        if (null !== $host) {
            // 设置主机，会覆盖$subCollection中的host
            $subCollection->setHost($host);
        }
        if (null !== $condition) {
            // 设置条件，会覆盖$subCollection中的condition
            $subCollection->setCondition($condition);
        }
        if (null !== $schemes) {
            // 设置协议，会覆盖$subCollection中的schemes
            $subCollection->setSchemes($schemes);
        }
        if (null !== $methods) {
            // 设置请求方法，会覆盖$subCollection中的methods
            $subCollection->setMethods($methods);
        }
        // 添加默认
        $subCollection->addDefaults($defaults);
        // 添加条件
        $subCollection->addRequirements($requirements);
        // 添加选项
        $subCollection->addOptions($options);

        // 添加子集合
        $collection->addCollection($subCollection);
    }
}
```

> **注意：**在配置了resource的情况下，resource、type、prefix、defaults、requirements、options、host、condition、schemes、methods这10个配置项有效。
> 
> \vendor\symfony\symfony\src\Symfony\Component\Routing\Loader\YamlFileLoader.php

RouteCollection类的addPrefix函数：

```php
public function addPrefix($prefix, array $defaults = array(), array $requirements = array())
{
    // 去掉prefix首尾空格字符和/
    $prefix = trim(trim($prefix), '/');

    // 为空直接返回。
    if ('' === $prefix) {
        return;
    }

    foreach ($this->routes as $route) {
        // path前拼接prefix，重新设置path
        $route->setPath('/'.$prefix.$route->getPath());
        $route->addDefaults($defaults);
        $route->addRequirements($requirements);
    }
}
```

> \vendor\symfony\symfony\src\Symfony\Component\Routing\RouteCollection.php

这个函数比较简单，作用就是遍历集合中所有路由，在path前面添加前缀。RouteCollection类中并没有setPrefix函数，因此prefix需要配合resource配置项使用，为resource内所有路由添加前缀。
配置了resource的情况下，由于resource的配置非常灵活，这里需要调用父类的import函数确定具体的加载器并重复上面步骤。FileLoader类的import函数：

```php
public function import($resource, $type = null, $ignoreErrors = false, $sourceResource = null)
{
    // 通过正则匹配来判断是否使用glob函数，第一次看到strcspn函数的使用
    if (\is_string($resource) && \strlen($resource) !== $i = strcspn($resource, '*?{[')) {
        $ret = array();
        // 是否是子路径
        $isSubpath = 0 !== $i && false !== strpos(substr($resource, 0, $i), '/');
        // 这个函数就不分析了，主要是使用PHP系统函数glob函数获取符合条件的文件并返回
        foreach ($this->glob($resource, false, $_, $ignoreErrors || !$isSubpath) as $path => $info) {
            // 导入文件
            if (null !== $res = $this->doImport($path, $type, $ignoreErrors, $sourceResource)) {
                $ret[] = $res;
            }
            $isSubpath = true;
        }

        if ($isSubpath) {
            return isset($ret[1]) ? $ret : (isset($ret[0]) ? $ret[0] : null);
        }
    }

    return $this->doImport($resource, $type, $ignoreErrors, $sourceResource);
}
```

> \vendor\symfony\symfony\src\Symfony\Component\Config\Loader\FileLoader.php

在什么情况下会使用到glob函数呢？看下面的例子。

```yaml
# 这是\app\config\routing.yml文件中的配置
app:
    resource: '@AppBundle/Controller/'
    type: annotation

# 将会解析@EmployeesBundle/Resources/config/routing下所有yaml文件
employees:
    resource: '@EmployeesBundle/Resources/config/routing/*.yml'

# 将会解析@EmployeesBundle/Resources/config/routing下所有yaml和xml文件
employees:
    resource: '@EmployeesBundle/Resources/config/routing/*.{yml,xml}'

# 将会解析@EmployeesBundle/Resources/config/routing下ess.yml和rss.yml文件
employees:
    resource: '@EmployeesBundle/Resources/config/routing/[er]ss.yml'

# 将会解析@EmployeesBundle/Resources/config/routing下ess.yml、eas.yml等文件
# 注意?与正则表达式中?含义不一样，这里的?表示除/以外的一个字符
employees:
    resource: '@EmployeesBundle/Resources/config/routing/e?s.yml'
```

> glob模式匹配的规则可以看[glob函数](http://php.net/manual/zh/function.glob.php "glob")下的第一个用户评论。

FileLoader类的doImport函数：

```php
private function doImport($resource, $type = null, $ignoreErrors = false, $sourceResource = null)
{
    try {
        // 调用LoaderResolver解析资源，结合type项的值确定具体使用的加载器，前面已经分析过
        $loader = $this->resolve($resource, $type);

        if ($loader instanceof self && null !== $this->currentDir) {
            // 定位文件
            $resource = $loader->getLocator()->locate($resource, $this->currentDir, false);
        }

        $resources = \is_array($resource) ? $resource : array($resource);
        for ($i = 0; $i < $resourcesCount = \count($resources); ++$i) {
            if (isset(self::$loading[$resources[$i]])) {
                if ($i == $resourcesCount - 1) {
                    // 循环导入异常
                    throw new FileLoaderImportCircularReferenceException(array_keys(self::$loading));
                }
            } else {
                // 当找到多个文件时，只解析第一个
                $resource = $resources[$i];
                break;
            }
        }
        self::$loading[$resource] = true;

        try {
            // 加载资源，重复上面步骤，返回RouteCollection
            $ret = $loader->load($resource, $type);
        } finally {
            unset(self::$loading[$resource]);
        }

        return $ret;
    } catch (FileLoaderImportCircularReferenceException $e) {
        throw $e;
    } catch (\Exception $e) {
        if (!$ignoreErrors) {
            // prevent embedded imports from nesting multiple exceptions
            if ($e instanceof FileLoaderLoadException) {
                throw $e;
            }

            throw new FileLoaderLoadException($resource, $sourceResource, null, $e, $type);
        }
    }
}
```

## 解析控制器
返回看[加载路由资源](#加载路由资源)，DelegatingLoader类的load函数在加载完路由配置后，需要进一步解析controller配置项，比如EmployeesBundle:Default:employees将被解析为EmployeesBundle\Controller\DefaultController::employeesAction。controller配置一般分为3个部分，Bundle、Controller和Action，使用:连接。ControllerNameParser类的parse函数：

```php
public function parse($controller)
{
    // 以:为分隔符分割字符串
    $parts = explode(':', $controller);
    // controller需要定义三个部分，Bundle、Controller和Action
    if (3 !== \count($parts) || \in_array('', $parts, true)) {
        throw new \InvalidArgumentException(sprintf('The "%s" controller is not a valid "a:b:c" controller string.', $controller));
    }

    $originalController = $controller;
    list($bundle, $controller, $action) = $parts;
    $controller = str_replace('/', '\\', $controller);
    $bundles = array();

    try {
        // 获取Bundle
        $allBundles = $this->kernel->getBundle($bundle, false, true);
    } catch (\InvalidArgumentException $e) {
        $message = sprintf(
            'The "%s" (from the _controller value "%s") does not exist or is not enabled in your kernel!',
            $bundle,
            $originalController
        );

        if ($alternative = $this->findAlternative($bundle)) {
            $message .= sprintf(' Did you mean "%s:%s:%s"?', $alternative, $controller, $action);
        }

        throw new \InvalidArgumentException($message, 0, $e);
    }

    if (!\is_array($allBundles)) {
        // happens when HttpKernel is version 4+
        $allBundles = array($allBundles);
    }

    foreach ($allBundles as $b) {
        // 拼接出完整的控制器
        $try = $b->getNamespace().'\\Controller\\'.$controller.'Controller';
        if (class_exists($try)) {
            // 控制器存在直接返回
            return $try.'::'.$action.'Action';
        }

        $bundles[] = $b->getName();
        $msg = sprintf('The _controller value "%s:%s:%s" maps to a "%s" class, but this class was not found. Create this class or check the spelling of the class and its namespace.', $bundle, $controller, $action, $try);
    }

    if (\count($bundles) > 1) {
        $msg = sprintf('Unable to find controller "%s:%s" in bundles %s.', $bundle, $controller, implode(', ', $bundles));
    }

    throw new \InvalidArgumentException($msg);
}
```

> **注意：**自定义Bundle必须在AppKernel中注册，即在registerBundles函数中声明，否则上面函数中getBundle函数会找不到指定的Bundle。
> 
> \vendor\symfony\symfony\src\Symfony\Bundle\FrameworkBundle\Controller\ControllerNameParser.php

## 解析参数
返回看[路由集合](#路由集合)，在加载完路由配置后，需要解析其中的参数。Router中类的resolveParameters函数：

```php
private function resolveParameters(RouteCollection $collection)
{
    // 
    foreach ($collection as $route) {
        // 解析defaults配置项
        foreach ($route->getDefaults() as $name => $value) {
            $route->setDefault($name, $this->resolve($value));
        }

        // 解析requirements配置项
        foreach ($route->getRequirements() as $name => $value) {
            $route->setRequirement($name, $this->resolve($value));
        }

        // 解析path配置项
        $route->setPath($this->resolve($route->getPath()));
        // 解析host配置项
        $route->setHost($this->resolve($route->getHost()));

        $schemes = array();
        // 解析schemes配置项
        foreach ($route->getSchemes() as $scheme) {
            $schemes = array_merge($schemes, explode('|', $this->resolve($scheme)));
        }
        $route->setSchemes($schemes);

        $methods = array();
        // 解析methods配置项
        foreach ($route->getMethods() as $method) {
            $methods = array_merge($methods, explode('|', $this->resolve($method)));
        }
        $route->setMethods($methods);
        // 解析condition配置项
        $route->setCondition($this->resolve($route->getCondition()));
    }
}
```

> **注意：**可以看出，schemes和methods有两种配置方法，以yaml格式为例，有methods: [get, post]和methods: get|post两种配置方法。
> 
> \vendor\symfony\symfony\src\Symfony\Bundle\FrameworkBundle\Routing\Router.php

resolve函数比较简单，主要解析由%包含的参数。Router类的resolve函数：

```php
private function resolve($value)
{
    // 参数为数组的情况，递归解析
    if (\is_array($value)) {
        foreach ($value as $key => $val) {
            $value[$key] = $this->resolve($val);
        }

        return $value;
    }

    // 参数不是字符串，直接返回
    if (!\is_string($value)) {
        return $value;
    }

    // 容器
    $container = $this->container;

    // 解析%包围的参数，比如%host%
    $escapedValue = preg_replace_callback('/%%|%([^%\s]++)%/', function ($match) use ($container, $value) {
        // skip %%
        if (!isset($match[1])) {
            return '%%';
        }

        if (preg_match('/^env\(\w+\)$/', $match[1])) {
            throw new RuntimeException(sprintf('Using "%%%s%%" is not allowed in routing configuration.', $match[1]));
        }

        // 获取参数实际值
        $resolved = $container->getParameter($match[1]);

        if (\is_string($resolved) || is_numeric($resolved)) {
            $this->collectedParameters[$match[1]] = $resolved;

            return (string) $resolved;
        }

        throw new RuntimeException(sprintf(
            'The container parameter "%s", used in the route configuration value "%s", '.
            'must be a string or numeric, but it is of type %s.',
            $match[1],
            $value,
            \gettype($resolved)
            )
        );
    }, $value);

    return str_replace('%%', '%', $escapedValue);
}
```

> **注意：**一般自定义参数会在\app\config\parameters.yml和\app\config\services.yml中定义，定义后可以在其他地方以%key%的形式调用。其中\app\config\parameters.yml包含全局自定义参数，\app\config\services.yml包含服务自定义参数。
> 
> \vendor\symfony\symfony\src\Symfony\Bundle\FrameworkBundle\Routing\Router.php

至此，所有路由配置解析完毕，返回RouteCollection对象。RouteCollection类中有两个属性，routes属性保存所有的路由信息，是一个Route对象数组，resources保存已经解析过的资源文件，是一个实现了ResourceInterface接口的对象数组，比如FileResource、DirectoryResource。

> ResourceInterface：Symfony\Component\Config\Resource\ResourceInterface