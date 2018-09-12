---
title: Symfony路由机制-三
date: 2018-08-29 18:08:09
tags:
- php
- symfony
- 路由
categories:
- [symfony]
---
本文主要讲解Symfony3.4如何缓存路由。

上一篇文章提到匹配请求的匹配器实际是缓存路由后生成的PHP文件，而PHP文件中的代码主要由PhpMatcherDumper类中dump函数生成，这也是本篇文章分析的重点。

PhpMatcherDumper类的dump函数：

```php
public function dump(array $options = array())
{
    // 传入的$options会覆盖下面的值
    $options = array_replace(array(
        // 匹配器类名
        'class' => 'ProjectUrlMatcher',
        // 匹配器父类名
        'base_class' => 'Symfony\\Component\\Routing\\Matcher\\UrlMatcher',
    ), $options);

    $interfaces = class_implements($options['base_class']);
    $supportsRedirections = isset($interfaces['Symfony\\Component\\Routing\\Matcher\\RedirectableUrlMatcherInterface']);

    return <<<EOF
<?php

use Symfony\Component\Routing\Exception\MethodNotAllowedException;
use Symfony\Component\Routing\Exception\ResourceNotFoundException;
use Symfony\Component\Routing\RequestContext;

/**
* This class has been auto-generated
* by the Symfony Routing Component.
*/
class {$options['class']} extends {$options['base_class']}
{
public function __construct(RequestContext \$context)
{
    // 初始化$context，这个值可用于condition配置项中
    \$this->context = \$context;
}

// 生成匹配器类match函数的代码
{$this->generateMatchMethod($supportsRedirections)}
}

EOF;
}
```

> \vendor\symfony\symfony\src\Symfony\Component\Routing\Matcher\Dumper\PhpMatcherDumper.php

PhpMatcherDumper类的generateMatchMethod函数：

```php
private function generateMatchMethod($supportsRedirections)
{
    // 编译路由
    $code = rtrim($this->compileRoutes($this->getRoutes(), $supportsRedirections), "\n");

    // match函数的代码
    return <<<EOF
public function match(\$rawPathinfo)
{
    \$allow = array();
    \$pathinfo = rawurldecode(\$rawPathinfo);
    \$trimmedPathinfo = rtrim(\$pathinfo, '/');
    \$context = \$this->context;
    \$request = \$this->request ?: \$this->createRequest(\$pathinfo);
    \$requestMethod = \$canonicalMethod = \$context->getMethod();

    if ('HEAD' === \$requestMethod) {
        \$canonicalMethod = 'GET';
    }

$code

    throw 0 < count(\$allow) ? new MethodNotAllowedException(array_unique(\$allow)) : new ResourceNotFoundException();
}
EOF;
}
```

> \vendor\symfony\symfony\src\Symfony\Component\Routing\Matcher\Dumper\PhpMatcherDumper.php

PhpMatcherDumper类的compileRoutes函数：

```php
private function compileRoutes(RouteCollection $routes, $supportsRedirections)
{
    $fetchedHost = false;
    // 根据host正则对路由分组
    $groups = $this->groupRoutesByHostRegex($routes);
    $code = '';

    foreach ($groups as $collection) {
        if (null !== $regex = $collection->getAttribute('host_regex')) {
            // 在host正则不为空的情况下，需要获取请求对象中host的值
            if (!$fetchedHost) {
                // 这段代码只需要生成一次
                $code .= "        \$host = \$context->getHost();\n\n";
                $fetchedHost = true;
            }

            $code .= sprintf("        if (preg_match(%s, \$host, \$hostMatches)) {\n", var_export($regex, true));
        }

        // 构建静态前缀路由集合
        $tree = $this->buildStaticPrefixCollection($collection);
        // 根据路由静态前缀分组，编译路由后生成代码
        $groupCode = $this->compileStaticPrefixRoutes($tree, $supportsRedirections);

        if (null !== $regex) {
            // 添加缩进
            $groupCode = preg_replace('/^.{2,}$/m', '    $0', $groupCode);
            $code .= $groupCode;
            // 与前面host正则不为空对应，添加闭合标签}
            $code .= "        }\n\n";
        } else {
            $code .= $groupCode;
        }
    }

    // used to display the Welcome Page in apps that don't define a homepage
    $code .= "        if ('/' === \$pathinfo && !\$allow) {\n";
    $code .= "            throw new Symfony\Component\Routing\Exception\NoConfigurationException();\n";
    $code .= "        }\n";

    return $code;
}
```

> **注意：**在解析host配置项时，host配置项的值会被解析为字符串，因此host配置项需要使用正则表达式。host配置项使用场景一般用来区分桌面版和移动版页面，比如host: (www|m).pangpang.fun。
> 
> **注意：**正则表达式中m选项表示多行模式，^和$匹配每行的开头和结尾。
> 
> \vendor\symfony\symfony\src\Symfony\Component\Routing\Matcher\Dumper\PhpMatcherDumper.php

groupRoutesByHostRegex函数会将host正则相同的路由分为同一组，同组路由具有其连续性。在分组过程中，会编译路由。PhpMatcherDumper类的groupRoutesByHostRegex函数：

```php
private function groupRoutesByHostRegex(RouteCollection $routes)
{
    // 路由组
    $groups = new DumperCollection();
    // 当前组
    $currentGroup = new DumperCollection();
    $currentGroup->setAttribute('host_regex', null);
    // 添加子组
    $groups->add($currentGroup);

    foreach ($routes as $name => $route) {
        // 编译路由，获取host模式串对应的正则表达式
        $hostRegex = $route->compile()->getHostRegex();
        // 在当前组host正则不相等情况下，会新增子路由组
        if ($currentGroup->getAttribute('host_regex') !== $hostRegex) {
            $currentGroup = new DumperCollection();
            $currentGroup->setAttribute('host_regex', $hostRegex);
            $groups->add($currentGroup);
        }
        // 添加DumperRoute
        $currentGroup->add(new DumperRoute($name, $route));
    }

    return $groups;
}
```

> **注意：**host正则相同不代表host配置项的值相同，比如host: "{secondary}.pangpang.fun",requirements: {secondary: www|m}与host: "{secondary}.pangpang.fun"编译生成的正则表达式完全不同。
> 
> \vendor\symfony\symfony\src\Symfony\Component\Routing\Matcher\Dumper\PhpMatcherDumper.php

Route类的compile函数：

```php
public function compile()
{
    if (null !== $this->compiled) {
        return $this->compiled;
    }

    // 编译类，即RouteCompiler类
    $class = $this->getOption('compiler_class');

    return $this->compiled = $class::compile($this);
}
```

> \vendor\symfony\symfony\src\Symfony\Component\Routing\Route.php

compile函数会编译host模式串和path模式串，结合其他配置项，比如defaults、requirements，生成正则表达式。RouteCompiler类的compile函数：

```php
public static function compile(Route $route)
{
    $hostVariables = array();
    $variables = array();
    $hostRegex = null;
    $hostTokens = array();

    if ('' !== $host = $route->getHost()) {
        // 编译host模式串
        $result = self::compilePattern($route, $host, true);

        // host模式串中的变量
        $hostVariables = $result['variables'];
        $variables = $hostVariables;

        $hostTokens = $result['tokens'];
        // host模式串对应的正则表达式
        $hostRegex = $result['regex'];
    }
    
    // 获取path模式串
    $path = $route->getPath();

    // 编译path模式串
    $result = self::compilePattern($route, $path, false);

    // 静态前缀
    $staticPrefix = $result['staticPrefix'];

    // path模式串中的变量数组
    $pathVariables = $result['variables'];

    foreach ($pathVariables as $pathParam) {
        // path模式串中不能包含_fragment变量，也就是说_fragment不能在path配置项中使用，
        // 但能在defaults配置项中使用，虽然没啥作用，_fragment一般用于生成URL中
        if ('_fragment' === $pathParam) {
            throw new \InvalidArgumentException(sprintf('Route pattern "%s" cannot contain "_fragment" as a path parameter.', $route->getPath()));
        }
    }

    // 合并host模式串和path模式串的变量数组
    $variables = array_merge($variables, $pathVariables);

    $tokens = $result['tokens'];
    // path模式串对应的正则表达式
    $regex = $result['regex'];

    return new CompiledRoute(
        $staticPrefix,
        $regex,
        $tokens,
        $pathVariables,
        $hostRegex,
        $hostTokens,
        $hostVariables,
        array_unique($variables)
    );
}
```
> **注意：**host模式串和path模式串能使用同一变量，但不推荐。
> 
> \vendor\symfony\symfony\src\Symfony\Component\Routing\RouteCompiler.php

RouteCompiler类的compilePattern函数：

```php
private static function compilePattern(Route $route, $pattern, $isHost)
{
    $tokens = array();
    $variables = array();
    $matches = array();
    $pos = 0;
    // 默认分隔符，编译host模式串时，分割符为.；编译path模式串时，分隔符为/
    $defaultSeparator = $isHost ? '.' : '/';
    // 模式串是否使用utf8字符集
    $useUtf8 = preg_match('//u', $pattern);
    // 获取utf8选项
    $needsUtf8 = $route->getOption('utf8');

    if (!$needsUtf8 && $useUtf8 && preg_match('/[\x80-\xFF]/', $pattern)) {
        $needsUtf8 = true;
        @trigger_error(sprintf('Using UTF-8 route patterns without setting the "utf8" option is deprecated since Symfony 3.2 and will throw a LogicException in 4.0. Turn on the "utf8" route option for pattern "%s".', $pattern), E_USER_DEPRECATED);
    }
    if (!$useUtf8 && $needsUtf8) {
        throw new \LogicException(sprintf('Cannot mix UTF-8 requirements with non-UTF-8 pattern "%s".', $pattern));
    }

    // 获取由{}包围的变量，使用\w避免匹配{和}，防止大括号嵌套，比如{foo{bar}}
    preg_match_all('#\{\w+\}#', $pattern, $matches, PREG_OFFSET_CAPTURE | PREG_SET_ORDER);
    foreach ($matches as $match) {
        // 获取变量名
        $varName = substr($match[0][0], 1, -1);
        // 获取当前变量前面的静态文本
        $precedingText = substr($pattern, $pos, $match[0][1] - $pos);
        // 更新$pos
        $pos = $match[0][1] + \strlen($match[0][0]);

        // 获取当前变量前面的静态字符
        if (!\strlen($precedingText)) {
            $precedingChar = '';
        } elseif ($useUtf8) {
            // 使用u选项匹配utf8
            preg_match('/.$/u', $precedingText, $precedingChar);
            $precedingChar = $precedingChar[0];
        } else {
            $precedingChar = substr($precedingText, -1);
        }
        // 是否为分隔符
        $isSeparator = '' !== $precedingChar && false !== strpos(static::SEPARATORS, $precedingChar);

        // A PCRE subpattern name must start with a non-digit. Also a PHP variable cannot start with a digit so the
        // variable would not be usable as a Controller action argument.
        // 变量名不能以数字开头
        if (preg_match('/^\d/', $varName)) {
            throw new \DomainException(sprintf('Variable name "%s" cannot start with a digit in route pattern "%s". Please use a different name.', $varName, $pattern));
        }
        // 不能重复使用变量名，比如path: /default/{id}/{id}
        if (\in_array($varName, $variables)) {
            throw new \LogicException(sprintf('Route pattern "%s" cannot reference variable name "%s" more than once.', $pattern, $varName));
        }

        // 变量名长度不能超过32
        if (\strlen($varName) > self::VARIABLE_MAXIMUM_LENGTH) {
            throw new \DomainException(sprintf('Variable name "%s" cannot be longer than %s characters in route pattern "%s". Please use a shorter name.', $varName, self::VARIABLE_MAXIMUM_LENGTH, $pattern));
        }

        if ($isSeparator && $precedingText !== $precedingChar) {
            // 比如path: /default/employees/{id}
            // 添加文本token，去掉末尾分隔符
            $tokens[] = array('text', substr($precedingText, 0, -\strlen($precedingChar)));
        } elseif (!$isSeparator && \strlen($precedingText) > 0) {
            // 比如path: /default/test{id}
            $tokens[] = array('text', $precedingText);
        }

        // 从requirements配置项获取变量名正则
        $regexp = $route->getRequirement($varName);
        if (null === $regexp) {
            $followingPattern = (string) substr($pattern, $pos);
            // Find the next static character after the variable that functions as a separator. By default, this separator and '/'
            // are disallowed for the variable. This default requirement makes sure that optional variables can be matched at all
            // and that the generating-matching-combination of URLs unambiguous, i.e. the params used for generating the URL are
            // the same that will be matched. Example: new Route('/{page}.{_format}', array('_format' => 'html'))
            // If {page} would also match the separating dot, {_format} would never match as {page} will eagerly consume everything.
            // Also even if {_format} was not optional the requirement prevents that {page} matches something that was originally
            // part of {_format} when generating the URL, e.g. _format = 'mobile.html'.
            $nextSeparator = self::findNextSeparator($followingPattern, $useUtf8);
            $regexp = sprintf(
                '[^%s%s]+',
                preg_quote($defaultSeparator, self::REGEX_DELIMITER),
                $defaultSeparator !== $nextSeparator && '' !== $nextSeparator ? preg_quote($nextSeparator, self::REGEX_DELIMITER) : ''
            );
            if (('' !== $nextSeparator && !preg_match('#^\{\w+\}#', $followingPattern)) || '' === $followingPattern) {
                // When we have a separator, which is disallowed for the variable, we can optimize the regex with a possessive
                // quantifier. This prevents useless backtracking of PCRE and improves performance by 20% for matching those patterns.
                // Given the above example, there is no point in backtracking into {page} (that forbids the dot) when a dot must follow
                // after it. This optimization cannot be applied when the next char is no real separator or when the next variable is
                // directly adjacent, e.g. '/{x}{y}'.
                $regexp .= '+';
            }
        } else {
            if (!preg_match('//u', $regexp)) {
                // 变量名正则未使用utf8字符集
                $useUtf8 = false;
            } elseif (!$needsUtf8 && preg_match('/[\x80-\xFF]|(?<!\\\\)\\\\(?:\\\\\\\\)*+(?-i:X|[pP][\{CLMNPSZ]|x\{[A-Fa-f0-9]{3})/', $regexp)) {
                // 变量名正则使用utf8字符集，但未设置utf8选项
                $needsUtf8 = true;
                @trigger_error(sprintf('Using UTF-8 route requirements without setting the "utf8" option is deprecated since Symfony 3.2 and will throw a LogicException in 4.0. Turn on the "utf8" route option for variable "%s" in pattern "%s".', $varName, $pattern), E_USER_DEPRECATED);
            }
            if (!$useUtf8 && $needsUtf8) {
                // 变量名正则未使用utf8字符集，但却设置了utf8选项
                throw new \LogicException(sprintf('Cannot mix UTF-8 requirement with non-UTF-8 charset for variable "%s" in pattern "%s".', $varName, $pattern));
            }
        }

        // 添加变量token
        $tokens[] = array('variable', $isSeparator ? $precedingChar : '', $regexp, $varName);
        // 添加变量名
        $variables[] = $varName;
    }

    // 处理完所有变量后，还未到模式串结尾，那剩下的字符串肯定为文本token
    if ($pos < \strlen($pattern)) {
        // 添加文本token
        $tokens[] = array('text', substr($pattern, $pos));
    }

    // 找到第一个可选token，显然token必须为变量token，并且需要设置默认值，这里只考虑了path模式串
    $firstOptional = PHP_INT_MAX;
    if (!$isHost) {
        // 考虑path: /{_locale}/default/employees/{id}，假设变量_locale和id都设置了默认值，
        // 变量token(id)是第一个可选token，也就是可以不用传参数，/en/default/employees能正确匹配，
        // 而变量token(_locale)必须传参数，尽管设置了默认值，/default/employees/1无法正确匹配，
        // 也就是说寻找第一个可选token过程中，讲究变量token的连续性
        for ($i = \count($tokens) - 1; $i >= 0; --$i) {
            // 从后向前找
            $token = $tokens[$i];
            if ('variable' === $token[0] && $route->hasDefault($token[3])) {
                $firstOptional = $i;
            } else {
                break;
            }
        }
    }

    $regexp = '';
    for ($i = 0, $nbToken = \count($tokens); $i < $nbToken; ++$i) {
        // 计算正则表达式
        $regexp .= self::computeRegexp($tokens, $i, $firstOptional);
    }
    // 使用默认分隔符拼接出最终的正则表达式，对于host模式串，会添加i选项，即忽略大小写
    $regexp = self::REGEX_DELIMITER.'^'.$regexp.'$'.self::REGEX_DELIMITER.'sD'.($isHost ? 'i' : '');

    // 支持utf8匹配需要拼接u选项
    if ($needsUtf8) {
        $regexp .= 'u';
        for ($i = 0, $nbToken = \count($tokens); $i < $nbToken; ++$i) {
            if ('variable' === $tokens[$i][0]) {
                $tokens[$i][] = true;
            }
        }
    }

    return array(
        'staticPrefix' => self::determineStaticPrefix($route, $tokens),     // 静态前缀
        'regex' => $regexp,                                                 // 正则表达式
        'tokens' => array_reverse($tokens),                                 // tokens数组
        'variables' => $variables,                                          // 变量名
    );
}
```
> **注意：**options配置项**似乎**只有compiler_class和utf8选项有效。
> 
> \vendor\symfony\symfony\src\Symfony\Component\Routing\RouteCompiler.php

RouteCompiler类的常量属性SEPARATORS定义了正则表达式可用的分隔符。

```php
const SEPARATORS = '/,;.:-_~+*=@|';
```

> **注意：**PHP正则表达式分隔符可以是任意非字母数字、非反斜线（\）、非空白字符。具体见[http://php.net/manual/zh/regexp.reference.delimiters.php](http://php.net/manual/zh/regexp.reference.delimiters.php "分隔符")。

RouteCompiler类的常量属性REGEX_DELIMITER定义了默认分隔符。

```php
const REGEX_DELIMITER = '#';
```

RouteCompiler类的常量属性VARIABLE_MAXIMUM_LENGTH定义了变量的最大长度。

```php
const VARIABLE_MAXIMUM_LENGTH = 32;
```

RouteCompiler类的findNextSeparator函数：

```php
private static function findNextSeparator($pattern, $useUtf8)
{
    if ('' == $pattern) {
        // return empty string if pattern is empty or false (false which can be returned by substr)
        return '';
    }
    // first remove all placeholders from the pattern so we can find the next real static character
    if ('' === $pattern = preg_replace('#\{\w+\}#', '', $pattern)) {
        return '';
    }
    if ($useUtf8) {
        preg_match('/^./u', $pattern, $pattern);
    }

    return false !== strpos(static::SEPARATORS, $pattern[0]) ? $pattern[0] : '';
}
```
computeRegexp函数用来转义文本token和计算可选token生成的正则表达式。RouteCompiler类的computeRegexp函数：

```php
private static function computeRegexp(array $tokens, $index, $firstOptional)
{
    $token = $tokens[$index];
    if ('text' === $token[0]) {
        // 转义文本token，除了转义特殊符号外额外需要转义#，因为#是生成最终正则表达式的默认分隔符
        return preg_quote($token[1], self::REGEX_DELIMITER);
    } else {
        // 变量token，需要考虑可选token
        if (0 === $index && 0 === $firstOptional) {
            // 当仅有一个token并且为可选token的时候，分隔符是必需的，比如path: /{test}，
            // 可选token生成的正则表达式结尾多个?
            return sprintf('%s(?P<%s>%s)?', preg_quote($token[1], self::REGEX_DELIMITER), $token[3], $token[2]);
        } else {
            // 比如path: /default/employees/{id}.{_format},defaults: {_format: html}
            $regexp = sprintf('%s(?P<%s>%s)', preg_quote($token[1], self::REGEX_DELIMITER), $token[3], $token[2]);
            if ($index >= $firstOptional) {
                $regexp = "(?:$regexp";
                $nbTokens = \count($tokens);
                if ($nbTokens - 1 == $index) {
                    // 遍历到最后一个token时，需要保证所有可选token都能正确闭合，即保证(?:与)?成对出现
                    // 前面$firstOptional为0的时候，使用了%s(?P<%s>%s)?作为正则，这里拼接)?次数要减去1
                    $regexp .= str_repeat(')?', $nbTokens - $firstOptional - (0 === $firstOptional ? 1 : 0));
                }
            }

            return $regexp;
        }
    }
}
```

> **注意：**PHP中正则表达式的特殊字符有：.\+*?[^]$(){}=!<>|:-。具体见[http://php.net/manual/zh/function.preg-quote.php](http://php.net/manual/zh/function.preg-quote.php "preg_quote")

determineStaticPrefix函数确定路由的静态前缀，后面会利用它分组。RouteCompiler类的determineStaticPrefix函数：

```php
private static function determineStaticPrefix(Route $route, array $tokens)
{
    // 比如path: /{_locale}/default/about/{id}，返回空
    if ('text' !== $tokens[0][0]) {
        return ($route->hasDefault($tokens[0][3]) || '/' === $tokens[0][1]) ? '' : $tokens[0][1];
    }

    $prefix = $tokens[0][1];

    if (isset($tokens[1][1]) && '/' !== $tokens[1][1] && false === $route->hasDefault($tokens[1][3])) {
        // 比如path: /defalut/avatar.{_format}，返回/defalut/avatar.
        $prefix .= $tokens[1][1];
    }

    // 比如path: /defalut/avatar.{_format},defaults: {_format: jpg}，返回/defalut/avatar
    return $prefix;
}
```

PhpMatcherDumper类的buildStaticPrefixCollection函数：

```php
private function buildStaticPrefixCollection(DumperCollection $collection)
{
    // 静态前缀路由集合
    $prefixCollection = new StaticPrefixCollection();

    foreach ($collection as $dumperRoute) {
        // 获取静态前缀
        $prefix = $dumperRoute->getRoute()->compile()->getStaticPrefix();
        $prefixCollection->addRoute($prefix, $dumperRoute);
    }

    // 优化分组
    $prefixCollection->optimizeGroups();

    return $prefixCollection;
}
```

> \vendor\symfony\symfony\src\Symfony\Component\Routing\Matcher\Dumper\PhpMatcherDumper.php

StaticPrefixCollection类的addRoute函数：

```php
public function addRoute($prefix, $route)
{
    $prefix = '/' === $prefix ? $prefix : rtrim($prefix, '/');
    // 防止添加不被接受的路由
    $this->guardAgainstAddingNotAcceptedRoutes($prefix);

    if ($this->prefix === $prefix) {
        // 路由集合的前缀与路由前缀严格相等的情况下，直接添加路由，并设置匹配开始位置，防止重复比较，
        // 注意这里的===
        $this->items[] = array($prefix, $route);
        $this->matchStart = \count($this->items);

        return;
    }

    // 不相等的情况下，需要遍历$items逐个比较，提取公共前缀，调整分组
    foreach ($this->items as $i => $item) {
        if ($i < $this->matchStart) {
            // 防止重复比较
            continue;
        }

        if ($item instanceof self && $item->accepts($prefix)) {
            // 递归调用，进入这里说明$item和$route有公共前缀
            $item->addRoute($prefix, $route);

            return;
        }

        // $item和$route组成子路由集合，成功返回新组，失败返回null
        $group = $this->groupWithItem($item, $prefix, $route);

        if ($group instanceof self) {
            // 替换成子路由集合
            $this->items[$i] = $group;

            return;
        }
    }

    // 遍历$items后无法组成新组，直接添加
    $this->items[] = array($prefix, $route);
}
```

> **注意：**父路由集合前缀为空，可以接受所有路由。
> 
> \vendor\symfony\symfony\src\Symfony\Component\Routing\Matcher\Dumper\StaticPrefixCollection.php

StaticPrefixCollection类的guardAgainstAddingNotAcceptedRoutes函数：

```php
private function guardAgainstAddingNotAcceptedRoutes($prefix)
{
    if (!$this->accepts($prefix)) {
        $message = sprintf('Could not add route with prefix %s to collection with prefix %s', $prefix, $this->prefix);

        throw new \LogicException($message);
    }
}
```

accepts函数规定了路由集合接受哪些路由。第一种情况，路由集合的前缀为空情况下，接受所有路由；第二种情况，路由集合的前缀是路由的前缀的前缀时，接受该路由。StaticPrefixCollection类的accepts函数：

```php
private function accepts($prefix)
{
    return '' === $this->prefix || 0 === strpos($prefix, $this->prefix);
}
```

StaticPrefixCollection的groupWithItem函数：

```php
private function groupWithItem($item, $prefix, $route)
{
    // 获取前缀
    $itemPrefix = $item instanceof self ? $item->prefix : $item[0];
    // 获取公共前缀
    $commonPrefix = $this->detectCommonPrefix($prefix, $itemPrefix);

    if (!$commonPrefix) {
        // 无公共前缀或者公共前缀比路由集合的前缀短，直接返回
        return;
    }

    // $item和$route组成子路由集合，
    $child = new self($commonPrefix);

    // 子路由集合添加$item
    if ($item instanceof self) {
        // $item为路由集合
        $child->items = array($item);
    } else {
        // $item为数组
        $child->addRoute($item[0], $item[1]);
    }

    // 子路由集合添加$route
    $child->addRoute($prefix, $route);

    return $child;
}
```

StaticPrefixCollection的detectCommonPrefix函数：

```php
private function detectCommonPrefix($prefix, $anotherPrefix)
{
    $baseLength = \strlen($this->prefix);
    $commonLength = $baseLength;
    $end = min(\strlen($prefix), \strlen($anotherPrefix));

    for ($i = $baseLength; $i <= $end; ++$i) {
        if (substr($prefix, 0, $i) !== substr($anotherPrefix, 0, $i)) {
            break;
        }

        $commonLength = $i;
    }

    // 公共前缀
    $commonPrefix = rtrim(substr($prefix, 0, $commonLength), '/');

    // 公共前缀比路由集合的前缀长时，返回
    if (\strlen($commonPrefix) > $baseLength) {
        return $commonPrefix;
    }

    return false;
}
```

PhpMatcherDumper类的compileStaticPrefixRoutes函数：

```php
private function compileStaticPrefixRoutes(StaticPrefixCollection $collection, $supportsRedirections, $ifOrElseIf = 'if')
{
    $code = '';
    // 获取前缀
    $prefix = $collection->getPrefix();

    if (!empty($prefix) && '/' !== $prefix) {
        // 前缀不为空的情况下，先匹配前缀
        $code .= sprintf("    %s (0 === strpos(\$pathinfo, %s)) {\n", $ifOrElseIf, var_export($prefix, true));
    }

    $ifOrElseIf = 'if';

    foreach ($collection->getItems() as $route) {
        if ($route instanceof StaticPrefixCollection) {
            // 递归调用
            $code .= $this->compileStaticPrefixRoutes($route, $supportsRedirections, $ifOrElseIf);
            $ifOrElseIf = 'elseif';
        } else {
            $code .= $this->compileRoute($route[1]->getRoute(), $route[1]->getName(), $supportsRedirections, $prefix)."\n";
            $ifOrElseIf = 'if';
        }
    }

    if (!empty($prefix) && '/' !== $prefix) {
        $code .= "    }\n\n";
        // apply extra indention at each line (except empty ones)
        $code = preg_replace('/^.{2,}$/m', '    $0', $code);
    }

    return $code;
}
```

PhpMatcherDumper类的compileRoute函数：

```php
private function compileRoute(Route $route, $name, $supportsRedirections, $parentPrefix = null)
{
    $code = '';
    // 已经编译过的路由
    $compiledRoute = $route->compile();
    $conditions = array();
    // 是否有尾斜杠
    $hasTrailingSlash = false;
    // path是否匹配
    $matches = false;
    // host是否匹配
    $hostMatches = false;
    // 获取路由methods配置项的值
    $methods = $route->getMethods();

    // 是否支持尾斜杠
    $supportsTrailingSlash = $supportsRedirections && (!$methods || \in_array('GET', $methods));
    // 获取path正则
    $regex = $compiledRoute->getRegex();

    if (!\count($compiledRoute->getPathVariables()) && false !== preg_match('#^(.)\^(?P<url>.*?)\$\1#'.('u' === substr($regex, -1) ? 'u' : ''), $regex, $m)) {
        // path中不包含变量
        if ($supportsTrailingSlash && '/' === substr($m['url'], -1)) {
            $conditions[] = sprintf('%s === $trimmedPathinfo', var_export(rtrim(str_replace('\\', '', $m['url']), '/'), true));
            $hasTrailingSlash = true;
        } else {
            $conditions[] = sprintf('%s === $pathinfo', var_export(str_replace('\\', '', $m['url']), true));
        }
    } else {
        // path中包含变量
        if ($compiledRoute->getStaticPrefix() && $compiledRoute->getStaticPrefix() !== $parentPrefix) {
            // 父前缀路由集合的前缀已经编译过，这里要忽略父前缀
            $conditions[] = sprintf('0 === strpos($pathinfo, %s)', var_export($compiledRoute->getStaticPrefix(), true));
        }

        if ($supportsTrailingSlash && $pos = strpos($regex, '/$')) {
            $regex = substr($regex, 0, $pos).'/?$'.substr($regex, $pos + 2);
            $hasTrailingSlash = true;
        }
        $conditions[] = sprintf('preg_match(%s, $pathinfo, $matches)', var_export($regex, true));

        $matches = true;
    }

    if ($compiledRoute->getHostVariables()) {
        // host中包含变量
        $hostMatches = true;
    }

    if ($route->getCondition()) {
        // 编译condition配置项
        $conditions[] = $this->getExpressionLanguage()->compile($route->getCondition(), array('context', 'request'));
    }

    // 拼接匹配条件
    $conditions = implode(' && ', $conditions);

    $code .= <<<EOF
    // $name
    if ($conditions) {

EOF;

    $gotoname = 'not_'.preg_replace('/[^A-Za-z0-9_]/', '', $name);

    // the offset where the return value is appended below, with indendation
    $retOffset = 12 + \strlen($code);

    // optimize parameters array
    if ($matches || $hostMatches) {
        $vars = array();
        if ($hostMatches) {
            $vars[] = '$hostMatches';
        }
        if ($matches) {
            $vars[] = '$matches';
        }
        $vars[] = "array('_route' => '$name')";

        $code .= sprintf(
            "            \$ret = \$this->mergeDefaults(array_replace(%s), %s);\n",
            implode(', ', $vars),
            str_replace("\n", '', var_export($route->getDefaults(), true))
        );
    } elseif ($route->getDefaults()) {
        $code .= sprintf("            \$ret = %s;\n", str_replace("\n", '', var_export(array_replace($route->getDefaults(), array('_route' => $name)), true)));
    } else {
        $code .= sprintf("            \$ret = array('_route' => '%s');\n", $name);
    }

    if ($hasTrailingSlash) {
        $code .= <<<EOF
        if ('/' === substr(\$pathinfo, -1)) {
            // no-op
        } elseif ('GET' !== \$canonicalMethod) {
            goto $gotoname;
        } else {
            return array_replace(\$ret, \$this->redirect(\$rawPathinfo.'/', '$name'));
        }


EOF;
    }

    if ($methods) {
        $methodVariable = \in_array('GET', $methods) ? '$canonicalMethod' : '$requestMethod';
        $methods = implode("', '", $methods);
    }

    if ($schemes = $route->getSchemes()) {
        if (!$supportsRedirections) {
            throw new \LogicException('The "schemes" requirement is only supported for URL matchers that implement RedirectableUrlMatcherInterface.');
        }
        $schemes = str_replace("\n", '', var_export(array_flip($schemes), true));
        if ($methods) {
            $code .= <<<EOF
        \$requiredSchemes = $schemes;
        \$hasRequiredScheme = isset(\$requiredSchemes[\$context->getScheme()]);
        if (!in_array($methodVariable, array('$methods'))) {
            if (\$hasRequiredScheme) {
                \$allow = array_merge(\$allow, array('$methods'));
            }
            goto $gotoname;
        }
        if (!\$hasRequiredScheme) {
            if ('GET' !== \$canonicalMethod) {
                goto $gotoname;
            }

            return array_replace(\$ret, \$this->redirect(\$rawPathinfo, '$name', key(\$requiredSchemes)));
        }


EOF;
        } else {
            $code .= <<<EOF
        \$requiredSchemes = $schemes;
        if (!isset(\$requiredSchemes[\$context->getScheme()])) {
            if ('GET' !== \$canonicalMethod) {
                goto $gotoname;
            }

            return array_replace(\$ret, \$this->redirect(\$rawPathinfo, '$name', key(\$requiredSchemes)));
        }


EOF;
        }
    } elseif ($methods) {
        $code .= <<<EOF
        if (!in_array($methodVariable, array('$methods'))) {
            \$allow = array_merge(\$allow, array('$methods'));
            goto $gotoname;
        }


EOF;
    }

    if ($hasTrailingSlash || $schemes || $methods) {
        $code .= "            return \$ret;\n";
    } else {
        $code = substr_replace($code, 'return', $retOffset, 6);
    }
    $code .= "        }\n";

    if ($hasTrailingSlash || $schemes || $methods) {
        $code .= "        $gotoname:\n";
    }

    return $code;
}
```

# 未完待续