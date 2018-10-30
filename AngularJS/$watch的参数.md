# $watch的参数
> 之前只知道$watch方法填写字符串即可监听$scope的指定属性，deepWatch设为true即可深度监听，昨天看[ng-echarts](https://github.com/liekkas/ng-echarts)源码发现，它的watch是这样写的
```js
scope.$watch(
    function () { return scope.option; },
    function (value) {if (value) {refreshChart();}},
    true
);
```
## 函数
```js
$watch: function(watchExp, listener, objectEquality, prettyPrintExpression) {
        //$parse是ng内置的一个编译器，用于将一个angular的表达式转化为一个函数，$parse的源码分析，这位哥写得非常好
        //（目测是他首发，可是2关注16粉丝惨案……）
        //http://www.cnblogs.com/web2-developer/p/angular-10.html
        var get = $parse(watchExp);
        var fn = isFunction(listener) ? listener : noop;

        if (get.$$watchDelegate) {
        //若watchExp已有监听委托，则直接返回该委托并增加对应参数
          return get.$$watchDelegate(this, fn, objectEquality, get, watchExp);
        }
        var scope = this,
            array = scope.$$watchers,
            watcher = {
              fn: fn,
              last: initWatchVal,
              get: get,
              exp: prettyPrintExpression || watchExp,
              eq: !!objectEquality
            };

        lastDirtyWatch = null;

        if (!array) {
          array = scope.$$watchers = [];
          array.$$digestWatchIndex = -1;
        }
        // we use unshift since we use a while loop in $digest for speed.
        // the while loop reads in reverse order.
        array.unshift(watcher);
        array.$$digestWatchIndex++;
        incrementWatchersCount(this, 1);

        //返回一个函数，用于删除监听
        return function deregisterWatch() {
          var index = arrayRemove(array, watcher);
          if (index >= 0) {
            incrementWatchersCount(scope, -1);
            if (index < array.$$digestWatchIndex) {
              array.$$digestWatchIndex--;
            }
          }
          lastDirtyWatch = null;
        };
      },
```
显然这里利用了var get = $parse(watchExp)，将第一个参数转为一个ng的function，要了解$parse,需要看$parse的源码，从它的$get可以看到
```js
this.$get = ['$filter', function($filter) {
    var noUnsafeEval = csp().noUnsafeEval;
    var $parseOptions = {
          csp: noUnsafeEval,
          literals: copy(literals),
          isIdentifierStart: isFunction(identStart) && identStart,
          isIdentifierContinue: isFunction(identContinue) && identContinue
        };
    $parse.$$getAst = $$getAst;
    return $parse;

    function $parse(exp, interceptorFn) {
      var parsedExpression, cacheKey;

      switch (typeof exp) {
        case 'string':
          exp = exp.trim();
          cacheKey = exp;

          parsedExpression = cache[cacheKey];

          if (!parsedExpression) {
            var lexer = new Lexer($parseOptions);
            var parser = new Parser(lexer, $filter, $parseOptions);
            parsedExpression = parser.parse(exp);

            cache[cacheKey] = addWatchDelegate(parsedExpression);
          }
          return addInterceptor(parsedExpression, interceptorFn);

        case 'function':
          return addInterceptor(exp, interceptorFn);

        default:
          return addInterceptor(noop, interceptorFn);
      }
    }
 ```
 如果是function，返回了addInterceptor方法，再看……
 ```js
 function addInterceptor(parsedExpression, interceptorFn) {
      if (!interceptorFn) return parsedExpression;

      // Extract any existing interceptors out of the parsedExpression
      // to ensure the original parsedExpression is always the $$intercepted
      if (parsedExpression.$$interceptor) {
        interceptorFn = chainInterceptors(parsedExpression.$$interceptor, interceptorFn);
        parsedExpression = parsedExpression.$$intercepted;
      }

      var useInputs = false;

      var fn = function interceptedExpression(scope, locals, assign, inputs) {
        var value = useInputs && inputs ? inputs[0] : parsedExpression(scope, locals, assign, inputs);
        return interceptorFn(value);
      };

      // Maintain references to the interceptor/intercepted
      fn.$$intercepted = parsedExpression;
      fn.$$interceptor = interceptorFn;

      // Propogate the literal/oneTime/constant attributes
      fn.literal = parsedExpression.literal;
      fn.oneTime = parsedExpression.oneTime;
      fn.constant = parsedExpression.constant;

      // Treat the interceptor like filters.
      // If it is not $stateful then only watch its inputs.
      // If the expression itself has no inputs then use the full expression as an input.
      if (!interceptorFn.$stateful) {
        useInputs = !parsedExpression.inputs;
        fn.inputs = parsedExpression.inputs ? parsedExpression.inputs : [parsedExpression];

        if (!interceptorFn.$$pure) {
          fn.inputs = fn.inputs.map(function(e) {
              // Remove the isPure flag of inputs when it is not absolute because they are now wrapped in a
              // non-pure interceptor function.
              if (e.isPure === PURITY_RELATIVE) {
                return function depurifier(s) { return e(s); };
              }
              return e;
            });
        }
      }

      return addWatchDelegate(fn);
    }
    
    function addWatchDelegate(parsedExpression) {
      if (parsedExpression.constant) {
        parsedExpression.$$watchDelegate = constantWatchDelegate;
      } else if (parsedExpression.oneTime) {
        parsedExpression.$$watchDelegate = oneTimeWatchDelegate;
      } else if (parsedExpression.inputs) {
        parsedExpression.$$watchDelegate = inputsWatchDelegate;
      }

      return parsedExpression;
    }
 ```
这里，interceptorFn是undefined，就直接返回parsedExpression了，回到$watch的array，那个保存着watcher的array，watcher的get最终保存的还是当时传入的function，但是这个function为什么可以作为一个对象被监听到变动呢？在$watch的注释中，有这样的说明
```js
   * @description
   * Registers a `listener` callback to be executed whenever the `watchExpression` changes.
   *
   * - The `watchExpression` is called on every call to {@link ng.$rootScope.Scope#$digest
   *   $digest()} and should return the value that will be watched. (`watchExpression` should not change
   *   its value when executed multiple times with the same input because it may be executed multiple
   *   times by {@link ng.$rootScope.Scope#$digest $digest()}. That is, `watchExpression` should be
   *   [idempotent](http://en.wikipedia.org/wiki/Idempotence).)
```
> 该参数是一个带有Angular表达式或者函数的字符串，它会返回被监控的数据模型的当前值。这个表达式将会被执行很多次，所以你要保证它不会产生其他副作用。
