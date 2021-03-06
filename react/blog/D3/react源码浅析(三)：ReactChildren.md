[react.children官方API](http://react.html.cn/docs/react-api.html#react.children)
,阅读全文，可以了解react.children基本原理，掌握react.children各个API的用法，还能了解到官方API以外的补充用法。

预备知识：
react元素的$$typeof:

	$$typeof: REACT_FORWARD_REF_TYPE
	$$typeof: REACT_MEMO_TYPE
	$$typeof: REACT_CONTEXT_TYPE
	$$typeof: REACT_PROVIDER_TYPE
	$$typeof: REACT_ELEMENT_TYPE
	$$typeof: REACT_LAZY_TYPE
	$$typeof: REACT_PORTAL_TYPE

## 内部工具函数 ##
### traverseContextPool数据结构 ###

	//数据结构：context池，大小为10。当做一个栈使用
	const POOL_SIZE = 10;
	const traverseContextPool = [];
	//获取一个context
	//给栈顶的context设置相应属性值，并弹出返回。
	//如果栈中没有元素，直接返回一个对象，相应的设置了属性值
	function getPooledTraverseContext(
	  mapResult,
	  keyPrefix,
	  mapFunction,
	  mapContext,
	) {
	  if (traverseContextPool.length) {
	    const traverseContext = traverseContextPool.pop();
	    traverseContext.result = mapResult;
	    traverseContext.keyPrefix = keyPrefix;
	    traverseContext.func = mapFunction;
	    traverseContext.context = mapContext;
	    traverseContext.count = 0;
	    return traverseContext;
	  } else {
	    return {
	      result: mapResult,
	      keyPrefix: keyPrefix,
	      func: mapFunction,
	      context: mapContext,
	      count: 0,
	    };
	  }
	}
	
	//如果栈未满，push一个空context对象
	function releaseTraverseContext(traverseContext) {
	  traverseContext.result = null;
	  traverseContext.keyPrefix = null;
	  traverseContext.func = null;
	  traverseContext.context = null;
	  traverseContext.count = 0;
	  if (traverseContextPool.length < POOL_SIZE) {
	    traverseContextPool.push(traverseContext);
	  }
	}


### escape ###
将传入的key中所有的'='替换成'=0',':'替换成 '=2',并在key之前加上'$'

	function escape(key) {
	  const escapeRegex = /[=:]/g;
	  const escaperLookup = {
	    '=': '=0',
	    ':': '=2',
	  };
	  const escapedString = ('' + key).replace(escapeRegex, function(match) {
	    return escaperLookup[match];
	  });
	
	  return '$' + escapedString;
	}

### getComponentKey ###
如果component存在不为null的key，则返回escape(component.key)，否则返回index.toString(36)

	function getComponentKey(component, index) {
	  // Do some typechecking here since we call this blindly. We want to ensure
	  // that we don't blog potential future ES APIs.
	  if (
	    typeof component === 'object' &&
	    component !== null &&
	    component.key != null
	  ) {
	    // Explicit key
	    return escape(component.key);
	  }
	  // Implicit key determined by the index in the set
	  // 转换成36进制
	  return index.toString(36);
	}

### traverseAllChildrenImpl ###
**Children不能是一个对象**
代码有点长，简述其作用：输入children树，返回树中节点类型是string，number，或者节点的即$$typeof为REACT_ELEMENT_TYPE，REACT_PORTAL_TYPE的节点数量。因此React.Fragment的$$typeof也为REACT_ELEMENT_TYPE,所以React.Fragment为一个节点。如果children是Array或者其他类型的子节点，则递归调用traverseAllChildrenImpl，直到children的typeof是string，number，或者$$typeof为REACT_ELEMENT_TYPE，REACT_PORTAL_TYPE时，对该children执行callback函数，并返回1。注意：不是对所有的节点遍历。

callback传入的参数为traverseContext，children，nameSoFar 
其中nameSoFar === '' ? '.' + getComponentKey(children, 0) : nameSoFar。

	callback(
      traverseContext,
      children,
      // If it's the only child, treat the name as if it was wrapped in an array
      // so that it's consistent if the number of children grows.
      nameSoFar === '' ? SEPARATOR + getComponentKey(children, 0) : nameSoFar,
    );

可以参看blog/D3文件下的reactchildren.vsdx文件中的流程图以及react文件夹下对应的源码注释。

	function traverseAllChildrenImpl(
	  children,
	  nameSoFar,
	  callback,
	  traverseContext,
	){...}

### traverseAllChildren ###
traverseAllChildrenImpl调用封装，与其功能一样。


### forEachSingleChild ###
执行bookKeeping.func，并将bookKeeping.count的值加1。func传入的参数为bookKeeping.context,child以及bookKeeping.count。

	function forEachSingleChild(bookKeeping, child, name) {
	  const {func, context} = bookKeeping;
	  //执行bookKeeping.func，bookKeeping.count计数增加一
	  func.call(context, child, bookKeeping.count++);
	}

### forEachChildren ###
通过调用getPooledTraverseContext将传入的参数forEachFunc以及forEachContext赋值给traverseContext的func与context属性。
调用traverseAllChildren

	function forEachChildren(children, forEachFunc, forEachContext) {
	  if (children == null) {
	    return children;
	  }
	  const traverseContext = getPooledTraverseContext(
	    null,
	    null,
	    forEachFunc,
	    forEachContext,
	  );
	  traverseAllChildren(children, forEachSingleChild, traverseContext);
	  releaseTraverseContext(traverseContext);
	}

### escapeUserProvidedKey ###
匹配一个或者多个 "/",并用'$&/'替换

	const userProvidedKeyEscapeRegex = /\/+/g;
	function escapeUserProvidedKey(text) {
	  return ('' + text).replace(userProvidedKeyEscapeRegex, '$&/');
	}

### mapIntoWithKeyPrefixInternal ###
调用escapeUserProvidedKey对传入的prefix进行处理得到escapedPrefix，载
通过调用getPooledTraverseContext将传入的参数array、escapedPrefix、func以及context赋值给traverseContext的result、keyPrefix、func与context属性。
调用traverseAllChildren。最后清除traverseContext上的属性，并入栈。

	function mapIntoWithKeyPrefixInternal(children, array, prefix, func, context) {
	  let escapedPrefix = '';
	  if (prefix != null) {
	    escapedPrefix = escapeUserProvidedKey(prefix) + '/';
	  }
	  const traverseContext = getPooledTraverseContext(
	    array,
	    escapedPrefix,
	    func,
	    context,
	  );
	  traverseAllChildren(children, mapSingleChildIntoContext, traverseContext);
	  releaseTraverseContext(traverseContext);
	}

### mapSingleChildIntoContext ###
对children执行func（func为传入的React.Children.map中的func）,
如果返回了一个数组，则对这个数组调用mapIntoWithKeyPrefixInternal目的是添加特定的key
克隆以child节点为根节点的树中的所有child，替换掉每个新child元素的key，push到bookKeeping中的result

	function mapSingleChildIntoContext(bookKeeping, child, childKey) {
	  const {result, keyPrefix, func, context} = bookKeeping;
	
	  let mappedChild = func.call(context, child, bookKeeping.count++);
	  if (Array.isArray(mappedChild)) {
	      //如果child中包含多个child，则返回的mappedChild是一个数组，则递归调用mapIntoWithKeyPrefixInternal
	    mapIntoWithKeyPrefixInternal(mappedChild, result, childKey, c => c);
	  } else if (mappedChild != null) {
	      //如果child中只有一个child，并且是合法的react元素，
	      // 则将mappedChild的key属性值替换掉
	    //  最后将新的react元素push到bookKeeping.result
	    if (isValidElement(mappedChild)) {
	      mappedChild = cloneAndReplaceKey(
	        mappedChild,
	        // Keep both the (mapped) and old keys if they differ, just as
	        // traverseAllChildren used to do for objects as children
	        keyPrefix +
	          (mappedChild.key && (!child || child.key !== mappedChild.key)
	            ? escapeUserProvidedKey(mappedChild.key) + '/'
	            : '') +
	          childKey,
	      );
	    }
	    result.push(mappedChild);
	  }
	}

## 对外接口 ##

### mapChildren ###

	React.Children.map(children, func, context)
参数描述：

	children：
		可以是一个对象，但是必须具备属性$$typeof为Symbol.for('react.portal')或者Symbol.for('react.element')，可以称其为类reactChild对象，否则报错。
		children为null或者undefined就返回null或者undefined，children中的Fragment为一个子组件。
	func：
		对符合规定的children执行的函数，func会被传入两个参数，符合规定的children以及到当前children的数量。所有执行func返回的children都会添加到一个数组中，没有嵌套。
	context:
		一般都为null，如果传入context则func运行中的this为context，看例2

返回值：
	返回一个平面数组，看例1

例子相关代码，见runLogic文件夹的index.js：

	//
	<App>
        {/*测试*/}
        <Header/>
        <Content/>
        string 1
        <React.Fragment>
            Some text.
            <h2>A heading</h2>
        </React.Fragment>
        <Footer>覆盖</Footer>
        string 2
    </App>

	//
	class App extends React.Component{
	    render(){
			例1代码
			例2代码
	        return (
	            <div>
					...
	            </div>
	        )
	    }
	}																

例1：测试children是一个嵌套结构，返回的数组是否是一个平面数组：

	let reactChildLike = {$$typeof:Symbol.for('react.element')}
	let complexChildren = [reactChildLike,[reactChildLike,this.props.children]]
	console.log(React.Children.map(complexChildren,(children)=>[children,children,children]))

结果：

	this.props.children为：
	(6) [{…}, {…}, "string 1", {…}, {…}, "string 2"]

	complexChildren中符合规定的child为1+1+6=8，所以输出的result为3*8=24个元素的平面数组

	(24) [{…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, "string 1", "string 1", "string 1", {…}, {…}, {…}, {…}, {…}, {…}, "string 2", "string 2", "string 2"]

例2：测试context的作用

		let reactChildLike = {$$typeof:Symbol.for('react.element')}
        let func = function (child) {
            console.log(this)
            this.a=1000;
            return child
        }
        let contextTest = {a:1}
        console.log("React.Children.map test",React.Children.map(reactChildLike,func,contextTest))
        console.log("contextTest.a",contextTest.a)

结果：

		React.Children.map test 
			[{…}]
			0: {$$typeof: Symbol(react.element), type: undefined, key: ".0", ref: undefined, props: undefined, …}length: 1__proto__: Array(0)
		contextTest.a 1000
func给this.a赋值为1000，在传入context的时候，外部的context.a变成了1000。



#### 源码： ####


	function mapChildren(children, func, context) {
	//children为null或者undefined就返回null或者undefined
	  if (children == null) {
	    return children;
	  }
	  const result = [];
	  mapIntoWithKeyPrefixInternal(children, result, null, func, context);
	  return result;
	}

#### 运行逻辑： ####

----------
#### 1. 初始调用 ####
**mapChildren函数中**：mapChildren传入（children，func，context），调用mapIntoWithKeyPrefixInternal(children, [], null, func, context)


----------
#### 2. 构建traverseContext ####
**mapIntoWithKeyPrefixInternal函数中**：prefix为传入的第三个参数null，此时traverseContextPool=[]，直接返回

	traverseContext={
      result: [],
      keyPrefix: null,
      func: func,
      context: context,
      count: 0,
    }
调用traverseAllChildren(children, mapSingleChildIntoContext, traverseContext)

----------

#### 3. children构建key ####
**traverseAllChildren函数中**：调用traverseAllChildrenImpl(children, '', mapSingleChildIntoContext, traverseContext)

**traverseAllChildrenImpl函数中**：
如果children是符合规定的则调用mapSingleChildIntoContext(traverseContext,children,'.’+children的key)。


----------
#### 4. 对children执行func，并添加到结果数组 ####
**mapSingleChildIntoContext函数中**：traverseContext.func为mapChildren函数接收到的func，调用traverseContext.func.call(context, children, traverseContext.count++)。如果传入mapChildren的func对该children进行了改造。并返回了一数组的新children，则调用mapIntoWithKeyPrefixInternal为符合规定的成员添加特定的Key之后，添加到result数组中。

----------



### toArray ###
利用mapChildren也能实现toArray的功能，只需要func为child => child即可

	function toArray(children) {
	  const result = [];
	  mapIntoWithKeyPrefixInternal(children, result, null, child => child);
	  return result;
	}

### onlyChild ###
判断children是否是单个React element child

	function onlyChild(children) {
	  invariant(
	    isValidElement(children),
	    'React.Children.only expected to receive a single React element child.',
	  );
	  return children;
	}

### countChildren ###
计算children个数

	function countChildren(children) {
	  return traverseAllChildren(children, () => null, null);
	}
