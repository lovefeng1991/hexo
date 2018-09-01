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
本文主要讲解路由规则是如何被缓存到PHP文件中的。

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

> 第一个tag节点表示事件订阅，第二个tag节点表示日志级别，其余argument节点表示服务实例化时所需的参数。
> \vendor\symfony\symfony\src\Symfony\Bundle\FrameworkBundle\Resources\config\routing.xml

## 容器编译
在依赖注入容器编译的过程中，会调用RegisterListenersPass.php中的process方法，获取所有配置了标签kernel.event_listener和kernel.event_subscriber的服务，然后会调用服务的getSubscribedEvents方法获取服务订阅的事件。

```php
public function process(ContainerBuilder $container)
{
    // 检查是否有事件分发服务
    if (!$container->hasDefinition($this->dispatcherService) && !$container->hasAlias($this->dispatcherService)) {
        return;
    }
	
    // 获取事件分发服务
    $definition = $container->findDefinition($this->dispatcherService);

    // 找到配置了kernel.event_listener标签的所有服务
    foreach ($container->findTaggedServiceIds($this->listenerTag, true) as $id => $events) {
        foreach ($events as $event) {
            // 事件优先级，默认值为0
            $priority = isset($event['priority']) ? $event['priority'] : 0;

            // 必须配置event属性，表示订阅了何种事件
            if (!isset($event['event'])) {
                throw new InvalidArgumentException(sprintf('Service "%s" must define the "event" attribute on "%s" tags.', $id, $this->listenerTag));
            }

            // 分发事件后调用的方法，对于kernel.request事件，默认值为onKernelRequest
            if (!isset($event['method'])) {
                $event['method'] = 'on'.preg_replace_callback(array(
                    '/(?<=\b)[a-z]/i',
                    '/[^a-z0-9]/i',
                ), function ($matches) { return strtoupper($matches[0]); }, $event['event']);
                $event['method'] = preg_replace('/[^a-z0-9]/i', '', $event['method']);
            }

            // 事件分发服务添加方法调用
            $definition->addMethodCall('addListener', array($event['event'], array(new ServiceClosureArgument(new Reference($id)), $event['method']), $priority));

            // FrameworkBundle在build的时候添加了5个hotPathEvents，分别是kernel.request，
            // kernel.controller，kernel.controller_arguments，kernel_response，kernel.finish_request
            if (isset($this->hotPathEvents[$event['event']])) {
                // 为服务添加标签container.hot_path，由ResolveHotPathPass.php处理
                $container->getDefinition($id)->addTag($this->hotPathTagName);
            }
        }
    }

    $extractingDispatcher = new ExtractingEventDispatcher();

    // 找到配置了kernel.event_subscriber标签的所有服务
    foreach ($container->findTaggedServiceIds($this->subscriberTag, true) as $id => $attributes) {
        $def = $container->getDefinition($id);

        // We must assume that the class value has been correctly filled, even if the service is created by a factory
        $class = $def->getClass();

        if (!$r = $container->getReflectionClass($class)) {
            throw new InvalidArgumentException(sprintf('Class "%s" used for service "%s" cannot be found.', $class, $id));
        }
        if (!$r->isSubclassOf(EventSubscriberInterface::class)) {
            throw new InvalidArgumentException(sprintf('Service "%s" must implement interface "%s".', $id, EventSubscriberInterface::class));
        }
        $class = $r->name;

        ExtractingEventDispatcher::$subscriber = $class;
        // 添加订阅者，这里会调用getSubscribedEvents方法返回订阅的事件
        $extractingDispatcher->addSubscriber($extractingDispatcher);
        foreach ($extractingDispatcher->listeners as $args) {
            // 这里处理方法和上面一样，可以看出来kernel.event_subscriber配置方法比kernel.event_listener更加灵活
            $args[1] = array(new ServiceClosureArgument(new Reference($id)), $args[1]);
            $definition->addMethodCall('addListener', $args);

            if (isset($this->hotPathEvents[$args[0]])) {
                $container->getDefinition($id)->addTag('container.hot_path');
            }
        }
        $extractingDispatcher->listeners = array();
    }
}
```

> 从这个方法可以看出，服务订阅事件只要配置kernel.event_listener和kernel.event_subscriber这两种标签中之一即可。
> 上面正则匹配用到了**零宽断言**，可以参考这篇教程：[正则表达式30分钟入门教程](https://deerchao.net/tutorials/regex/regex.htm "正则表达式30分钟入门教程")。
> \vendor\symfony\symfony\src\Symfony\Component\EventDispatcher\DependencyInjection\RegisterListenersPass.php

router_listener服务订阅了3个事件kernel.request、kernel.finish_request、kernel.exception。

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

> \vendor\symfony\symfony\src\Symfony\Component\HttpKernel\EventListener\RouterListener.php

## 处理请求
Symfony在处理请求时，会分发kernel.request事件。

```php
$event = new GetResponseEvent($this, $request, $type);
// 分发kernel.request事件
$this->dispatcher->dispatch(KernelEvents::REQUEST, $event);
```

在分发kernel.request事件过程中，因为router_listener服务订阅过该事件，因此会调用router_listener服务的onKernelRequest方法。

> \vendor\symfony\symfony\src\Symfony\Component\HttpKernel\HttpKernel.php

## onKernelRequest
onKernelRequest会调用匹配器来匹配请求。

```php
public function onKernelRequest(GetResponseEvent $event)
{
    // 获取请求对象
    $request = $event->getRequest();

    // 设置当前请求
    $this->setCurrentRequest($request);

    if ($request->attributes->has('_controller')) {
        // 路由匹配已经完成
        return;
    }

    // add attributes based on the request (routing)
    try {
        // 匹配请求
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

        // 请求对象属性添加参数
        $request->attributes->add($parameters);
        unset($parameters['_route'], $parameters['_controller']);
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
调用匹配器匹配请求。

```php
public function matchRequest(Request $request)
{
    // 匹配器通过缓存路由后得到，这是真正用于匹配请求的匹配器
    $matcher = $this->getMatcher();
    if (!$matcher instanceof RequestMatcherInterface) {
        // fallback to the default UrlMatcherInterface
        return $matcher->match($request->getPathInfo());
    }

    return $matcher->matchRequest($request);
}
```

> \vendor\symfony\symfony\src\Symfony\Component\Routing\Router.php

## 获取匹配器
在获取匹配器过程中，会缓存路由到PHP文件中。

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

## 获取匹配器Dumper
获取匹配器Dumper实例。

```php
protected function getMatcherDumperInstance()
{
    return new $this->options['matcher_dumper_class']($this->getRouteCollection());
}
```

> matcher_dumper_class指的是Symfony\Component\Routing\Matcher\Dumper\PhpMatcherDumper类，在配置文件\vendor\symfony\symfony\src\Symfony\Bundle\FrameworkBundle\Resources\config\routing.xml中可以看到。

获取路由集合。

```php
public function getRouteCollection()
{
    if (null === $this->collection) {
        // 调用routing.loader服务加载路由资源
        $this->collection = $this->container->get('routing.loader')->load($this->resource, $this->options['resource_type']);
        $this->resolveParameters($this->collection);
        $this->collection->addResource(new ContainerParametersResource($this->collectedParameters));
    }

    return $this->collection;
}
```
> $this->resource为routing_dev.yml，在配置文件\app\config\config_dev.yml中framework段可以看到。
> \vendor\symfony\symfony\src\Symfony\Bundle\FrameworkBundle\Routing\Router.php

## router.loader服务实例化
router.loader服务会在依赖注入容器编译过程中生成文件getRouting_LoaderService.php。

```
<?php

use Symfony\Component\DependencyInjection\Argument\RewindableGenerator;

// This file has been auto-generated by the Symfony Dependency Injection Component for internal use.
// Returns the public 'routing.loader' shared service.

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

$a = ${($_ = isset($this->services['file_locator']) ? $this->services['file_locator'] : $this->services['file_locator'] = new \Symfony\Component\HttpKernel\Config\FileLocator(${($_ = isset($this->services['kernel']) ? $this->services['kernel'] : $this->get('kernel', 1)) && false ?: '_'}, ($this->targetDirs[3].'\\app/Resources'), array(0 => ($this->targetDirs[3].'\\app')))) && false ?: '_'};
$b = ${($_ = isset($this->services['annotation_reader']) ? $this->services['annotation_reader'] : $this->getAnnotationReaderService()) && false ?: '_'};

$c = new \Symfony\Bundle\FrameworkBundle\Routing\AnnotatedRouteControllerLoader($b);

$d = new \Symfony\Component\Config\Loader\LoaderResolver();
$d->addLoader(new \Symfony\Component\Routing\Loader\XmlFileLoader($a));
$d->addLoader(new \Symfony\Component\Routing\Loader\YamlFileLoader($a));
$d->addLoader(new \Symfony\Component\Routing\Loader\PhpFileLoader($a));
$d->addLoader(new \Symfony\Component\Routing\Loader\GlobFileLoader($a));
$d->addLoader(new \Symfony\Component\Routing\Loader\DirectoryLoader($a));
$d->addLoader(new \Symfony\Component\Routing\Loader\DependencyInjection\ServiceRouterLoader($this));
$d->addLoader($c);
$d->addLoader(new \Symfony\Component\Routing\Loader\AnnotationDirectoryLoader($a, $c));
$d->addLoader(new \Symfony\Component\Routing\Loader\AnnotationFileLoader($a, $c));

return $this->services['routing.loader'] = new \Symfony\Bundle\FrameworkBundle\Routing\DelegatingLoader(${($_ = isset($this->services['controller_name_converter']) ? $this->services['controller_name_converter'] : $this->services['controller_name_converter'] = new \Symfony\Bundle\FrameworkBundle\Controller\ControllerNameParser(${($_ = isset($this->services['kernel']) ? $this->services['kernel'] : $this->get('kernel', 1)) && false ?: '_'})) && false ?: '_'}, $d);
```

可以看到routing.loader即为DelegatingLoader，DelegatingLoader字面意思为**委托加载器**，并不是最终加载资源的加载器，它会利用LoaderResolver来确定最终的加载器。从上面可以看到LoaderResolver添加了9种加载器，也可以看出Symfony中路由配置非常灵活。

> 在routing.loader服务实例化的同时，也实例化了file_locator服务，有些加载器中会用到。

```php
public function load($resource, $type = null)
{
    if ($this->loading) {
        throw new FileLoaderLoadException($resource, null, null, null, $type);
    }
    $this->loading = true;

    try {
        // 调用父类load函数
        $collection = parent::load($resource, $type);
    } finally {
        $this->loading = false;
    }

    foreach ($collection->all() as $route) {
        // 必须配置controller项或者defaults中_controller项，并且不能为空
        if (!\is_string($controller = $route->getDefault('_controller')) || !$controller) {
            continue;
        }

        try {
            // 解析控制器
            $controller = $this->parser->parse($controller);
        } catch (\InvalidArgumentException $e) {
            // unable to optimize unknown notation
        }

        $route->setDefault('_controller', $controller);
    }

    return $collection;
}
```

可以看出它会调用父类的load方法。

> \vendor\symfony\symfony\src\Symfony\Bundle\FrameworkBundle\Routing\DelegatingLoader.php

```php
public function load($resource, $type = null)
{
    // 调用LoaderResolver解析资源
    if (false === $loader = $this->resolver->resolve($resource, $type)) {
        throw new FileLoaderLoadException($resource, null, null, null, $type);
    }

    // 加载资源
    return $loader->load($resource, $type);
}
```

> \vendor\symfony\symfony\src\Symfony\Component\Config\Loader\DelegatingLoader.php

LoaderResolver会遍历检查9种加载器，通过调用supports方法来判断最终使用的加载器。

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

> \vendor\symfony\symfony\src\Symfony\Component\Config\Loader\LoaderResolver.php

路由资源为routing_dev.yml，因此会使用Yaml文件加载器YamlFileLoader。下面是YamlFileLoader的supports方法。

```php
public function supports($resource, $type = null)
{
    return \is_string($resource) && \in_array(pathinfo($resource, PATHINFO_EXTENSION), array('yml', 'yaml'), true) && (!$type || 'yaml' === $type);
}
```
> \vendor\symfony\symfony\src\Symfony\Component\Routing\Loader\YamlFileLoader.php

## 加载路由资源
加载路由资源routing_dev.yml过程中，有如下配置，

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

routing.yml一般为主路由配置，Symfony会在以下三个地址中寻找routing.yml文件
1. 加载器当前所在目录，一般为\app\config
2. \app\Resources（全局资源存放位置）
3. \app（前面都找不到的话，在这个目录下找）

> \app\config\routing_dev.yml

按顺序只会解析其中一个，具体配置可以在下面配置文件中找到，

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

```php
public function load($file, $type = null)
{
    // 定位器会检查文件是否存在，并将别名地址转换为绝对地址
    $path = $this->locator->locate($file);

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

    // 设置自定义错误处理函数，也就是解析yml文件时会用到这个错误处理函数
    $prevErrorHandler = set_error_handler(function ($level, $message, $script, $line) use ($file, &$prevErrorHandler) {
        $message = E_USER_DEPRECATED === $level ? preg_replace('/ on line \d+/', ' in "'.$file.'"$0', $message) : $message;

        return $prevErrorHandler ? $prevErrorHandler($level, $message, $script, $line) : false;
    });

    try {
        // 将yml配置文件解析为PHP数组
        $parsedConfig = $this->yamlParser->parseFile($path);
    } catch (ParseException $e) {
        throw new \InvalidArgumentException(sprintf('The file "%s" does not contain valid YAML.', $path), 0, $e);
    } finally {
        // 还原错误处理函数
        restore_error_handler();
    }

    $collection = new RouteCollection();
    // 添加资源
    $collection->addResource(new FileResource($path));

    // 文件为空
    if (null === $parsedConfig) {
        return $collection;
    }

    // not an array
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

YamlFileLoader类中有个私有静态属性$availableKeys，这里定义了路由配置的所有选项。

```php
private static $availableKeys = array(
    'resource', 'type', 'prefix', 'path', 'host', 'schemes', 'methods', 'defaults', 'requirements', 'options', 'condition', 'controller',
);
```

> resource：资源路径，相对地址或者是绝对地址
> type：类型，比如annotation
> prefix：前缀，与resource项搭配使用
> path：路径
> host：主机
> schemes：协议，http或者https
> methods：请求类型，比如get、post
> defaults：默认值，与path项搭配使用
> requirements：条件，一般为正则表达式
> options：选项，比如{'utf8': true}表示支持utf8
> condition：条件，使用[**The Expression Syntax**](https://symfony.com/doc/3.4/components/expression_language/syntax.html "The Expression Syntax")
> controller：映射的控制器，比如EmployeesBundle:Default:employees
> \vendor\symfony\symfony\src\Symfony\Component\Routing\Loader\YamlFileLoader.php

YamlFileLoader类中validate函数用来验证路由配置是否合法，从这里可以看到路由配置需要遵守的规则。

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
    // 配置项controller和defaults中的_controller不能同时出现，必须配置其中一个
    if (isset($config['controller']) && isset($config['defaults']['_controller'])) {
        throw new \InvalidArgumentException(sprintf('The routing file "%s" must not specify both the "controller" key and the defaults key "_controller" for "%s".', $path, $name));
    }
}
```

> \vendor\symfony\symfony\src\Symfony\Component\Routing\Loader\YamlFileLoader.php

首先看parseRoute这个函数，这个函数相对简单。

```php
protected function parseRoute(RouteCollection $collection, $name, array $config, $path)
{
    // 在配置了path的情况下，会考虑path、defaults、requirements、options、
    // host、schemes、methods、condition、controller这9个配置项
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

    // 添加路由
    $collection->add($name, $route);
}
```
> \vendor\symfony\symfony\src\Symfony\Component\Routing\Loader\YamlFileLoader.php

在来看parseImport这个函数。

```php
protected function parseImport(RouteCollection $collection, array $config, $path, $file)
{
    // 在配置了resource的情况下，会考虑resource、type、prefix、defaults、requirements、options
    // host、condition、schemes、methods这10个配置项
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

    // 设置当前目录
    $this->setCurrentDir(\dirname($path));

    // 导入路由资源，返回的是子路由集合
    $imported = $this->import($config['resource'], $type, false, $file);

    if (!\is_array($imported)) {
        $imported = array($imported);
    }

    foreach ($imported as $subCollection) {
        // 添加前缀
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

> \vendor\symfony\symfony\src\Symfony\Component\Routing\Loader\YamlFileLoader.php

在配置了resource的情况下，需要结合type的值重新确定加载器，这里会调用YamlFileLoader父类FileLoader的import函数。

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
            // 遍历文件并导入
            if (null !== $res = $this->doImport($path, $type, $ignoreErrors, $sourceResource)) {
                $ret[] = $res;
            }
            $isSubpath = true;
        }

        if ($isSubpath) {
            return isset($ret[1]) ? $ret : (isset($ret[0]) ? $ret[0] : null);
        }
    }

    // 导入文件
    return $this->doImport($resource, $type, $ignoreErrors, $sourceResource);
}
```

> \vendor\symfony\symfony\src\Symfony\Component\Config\Loader\FileLoader.php

在什么情况下，会使用到glob函数呢？请看下面的例子。

```yaml
# 这是\app\config\routing.yml文件中的配置
app:
    resource: '@AppBundle/Controller/'
    type: annotation

# 可以这样配置，将会解析@EmployeesBundle/Resources/config/routing下所有yaml文件
employees:
    resource: '@EmployeesBundle/Resources/config/routing/*.yml'

# 也可以这样配置，将会解析@EmployeesBundle/Resources/config/routing下所有yaml文件，并使用GlobFileLoader加载器
employees:
    resource: '@EmployeesBundle/Resources/config/routing/*.yml'
    type: 'glob'

# 也可以这样配置，将会解析@EmployeesBundle/Resources/config/routing下所有yaml和xml文件
employees:
    resource: '@EmployeesBundle/Resources/config/routing/*.{yml,xml}'

# 还可以这样配置，将会解析@EmployeesBundle/Resources/config/routing下ess.yml和rss.yml文件
employees:
    resource: '@EmployeesBundle/Resources/config/routing/[er]ss.yml'

# 还可以这样配置，将会解析@EmployeesBundle/Resources/config/routing下ess.yml、eas.yml等文件
# 注意?与正则表达式中?含义不一样，这里的?表示除/以外的一个字符
employees:
    resource: '@EmployeesBundle/Resources/config/routing/e?s.yml'
```

> golb模式匹配的规则具体看[glob函数](http://php.net/manual/zh/function.glob.php "glob")下的第一个用户评论。

接着来看doImport函数。

```php
private function doImport($resource, $type = null, $ignoreErrors = false, $sourceResource = null)
{
    try {
        // 调用LoaderResolver解析资源，结合type项的值确定具体使用的加载器，前面已经分析过
        $loader = $this->resolve($resource, $type);

        if ($loader instanceof self && null !== $this->currentDir) {
            // 定位文件，只有在$resource为相对地址时，返回的$resource才可能是数组
            $resource = $loader->getLocator()->locate($resource, $this->currentDir, false);
        }

        $resources = \is_array($resource) ? $resource : array($resource);
        for ($i = 0; $i < $resourcesCount = \count($resources); ++$i) {
            if (isset(self::$loading[$resources[$i]])) {
                if ($i == $resourcesCount - 1) {
                    throw new FileLoaderImportCircularReferenceException(array_keys(self::$loading));
                }
            } else {
                $resource = $resources[$i];
                break;
            }
        }
        self::$loading[$resource] = true;

        try {
            // 加载资源，返回RouteCollection，和加载routing_dev.yml过程一样
            $ret = $loader->load($resource, $type);
        } finally {
            unset(self::$loading[$resource]);
        }

        // 从这里可以看到，当找到多个文件时，比如routing.yml，只解析第一个
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
至此，所有路由配置解析完毕，返回RouteCollection对象。RouteCollection类中有两个属性，routes保存所有的路由信息，是一个Route对象数组，resources保存已经解析过的资源文件，是一个实现了SelfCheckingResourceInterface接口的对象数组，比如FileResource。

> **注意：**当resource为相对地址时，FileLocator会在3个路径中搜索该文件，具体看[加载路由资源](#加载路由资源)
> \vendor\symfony\symfony\src\Symfony\Component\Config\Loader\FileLoader.php
# 未完待续