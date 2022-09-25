## 前言

babel 和 eslint 都是我们项目中常用的工具，两者都是基于 AST 去扩展的，前者做代码的转换，后者做错误检查和修复。两者都能够做到分析和转换代码。所以两者有啥不同呢？

## babel插件

babel 的编译流程分为`parse → transform → generate`三步，可以指定插件，在遍历 AST 的时候调用 visitor，对某些节点做处理

### babel插件示例—no-function-assign-plugin

在之前的 babel 插件中，有讲过这个示例

实现思路是: 根据赋值语句去查找作用域，左边的引用是否是一个函数

- 当我们处理赋值语句`AssignmentExpression`，判断 left 的引用是否是一个函数
- 使用`path.scope.getBinding`，从作用域中查找`binding`
- 获取到`binding`是否为`FunctionDeclaration/FunctionExpression`

```jsx
module.exports = function ({ }, options) {
    return {
        pre(file) {
            file.set('errors', []);
        },
        visitor: {
            AssignmentExpression(path, state) {
                const errors = state.file.get("errors")
                const assignTarget = path.get("left").toString()
                const binding = path.scope.getBinding(assignTarget)
                if (binding) {
                    if (binding.path.isFunctionDeclaration() || binding.path.isFunctionExpression()) {
                        const tmp = Error.stackTraceLimit;
                        Error.stackTraceLimit = 0;
                        errors.push(path.buildCodeFrameError('不能重复给函数赋值', Error));
                        Error.stackTraceLimit = tmp;
                    }
                }
            }
        },
        post(file) {
            console.log(file.get('errors'));
        }
    }
}
```

### 插件特点

- 插件返回一个对象，vistor 属性中声明对节点的处理
- visitor 函数可以通过 path 来对 ast 做增删改查
- 修改之后的 ast 通过`@babel/generator`能够生成目标代码

## eslint插件

eslint 完全插件化的，每一个规则都是一个插件，在项目中可以配置多个规则。[规则列表](https://github.com/eslint/eslint/tree/main/lib/rules)

eslint 也是 AST 的应用，也需要通过 parser 将源码转为 AST。Eslint 默认使用的是 espree，也可以在配置文件中配置不同的 parser

eslint 和 babel 都会使用 parser 来做转译源码，其实它们的 parser 都是基于 estree 标准实现和扩充的

![parser](https://user-images.githubusercontent.com/38368040/169675963-e3800b75-a1fc-4515-86b3-f1672510898f.png)

acorn 的实现是基于插件的，所以 espree / babel parser 也能够通过插件来扩充

### eslint插件示例(no-function-assign)

eslint 的 rule 实现包括两部分：

- meta: 信息/文档/报错信息等
- create: 返回一个对象，其中包含一些 Eslint 遍历 AST 时，访问节点的方法

#### 创建 eslint 插件

基于 [Yeoman generator](https://yeoman.io/authoring/) 快速创建 Eslint Plugin 项目
```jsx
npm i -g yo
npm i -g generator-eslint
// 创建一个plugin
yo eslint:plugin
// 创建一个规则
yo eslint:rule
```
![rule](https://user-images.githubusercontent.com/38368040/169675998-c287b03a-a7a0-4e1d-9a35-c943fd9a29e6.png)

lib/rules 中存储的是当前插件的所有 rule，tests/lib/rules 中存储的是对应的测试文件
package.json 中 name 属性就是当前插件的名字全部以 eslint-plugin-xxx 来命名

#### no-function-assign实现

上述的 babel 插件，是找到赋值语句再去判断需要赋值的引用是否为函数声明或者函数表达式

那其实 eslint 也是可以采用相同的思路去实现的，找到找到赋值语句再去判断需要赋值的引用是为函数，那在这其中我们需要用到`context.getScope`这个 API 去获取作用域里面的信息

```jsx
module.exports = {
    create(context) {
        function checkIdentifierIsFunction(scope, leftName) {
            const allVariables = scope.variables
            for (const variable of allVariables) {
                const defs = variable.defs
                for (const def of defs) {
                    if (def.name.name === leftName && def.type === "FunctionName") return true
                    if (def.name.name === leftName) return false
                }
            }
            if (scope.upper) {
                return checkIdentifierIsFunction(scope.upper, leftName)
            }
            return false
        }
        return {
            AssignmentExpression: (node) => {
                const left = node.left
                const scope = context.getScope()
                if (checkIdentifierIsFunction(scope, left.name)) {
                    context.report({
                        node: node,
                        message: `${left} is a function`
                    })
                }
            }
        };
    },
};
```
但是上述代码随着 ES6 结构语法的出现，就不能够实现我们的效果，`[foo] = bar; function foo(){}`，对于这个代码来说，赋值语句的左边是`[foo]`，也就是一个 ArrayPattern 节点，根本不会存在 name 值，所以`checkIdentifierIsFunction`的返回将一直是 false，所以根本不会报错(上面 babel 实现的插件也会有这个问题)。

或许你会问，是不是可以再多加一层的判断对 ArrayPattern 节点做一个特殊处理，比如下面这样

```jsx
return {
    AssignmentExpression: (node) => {
        const left = node.left
        const scope = context.getScope()
        if (!left.name) {
            if (left.type === "ArrayPattern") {
                const elements = left.elements
                for (const element of elements) {
                    if (checkIdentifierIsFunction(scope, element.name)) {
                        context.report({
                            node: node,
                            message: `${element.name} is a function`
                        })
                    }
                }
                return
            }
        }
        if (checkIdentifierIsFunction(scope, left.name)) {
            context.report({
                node: node,
                message: `${left} is a function`
            })
        }
    }
};
```

虽然这样能够兼容 ArrayPattern，奈何我们 ES6 还提供了对象的结构，那总不能针对于每一个情况都去做特殊处理吧？
为了兼容所有的情况，Eslint 目前的解决思路是，找到对应的函数声明或者函数表达式，再去获得作用域中当前节点引用是否为一个赋值语句

```jsx
module.exports = {
    meta: {
        type: null, // `problem`, `suggestion`, or `layout`
        docs: {
            description: "函数不能重新赋值",
            category: "Fill me in",
            recommended: false,
            url: null, // URL to the documentation page for this rule
        },
        fixable: null, // Or `code` or `whitespace`
        schema: [], // Add a schema if the rule has options
        messages: {
            isAFunction: "'{{name}}' 是一个函数，不能够重新赋值"
        }
    },

    create(context) {
        function checkForFunction(node) {
            context.getDeclaredVariables(node).forEach(checkVariable);
        }

        function checkVariable(variable) {
            if (variable.defs[0].type === "FunctionName") {
                checkReference(variable.references);
            }
        }

        function checkReference(references) {
            // 如果是赋值语句这种
            getModifyingReferences(references).forEach(reference => {
                context.report({
                    node: reference.identifier,
                    messageId: "isAFunction",
                    data: {
                        name: reference.identifier.name
                    }
                });
            });
        }

        return {
            FunctionDeclaration: checkForFunction,
            FunctionExpression: checkForFunction
        };
    },
};
```

通过`context.getDeclaredVariables`拿到当前 node 的 variable，再去遍历每一个 variable 的引用中是否有函数

#### 项目中引入插件

由于是本地开发，所以采用`npm link`的方式，使用我们创建的 plugins
![link](https://user-images.githubusercontent.com/38368040/169676050-bb1f0d7e-7d85-463d-bb6b-53232a517ce9.png)
在我们的项目中引入该插件npm link eslint-plugin-demo，创建对应的软链接
![demo](https://user-images.githubusercontent.com/38368040/169676054-8220744a-b145-4279-ada5-1f6ffebe9459.png)

配置如下的 .eslintrc.js 文件，编写对应的代码，执行一下 eslint

```jsx
module.exports = {
    "plugins": [
        "eslint-plugin-demo"
    ],
    "rules": {  //0或off(关闭) 1或warn 2或error
        "demo/no-function-assign": 2
    }
};
```
![lint](https://user-images.githubusercontent.com/38368040/169676112-74468669-cd1a-4d7f-af31-c325d4486baa.png)
### eslint插件示例(block-one-line)

在该示例中，会对大括号的位置进行校验，并且开始自动修复模式，能够自动修复报错。

![line](https://user-images.githubusercontent.com/38368040/169676117-c09dfac5-1378-4551-84fa-6fe9da84b870.png)
我们需要处理块级语句，也就是`BlockStatement`，对于该节点来说，希望它和判断条件在一行并且中间保持一个空格。

⚠️ 我们之所以在 Eslint 中可以做格式校验，是因为我们可以通过 context 提供的`getSourceCode`API获得 AST 以及相关的 tokens
![sourceCode](https://user-images.githubusercontent.com/38368040/169676114-74a8eb9f-8424-4ac8-ae2e-0c4bdc5b6e8f.png)
![token](https://user-images.githubusercontent.com/38368040/169676121-32809119-21da-4318-a837-e8cffa0138be.png)

token 中包含了每一个单词的位置信息，sourceCode 提供了通过 node 获取对应 token

```jsx
//块级node开始的token，也就是红色部分
const firstToken = sourceCode.getFirstToken(node);
//块级node前一个token，也就是绿色部分
const beforeFirstToken = sourceCode.getTokenBefore(node);
```
当我们拿到两个对应 token 之后，将其位置(line/range)做一个判断，就能完成我们的校验

实现自动修复功能，只需要在`context.report`中定义`fix`函数，而`fix`函数也是调用 eslint 的`fixer`，我们采用`replaceTextRange`，将`[beforeFirstToken.range[1], firstToken.range[0]]`中间的内容替换为空格即可

```jsx
module.exports = {
    create(context) {
        const sourceCode = context.getSourceCode();
        return {
            BlockStatement(node) {
                //块级node开始的token
                const firstToken = sourceCode.getFirstToken(node);
                //块级node前一个token
                const beforeFirstToken = sourceCode.getTokenBefore(node);
                if (firstToken.loc.start.line !== beforeFirstToken.loc.start.line) {
                    return context.report({
                        node,
                        loc: firstToken.loc,
                        message: '大括号不需要换行',
                        fix: fixer => {
                            return fixer.replaceTextRange([beforeFirstToken.range[1], firstToken.range[0]], ' ');
                        }
                    });
                }
                if (firstToken.range[0] - 1 !== beforeFirstToken.range[1]) {
                    return context.report({
                        node,
                        loc: firstToken.loc,
                        message: '大括号前需要空格',
                        fix: fixer => {
                            return fixer.replaceTextRange([beforeFirstToken.range[1], firstToken.range[0]], ' ');
                        }
                    });
                }
            }
        };
    },
};
```

### eslint 实现原理
![原理](https://user-images.githubusercontent.com/38368040/169676159-b5250b78-c2c9-442a-b9c1-bdf7e34dec0c.png)


- 读取配置时，可以利用层叠配置。层叠配置能够让检测的文件最近的 eslintrc 文件的优先级最高

    ```jsx
    your-project
    ├── .eslintrc
    ├── lib
    │ └── source.js     // 使用根目录下的eslintrc
    └─┬ tests
      ├── .eslintrc
      └── test.js       // 使用根目录下和tests/eslintrc的组合，且tests/eslintrc中的配置优先级更高
    ```

    当项目中 package.json 中有 eslintConfig 配置时，可用整个项目，但是根目录 eslintrc 优先级更高
    关于更多读取配置信息，[点击查看](https://eslint.bootcss.com/docs/user-guide/configuring#configuration-cascading-and-hierarchy)
- 加载配置，对于 extends 来说，支持我们使用插件中的配置，所以会递归扩展配置，并且前面的配置项优先级会高于 extends的

    关于更多加载配置信息， [点击查看](https://eslint.bootcss.com/docs/user-guide/configuring#extending-configuration-files)

### eslint插件的特点

- 一个 plugin 是由 一个或者多个 rule 组成的
- 每一个 rule 都是一个对象，其中包含 meta 一些元信息，create 函数返回一个对象，定义对应的访问节点
- 对于定义的节点可以通过 context 来获取到源码中 tokens 等，来进行格式检查
- 通过在`context.report`中定义 fix 函数，使用 fixer 来对某个位置的代码进行字符的增删改，可以通过 eslint 配置开启是否自动修复功能

### 为什么Eslint能够做格式检查

Eslint 之所以能够做错误检查，它的 AST 记录了源代码所有的 token，token 中有行列号信息，而且 AST 中也保存了 range，也就是当前节点的开始结束位置。并且还提供了 SourceCode 的 api 可以根据 range 去查询 token。这是它能实现格式检查的原因。

而 Babel 其实也支持 range 和 token，但是却没有提供根据 range 查询 token 的 api，这是它不能做格式检查的原因。

## 总结

Babel 和 Eslint 原理是差不多的，先将源代码 parse，在提供对应的访问节点。但是 Eslint 是被设计来做代码错误和格式检查与修复的，而 Babel 是被设计用来做代码分析和转换的，所以也就提供了不同的 api，支持做不同的事情

![对比](https://user-images.githubusercontent.com/38368040/169676160-2030cce6-7732-41a2-8bc9-15346f32a456.png)

> 参考链接
- [零基础理解 ESLint 核心原理](https://mp.weixin.qq.com/s/wzFh_dvB13hq9OV3pC955w)
- [深入对比 eslint 插件 和 babel 插件的异同点](https://mp.weixin.qq.com/s/73TYS14n_J4nRZrj9pCt0g)
- [为什么 Eslint 可以检查和修复格式问题，而 Babel 不可以？](https://mp.weixin.qq.com/s?__biz=Mzg3OTYzMDkzMg==&mid=2247487749&idx=1&sn=4afff548495310159a8f09c4afacdd1a&chksm=cf00de3ef877572870421beb6b23dcbd437b84095e93c8b9e09736f00aba284841d4a8080cd6&scene=178&cur_album_id=2150419755759992834#rd)
- [自定义 ESLint 规则，让代码持续美丽](https://mp.weixin.qq.com/s?__biz=Mzg3NTcwMTUzNA==&mid=2247486276&idx=1&sn=1e5a8b20b03ebe94b9284cd62dc0d933&source=41#wechat_redirect)
