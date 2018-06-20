## node模块加载

node/lib/internal/bootstarp/loaders.js，这个文件用于创建内部模块以及其使用的绑定加载器，而用户模块的加载则可以使用lib/internal/modules/cjs/loader.js或者lib/internal/modules/esm/*里的方法。

该文件在bootstrap/node.js被触发前，会被node.cc编译后运行，所以该加载器会在我们启动node.js之前就已经被启动了。它创建了以下这些对象：c++绑定加载器，js内部模块加载器等。

#### C++绑定加载器：

- process.binding()：我们可以利用的c++绑定加载器，这是全局对象process的一个方法，我们可以利用他获取到c++里的内容，这相当于是js和c++的一个通道。利用c++中的`NODE_BUILTIN_MODULE_CONTEXT_AWARE()` 创建，这些绑定的稳定性无法保证，但是也要时刻注意它们引起的兼容性问题。
- process._linkedBinding() ：让我们在应用中可以加入额外的c++内容绑定，这些c++内容绑定可以利用c++中的`NODE_MODULE_CONTEXT_AWARE_CPP()`创建。
- internalBinding()：c++私有的内部绑定加载器，我们不能使用的。因为它们只用在`NativeModule.require()`中被调用。这些c++绑定可以利用c++中的`NODE_MODULE_CONTEXT_AWARE_INTERNAL()`创建。

#### js内部模块加载器：

NativeModule：最小的模块系统，它用于加载在lib/\*.js and deps/\*\*/\*.js中的核心js模块，js2c.py生成的node_javascript.cc把所有核心模块都编译成二进制文件，这样它们就能更快的被编译，没有I/O消耗。这个类使得lib/internal/\*, deps/internal/\*模块以及internalBinding()方法能够默认用于核心模块，让这些核心模块可以通过require('internal/bootstrap/loaders')等语法引入，即使该文件没有写入CommonJS风格。

#### 其他对象:

- process.moduleLoadList：数组，用于按顺序记录所绑定的和已经加载进process的模块。

```javascript

(function bootstrapInternalLoaders(process, getBinding, getLinkedBinding,
                                   getInternalBinding) {
  //引入Reflect对象中的部分方法，并设置别名  
  const {
    apply: ReflectApply,//改变this指向
    deleteProperty: ReflectDeleteProperty,//删除属性
    get: ReflectGet,//取值
    getOwnPropertyDescriptor: ReflectGetOwnPropertyDescriptor,
    has: ReflectHas,//判断是否有相应属性
    set: ReflectSet,//赋值
  } = Reflect;
  
  //引入Object对象中的部分方法，并设置别名  
  const {
    prototype: {
      hasOwnProperty: ObjectHasOwnProperty,//判断是否有属性，不检查原型链
    },
    create: ObjectCreate,//创建对象
    defineProperty: ObjectDefineProperty,//设置属性
    keys: ObjectKeys,//将属性转为数组
  } = Object;

  //数组，用于按顺序记录所绑定的和已经加载进process的模块
  const moduleLoadList = [];
    
  //将moduleLoadList作为process对象的属性
  ObjectDefineProperty(process, 'moduleLoadList', {
    value: moduleLoadList,
    configurable: true,
    enumerable: true,//允许遍历
    writable: false//不允许重写
  });

  
  //块级作用域，定义process.binding()和process._linkedBinding()
  {
    //创建对象，用来记录模块
    const bindingObj = ObjectCreate(null);
	//process.binding方法，传入一个module参数
    //绑定并加载模块，按顺序保存起来
    process.binding = function binding(module) {
      //将module转为字符串
      module = String(module);
      //记录模块
      let mod = bindingObj[module];
        
      if (typeof mod !== 'object') {  
        //将module的内容读取到内存中，保存在bindingObj
        mod = bindingObj[module] = getBinding(module);
        //按顺序记录在moduleLoadList数组
        moduleLoadList.push(`Binding ${module}`);
      }
      
      return mod;
    };

    //加入额外的c++绑定
    process._linkedBinding = function _linkedBinding(module) {
      module = String(module);
      let mod = bindingObj[module];
      if (typeof mod !== 'object')
        mod = bindingObj[module] = getLinkedBinding(module);
      return mod;
    };
  }

  //定义internalBinding()
  let internalBinding;
  { 
    const bindingObj = ObjectCreate(null);
    //process.internalBinding方法，传入一个module参数
    //绑定并加载内部模块，按顺序保存起来
    internalBinding = function internalBinding(module) {
      let mod = bindingObj[module];
      if (typeof mod !== 'object') {
        mod = bindingObj[module] = getInternalBinding(module);
        moduleLoadList.push(`Internal Binding ${module}`);
      }
      return mod;
    };
  }

  const { ContextifyScript } = process.binding('contextify');

  //NativeModule 本地模块类
  function NativeModule(id) {
    this.filename = `${id}.js`;//文件名
    this.id = id;//文件路径（标识id）
    this.exports = {};
    this.reflect = undefined;
    this.exportKeys = undefined;
    this.loaded = false;
    this.loading = false;
  }
  //绑定内部模块
  NativeModule._source = getBinding('natives');
  //创建缓存对象_cache
  NativeModule._cache = {};
    
  //绑定配置
  const config = getBinding('config');

  //导出internalBinding和NativeModule
  const loaderExports = { internalBinding, NativeModule };
  //记录改文件路径
  const loaderId = 'internal/bootstrap/loaders';
  //require方法
  NativeModule.require = function(id) {
    //引入本文件时直接返回
    if (id === loaderId) {
      return loaderExports;
    }
	//获取缓存
    const cached = NativeModule.getCached(id);
    //判断是否为缓存，还得是已经加载完的模块
    //如果是，就直接返回缓存的内容
    if (cached && (cached.loaded || cached.loading)) {
      return cached.exports;
    }
	//判断该模块是否内建模块
    if (!NativeModule.exists(id)) {
      const err = new Error(`No such built-in module: ${id}`);
      err.code = 'ERR_UNKNOWN_BUILTIN_MODULE';
      err.name = 'Error [ERR_UNKNOWN_BUILTIN_MODULE]';
      throw err;
    }
	//按顺序保存加载的内部模块，
    moduleLoadList.push(`NativeModule ${id}`);
	//创建NativeModule实例
    const nativeModule = new NativeModule(id);
	
    //缓存
    nativeModule.cache();
    //编译
    nativeModule.compile();
	//返回要导出的模块
    return nativeModule.exports;
  };
  
  NativeModule.requireForDeps = function(id) {
    if (!NativeModule.exists(id) ||
        id.startsWith('node-inspect/') ||
        id.startsWith('v8/')) {
      id = `internal/deps/${id}`;
    }
    return NativeModule.require(id);
  };

  //从_cache对象中获取缓存
  NativeModule.getCached = function(id) {
    return NativeModule._cache[id];
  };
    
  //判断传入的参数所对应的模块是否内部模块
  NativeModule.exists = function(id) {
    return NativeModule._source.hasOwnProperty(id);
  };

  
  if (config.exposeInternals) {
    NativeModule.nonInternalExists = function(id) {
      // 即使配置了--expose-internals，也不内部模块暴露给开发者
      if (id === loaderId) {
        return false;
      }
      return NativeModule.exists(id);
    };
    NativeModule.isInternal = function(id) {
	  // 即使配置了--expose-internals，也不暴露内部模块给开发者
      return id === loaderId;
    };
  } else {
    NativeModule.nonInternalExists = function(id) {
      return NativeModule.exists(id) && !NativeModule.isInternal(id);
    };

    NativeModule.isInternal = function(id) {
      return id.startsWith('internal/') ||
          (id === 'worker_threads' &&
           !process.binding('config').experimentalWorker);
    };
  }
  
  //从_source中获取内部模块
  NativeModule.getSource = function(id) {
    return NativeModule._source[id];
  };
    
  //给读入的模块代码添加一层包裹，进行隔离封装
  NativeModule.wrap = function(script) {
    return NativeModule.wrapper[0] + script + NativeModule.wrapper[1];
  };

  NativeModule.wrapper = [
     //四个参数，模块导出的内容，require方法，模块，process全局对象
    '(function (exports, require, module, process) {',
    '\n});'
  ];
  
  //获取target的属性
  //如果有，则返回该属性的值；否则返回undefined
  const getOwn = (target, property, receiver) => {
    return ReflectApply(ObjectHasOwnProperty, target, [property]) ?
      ReflectGet(target, property, receiver) :
      undefined;
  };
    
  //将compile捆绑在NativeModule的原型链上
  NativeModule.prototype.compile = function() {
    //获取引入的内部模块
    let source = NativeModule.getSource(this.id);
    //对模块外部进行一层包裹，封装隔离
    source = NativeModule.wrap(source);
	//模块正在加载
    this.loading = true;

    try {
      //执行该模块函数
      const script = new ContextifyScript(source, this.filename);
      const fn = script.runInThisContext(-1, true, false);
      //导入所需的依赖
      const requireFn = this.id.startsWith('internal/deps/') ?
        NativeModule.requireForDeps :
        NativeModule.require;
      //将参数传入，分别是：模块希望导出的内容，require方法，模块，process全局对象  
      fn(this.exports, requireFn, this, process);
        
	  //判断传入的是否是内部模块
      if (config.experimentalModules && !NativeModule.isInternal(this.id)) {
        //获取需要导出的内容
        this.exportKeys = ObjectKeys(this.exports);
          
	    //定义update函数，对this.reflect.exports中需要导出的内容进行更新，
        //如果已经有该属性，则更新，若没有则进行添加
        const update = (property, value) => {
          if (this.reflect !== undefined &&
              ReflectApply(ObjectHasOwnProperty,
                           this.reflect.exports, [property]))
            this.reflect.exports[property].set(value);
        };
		
        //句柄，用于代理拦截
        const handler = {
          //原型链指向null对象
          __proto__: null,
          //代理拦截defineProperty方法
          defineProperty: (target, prop, descriptor) => {
            //这里用Object.defineProperty而不用Reflect.defineProperty
            //目的是为了能在错误时报错，而后者会返回undifined
            //为target添加新得属性
            ObjectDefineProperty(target, prop, descriptor);
            //判断descriptor中是否有get函数，handler中是否无get字段
            //如果有
            if (typeof descriptor.get === 'function' &&
                !ReflectHas(handler, 'get')) {
              //为handler添加get函数
              handler.get = (target, prop, receiver) => {
                //获取target的prop属性的值
                const value = ReflectGet(target, prop, receiver);
                //对target的新属性的值进行更新
                if (ReflectApply(ObjectHasOwnProperty, target, [prop]))
                  update(prop, value);
                return value;
              };
            }
            //否则
            //直接更新更新
            update(prop, getOwn(target, prop));
            return true;
          },
          deleteProperty: (target, prop) => {
            //删除属性
            if (ReflectDeleteProperty(target, prop)) {
              //将target的该属性的值设置为undefined进行删除
              update(prop, undefined);
              //成功删除
              return true;
            }
            return false;
          },
          set: (target, prop, value, receiver) => {
            //对target的属性设置新值
            const descriptor = ReflectGetOwnPropertyDescriptor(target, prop);
            if (ReflectSet(target, prop, value, receiver)) {
              if (descriptor && typeof descriptor.set === 'function') {
                //对导出对象的每一个属性的值都进行重新设置  
                for (const key of this.exportKeys) {
                  update(key, getOwn(target, key, receiver));
                }
              } else {
                //对属性的值都进行重新设置  
                update(prop, getOwn(target, prop, receiver));
              }
              return true;
            }
            return false;
          }
        };
		
        //代理拦截处理后，将需要导出的内容挂载在this。exports中
        this.exports = new Proxy(this.exports, handler);
      }
      //加载编译结束
      this.loaded = true;
    } finally {
      //加载编译结束
      this.loading = false;
    }
  };
    
  //将cache方法绑定在原型链中
  NativeModule.prototype.cache = function() {
    //将引入的模块的上下文对象缓存在_cache
    NativeModule._cache[this.id] = this;
  };

  //返回的值将会在bootstrap/node.js的bootstrapNodeJSCore函数中被引入
  return loaderExports;
});
```

