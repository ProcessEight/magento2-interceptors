### ABOUT

This plugin changes the way Magento 2 generates Interceptor classes (a mechanism that allows plugins to work together).

Instead of generating boilerplate code it compiles the Interceptor using information from source code. 
This makes plugins slightly faster and as Magento uses a lot of plugins even at its core, it lowers request time by ~10%.
This is important in places where there is a lot of non-cached PHP logic going (for example admin panel is noticeably faster).

The dafault method uses code that is called on nearly eatch method to see if there are any plugins connected, in generated code this is not required and call-stack is reduced.
Having plugins called directly also makes code easier to debug and bugs easier to find.

Interceptors generated by this plugin are 100% compatible with ones generated by Magento by default so there is no need to change anything in your plugins.

### INSTALLATION

###### 1. Install module

`composer require creatuity/magento2-interceptors`

###### 2. Configure your instance to use CompiledInterceptor

to use in developer mode in `app/etc/di.xml` replace:
```
<item name="interceptor" xsi:type="string">\Magento\Framework\Interception\Code\Generator\Interceptor</item>
```
with:
```
<item name="interceptor" xsi:type="string">\Creatuity\Interception\Generator\CompiledInterceptor</item>
```

to use compiled interceptors in production mode please add following preference to your di.xml
```
<preference for="Magento\Framework\Interception\Code\Generator\Interceptor" type="Creatuity\Interception\Generator\CompiledInterceptor" />
```

###### 3. Upgrade and clear cache

upgrade setup:

`bin/magento setup:upgrade`

clear generated files:

`rm -rf generated/* && rm -rf var/cache/*`

clear magentos cache:

`bin/magento cache:clean`

### UNINSTALLATION

Replace back the lines in `app/etc/di.xml`, remove module and clear cache and generated files.

### TECHNICAL DETAILS 

Instead of interceptors that read plugins config at runtime like this:

```
public function methodX($arg) {
    $pluginInfo = $this->pluginList->getNext($this->subjectType, 'methodX');
    if (!$pluginInfo) {
        return parent::methodX($arg);
    } else {
        return $this->___callPlugins('methodX', func_get_args(), $pluginInfo);
    }
}
```

This generator generates static interceptors like:


```
public function methodX($arg) {
    switch(getCurrentScope()){
        case 'frontend':
            $this->_get_example_plugin()->beforeMethodX($this, $arg);
            $this->_get_another_plugin()->beforeMethodX($this, $arg);
            $result = $this->_get_around_plugin()->aroundMethodX($this, function($arg){
                return parent::methodX($arg);
            });
            return $this->_get_after_plugin()->afterMethodX($this, $result);
        case 'adminhtml':
            // ...
        default:
            return parent::methodX($arg);
    }
}
```


#### PROS

* easier debugging. 
  * If you ever stumbled upon `___callPlugins` when debugging you should know how painful it is to debug issues inside plugins.
  * generated code is decorated with PHPDoc for easier debugging in IDE

* fastest response time (5%-15% faster in developer and production mode)
  * no redundant calls to `___callPlugins` in call stack.
  * methods with no plugins are not overriden in parent at all.
  
* implemented as a module and can be easily reverted to default `Generator\Interceptor`

#### CONS

* each time after making change in etc plugins config, `generated/code/*` needs to be purged
* tiny longer code generation step when run with no cache
* as this does not load plugins at runtime might not work in an edge case of plugging into core Magento classes like PluginsList etc.
