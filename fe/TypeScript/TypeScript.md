# TypeScript



## 入门

```shell
# 安装
npm install -g typescript
# 使用ts-node运行ts文件
npm install -g ts-node
# 项目中 or 全局加-g
npm install @types/node
```



## 基本类型和扩展类型

### 基本类型

> 5个基本类型

```typescript
// number | Number
let my_num: number = 100
let my_big_num: Number = 200
console.log(my_num, my_big_num)

// string | String
let my_str: string = 'aaa'
let my_big_str: String = 'AAA'
console.log(my_str, my_big_str)

// boolean | Boolean
let my_boo: boolean = true
let my_big_boo: Boolean = false
console.log(my_boo, my_big_boo)

// undefined
let my_undefined: undefined = undefined
console.log(my_undefined, typeof my_undefined)// undefined undefined

// null
let my_nul: null = null
/*
 typeof null === object，不同的对象在底层都表示为二进制，在 JavaScript 中二进制前三位都为 0 的话会被判断为 object 类型，
 null 的二进制表示是全 0，自然前三位也是 0，所以执行 typeof 时会返回 “object”
*/
console.log(my_nul, typeof my_nul) // null object
// Number(null) === 0
console.log(Number(null))// 0
```



**字面量**

> 定义什么值就只能赋什么值

```typescript
let dog: 'dog'

// dog = 'cat' // error: '"cat"' is not assignable to type '"dog"'.
dog = 'dog'

// 同时赋予多个不同类型的可选值，用"|"分开
let my_var: 'cat'|'dog'|false|100
my_var = 'cat'
my_var = 'dog'
my_var = false
my_var = 100
```



**联合类型**

> 对于一个变量的类型可能是几种类型的时候我们可以使用 any ，但是 any 的范围有点大，不到万不得已不使用；如果知道是其中的哪几种类型的话，我们就可以使用 联合类型 用 | 分隔

```typescript
// 联合类型 union types
let union_var: string | number | boolean

// 注意：在没有赋值之前，只能访问共同的方法、属性，比如下面的例子，number 没有length 属性
// union_var.valueOf()

union_var = false
console.log(union_var)

union_var = 'union_var_str'
console.log(union_var, union_var.length)

union_var = 100
console.log(union_var)
```



**类型守卫**

```typescript
// 类型守卫 type guard
// typeof、instanceof、 in
// 遇到联合类型的时候，使用 类型守卫可以 缩小范围
function getLenGuard(param: number | string) : number {
    if (typeof param === 'string') {
        return param.length
    }
    return param.toString().length
}

console.log(getLen(122222))
```



**类型断言**

```typescript
// 类型断言
// 在上面联合类型的变量传入的时候，我们声明了这个类型为 number | string 它不能不能调用 length 方法
// 机器没法判断这个类型，可以人为指定类型 string 这里我们就可以用到 类型断言
function getLen(param: number | string) : number {
    // 1、用 as 来进行断言
    // const str = param as string
    // 2、用 范型 来进行断言
    const str = <string> param
    if (str.length) {
        return str.length
    }
    return str.toString().length
}

console.log(getLen(123))
```



### 特殊类型

```typescript
// any
let my_any: any = "aaa" + 100
console.log(my_any, typeof my_any)

// 方法一：带有any参数的方法
function any_func(arg: any): any {
    console.log(arg.length);
    return arg;
}

// 方法二：Array泛型方法
function array_func<T>(arg: Array<T>): Array<T> {
    console.log(arg.length);
    return arg;
}

/*
方法一，打印了arg参数的length属性。因为any可以代替任意类型，所以该方法在传入参数不是数组或者带有length属性对象时，会抛出异常。
方法二，定义了参数类型是Array的泛型类型，肯定会有length属性，所以不会抛出异常。
*/

// unknown
let my_unknown: unknown = false + "aaa"
console.log(my_unknown, typeof my_unknown)

// never

// void
function my_fun(): void {}
console.log(my_fun, typeof my_fun)

const my_fun1 = () => {}
console.log(my_fun1, typeof my_fun1)
```



**数组**

```typescript
// 数组 Array
let my_arr = [1, 2, 3, 4]
// typeof my_arr === object
// my_arr.constructor === Array
// my_arr instanceof  Array === true
console.log("=== my_arr ===")
console.log(my_arr, typeof my_arr, my_arr.constructor === Array, my_arr instanceof  Array)

// array 设置数组的类型 比如这个例子 true 这个就会报错，不属于number，数组的元素必须是规定好的类型 其他类型同理
let my_num_arr: number[] = [1,2,3,4,5,6]
console.log('=== my_num_arr ===')
console.log(my_num_arr, typeof my_num_arr, my_num_arr.constructor === Array)

let my_mix_arr = [1, , false, null, undefined, Object]
console.log('=== my_mix_arr ===')
console.log(my_mix_arr, typeof my_mix_arr)

// instanceof 判断是Array还是Object或者null
const isObjectArray = (params: any[]): any => {
    if (params === null) {
        return null
    } else {
        if (params instanceof Array) {
            return Object
        } else {
            return null
        }
    }
}
console.log(isObjectArray(my_num_arr));
```



**元组**

```typescript
// 元组数据类型需要跟给定的变量类型一致
let my_tuple: [string, boolean, number] = ['aaa', false, 100]
console.log(my_tuple)
```



**对象**

```typescript
// 直接 let a: object; 没有什么意义，因为 js 中对象太多
let my_obj: {
    id: number,
    name: string,
    isEnable: boolean
}

my_obj = {
    id: 1,
    name: "zhangsan",
    isEnable: true
}

console.log(my_obj)
```



## 函数

```typescript
// 函数 function
// 要规定函数的 输入类型 和 返回类型
// 在形参后面接冒号声明 形参的类型，在 ()后面冒号声明 返回值类型
// 也可以为函数添加可选参数 这里用 ? 即可，这样我们就可以调用两个参数或者三个参数不报错
function myFun1 (a: number, b: number, c?: number): number {
    return a + b
}

// 注意：可选参数之后不能再加规定类型的形参 error: A required parameter cannot follow an optional parameter.
// function myFun2 (a: number, b: number, c?: number, d: number): void {}
// 可以把它添加个 ？变为可选参数
function myFun3 (a: number, b: number, c?: number, d?: number): void {}

console.log(myFun1(1, 2))

// 除了上面这种声明式写法还有一种表达式写法
const res = (a: number, b: number): number => {
    return a - b
}

interface ISum {
    (a: number, b: number): number
}

let mysum: ISum = res

console.log(mysum)
```



`function.length`，就是==第一个具有默认值之前的参数个数==

```typescript
const fun0 = () => {}
const fun1 = (a: any) => {}
const fun2 = (a: any, b: any) => {}

// fun0.length === 0
console.log('fun0', fun0.length)
// fun1.length === 1
console.log('fun1', fun1.length)
// fun2.length === 2
console.log('fun2', fun2.length)

// 123['toString'] === 1
console.log(123['toString'])

function fn1 (name: any) {}

function fn2 (name = 'aaa') {}

function fn3 (name: string, age = 22) {}

function fn4 (name: string, age = 22, gender: string) {}

function fn5(name = 'aaa', age: number, gender: string) { }

console.log(fn1.length) // 1
console.log(fn2.length) // 0
console.log(fn3.length) // 1
console.log(fn4.length) // 1
console.log(fn5.length) // 0
```



### 可选参数和非空断言操作符

**可选参数**

```typescript
function buildName(firstName: string, lastName?: string) {
    return firstName + ' ' + lastName
}

// 错误演示
buildName("firstName", "lastName", "lastName")
// 正确演示
buildName("firstName")
// 正确演示
buildName("firstName", "lastName")
```



**非空断言操作符**

> 能确定变量值一定不为空时使用。与可选参数不同的是，非空断言操作符不会防止出现 null 或 undefined

```typescript
let s = e!.name;  // 断言e是非空并访问name属性
```



## 类

```typescript
// 在 ES6 中就有 类的概念了，在 TS 中对类添加一些功能
class Person {
    private name: string
    static money: number = 100
    static readonly slogan: string = 'new bee'

    constructor(name?: string) {
        this.name = name
    }

    eat() {
        console.log(`${this.name} 在吃饭`)
    }

    setName(name: string) {
        this.name = name
    }
}

const zs = new Person("张三")
zs.eat()
zs.setName('lisi')
zs.eat()

// 1、3个访问修饰符，和 Java 一样
// Public: 修饰的属性或方法是共有的 在 任何地方 都能访问
// Protected: 修饰的属性或方法是受保护的 在 本类 和 子类中 能够访问
// Private: 修饰的属性或方法是私有的 只有 本类 中访问

// 2、静态属性 static
// 使用 static 修饰的属性是通过 类 去访问，是每个实例共有的
// 同样 static 可以修饰 方法，用 static 修饰的方法称为 类方法，可以使用类直接调用
Person.money = 99
console.log(Person.money)

// 3、只读 readonly
// 给属性添加上 readonly 就能保证该属性只读，不能修改，如果存在 static 修饰符，写在其后
// error: Cannot assign to 'slogan' because it is a read-only property.
// Person.slogan = 'old bee'
console.log(Person.slogan)

// 4、抽象类 abstract
// TS 新增的抽象类 写一个类的时候，不希望直接使用该类创建实例**（不能被 new ）**那么我们把它设置为抽象类，让它不能被实例化 只能被继承


// 在 class 前面 添加 abstract 修饰符
abstract class Animal {
    protected name: string

    constructor(name: string) {
        this.name = name
    }

    // 在抽象类中 可以写 抽象方法 ，抽象类没有方法体
    abstract say(): string
    abstract eat(): void
}

class Cat extends Animal {
    eat(): void {
        console.log(`${this.name} eating...fish`)
    }

    say(): string {
        return "miao miao miao";
    }
}

const m_cat = new Cat('lmao')
m_cat.eat()
console.log(m_cat.say())
```



## 修饰符/装饰器 @

> 装饰器是一种特殊类型的声明，它能够被附加到类声明，方法， 访问符，属性或参数上，可以修改类的行为。 装饰器使用 @expression这种形式，expression求值后必须为一个函数，它会在运行时被调用，被装饰的声明信息做为参数传入。类似Spring AOP中的环绕通知；手写一个模版方法也可起到同样的作用。



### 四种装饰器

> 在TypeScript中装饰器可以修饰四种语句：类，属性，访问器，方法以及方法参数。

1、**类装饰器**，应用于类构造函数，其参数是类的构造函数。注意`class`并不是像 Java 那种强类型语言中的类，而是 JavaScript 构造函数的语法糖



2、**方法装饰器**，会被应用到方法的属性描述符上，可以用来监视/修改/替换方法定义。
方法装饰会在运行时传入下列3个参数：

a）对于静态成员来说是类的构造函数，对于实例成员是类的原型对象

b）成员的名字

c）成员的属性描述符



3、**方法参数装饰器**，参数装饰器表达式会在运行时当作函数被调用，传入下列3个参数：

a）对于静态成员来说是类的构造函数，对于实例成员是类的原型对象。

b）参数的名字。

c）参数在函数参数列表中的索引。



4、**属性装饰器**

属性装饰器表达式会在运行时当作函数被调用，传入下列2个参数：

a）对于静态成员来说是类的构造函数，对于实例成员是类的原型对象。

b）成员的名字。



### 装饰器执行时期

修饰器对类的行为的改变，是代码编译时发生的（JS 在执行机中编译的阶段），而不是在运行时。这意味着，修饰器能在编译阶段运行代码。**修饰器本质就是编译时执行的函数**。在 Node.js 环境中模块一加载时就会执行。



### 自定义装饰器



在`TypeScript`中装饰器还属于实验性语法，所以要想使用必须在配置文件中`tsconfig.json`编译选项中开启

```typescript
{
    "compilerOptions": {
        "experimentalDecorators": true
    }
}
```



```typescript
function test(f: any) {
    console.log('function run...')
}

@test
class HelloClazz {}
```

## 接口

> 接口 interface 它能很方便的帮我们定义 Object 类型，它是非常的灵活可以描述对象的各种类型。与 java 的 interface 有些区别

```typescript
// 为了解决 继承 的困境(不能实现多继承)
// 接口可以多实现
// 接口之间可以多继承
interface IAnimal {
    eat(): void
}

interface IBeing {
    run(): void
}

interface People extends IAnimal, IBeing{

    say(): void

    sleep(): void
}

class Men implements People {
    run(): void {
    }

    sleep(): void {
    }

    eat(): void {
    }

    say(): void {
    }
}

interface Device {
    // readonly 不可改变的，定义完后就不能修改，是不是和 const 有点像，不过 const 是针对变量， readonly 是针对属性
    readonly id?: number
    brand: string
    type: string
    // 可选修饰符”?“，可选修饰符以?定义，为什么需要可选修饰符呢，因为如果我们不写可选修饰符，那interface里面的属性都是必填的
    // 在 interface 属性中添加 ”？“， 则该属性在赋值的时候可以省略
    price?: number
}

let MiPhone: Device = {
    id: 1111,
    brand: 'xiaomi',
    type: 'mi-phone',
    price: 1999.0
}

let HWPhone: Device = {
    brand: 'huawei',
    type: 'hw-phone',
}

console.log(MiPhone)
MiPhone.type = 'mix'
console.log(MiPhone)
console.log(HWPhone)
```



## Type 类型别名

```typescript
// Type
// 声明类型别名使的，别名类型只能定义是：基础静态类型、对象静态类型、元组、联合类型
// type别名不可以定义interface

// type类型别名和interface接口的区别

// 1.Types 不可以出现重复类型名称
// 报错 Duplicate identifier 'Types'.
// type Types = string
// type Types = number

// interface 接口可以出现重复类型名称，如果重复出现则是，合并起来也就是变成 { name：string, age: number }
interface Types1 {
    name: string
}
interface Types1 {
    age: number
}

const myTypes1: Types1 = {
    name: '',
    age: 10
}
```





### Type 与 Interface

**相同点**：

1、**都可以用来描述一个对象或函数**

```typescript
// interface
interface User {
  name: string
  age: number
}

interface SetUser {
  (name: string, age: number): void;
}

// type
type User = {
  name: string
  age: number
};

type SetUser = (name: string, age: number)=> void;
```

2、**都允许继承（extends）**，interface 和 type 都可以拓展，并且两者并不是相互独立的。也就是说 interface 可以 extends type，type 也可以 extends interface 。 虽然效果差不多，但是两者语法不同。

```typescript
// interface extends interface
interface Name { 
  name: string; 
}
interface User extends Name { 
  age: number; 
}

// type extends type
type Name = { 
  name: string; 
}
type User = Name & { age: number  };

// interface extends type
type Name = { 
  name: string; 
}
interface User extends Name { 
  age: number; 
}

// type extends interface
interface Name { 
  name: string; 
}
type User = Name & { 
  age: number; 
}
```



**不同点**：

1、`type` 可以声明基本类型别名，联合类型，元组等类型

```typescript
// 基本类型别名
type Name = string

// 联合类型
interface Dog {
    wang();
}
interface Cat {
    miao();
}

type Pet = Dog | Cat

// 具体定义数组每个位置的类型
type PetList = [Dog, Pet]
```

2、`type` 语句中还可以使用 `typeof`获取实例的 类型进行赋值

```typescript
// 当你想获取一个变量的类型时，使用 typeof
let div = document.createElement('div');
type B = typeof div

// 其他骚操作
type StringOrNumber = string | number;  
type Text = string | { text: string };  
type NameLookup = Dictionary<string, Person>;  
type Callback<T> = (data: T) => void;  
type Pair<T> = [T, T];  
type Coordinates = Pair<number>;  
type Tree<T> = T | { left: Tree<T>, right: Tree<T> };
```

3、`interface` 能够声明合并

```typescript
interface User {
  name: string
  age: number
}

interface User {
  sex: string
}

/*
User 接口为 {
  name: string
  age: number
  sex: string 
}
*/
```

4、`interface` 有可选属性和只读属性

```typescript
// 可选属性 ?
// 接口里的属性不全都是必需的。 有些是只在某些条件下存在，或者根本不存在。 例如给函数传入的参数对象中只有部分属性赋值了。带有可选属性的接口与普通的接口定义差不多

// 只读属性 readonly
// readonly修饰的属性是不可写的，对象属性的值只能在对象刚刚创建的时候修改
interface Device {
    // readonly 不可改变的，定义完后就不能修改，是不是和 const 有点像，不过 const 是针对变量， readonly 是针对属性
    readonly id?: number
    brand: string
    type: string
    // 可选修饰符”?“，可选修饰符以?定义，为什么需要可选修饰符呢，因为如果我们不写可选修饰符，那interface里面的属性都是必填的
    // 在 interface 属性中添加 ”？“， 则该属性在赋值的时候可以省略
    price?: number
}
```



## 枚举

```typescript
// 1、数字枚举
// 默认值从 0 开始
enum Week {
    // 自定义默认值 Mon = a 后，后续的枚举依次加 a
    Mon = 1,
    Tue,
    Wed,
    Thu,
    Fri,
    Sat,
    Sun
}

console.log(Week.Mon)
console.log(Week.Tue)

// 也可以通过下标获取
// concat() 在Week[4] 的值后面拼接'a','b', 'c' ==》 Thuabc
console.log(Week[4].concat('a','b', 'c'))

/ 2、字符串枚举
enum Week {
    Mon = "Mon",
    Tue = "Tue",
    Wed = "Wed",
    Thu = "Thu",
    Fri = "Fri",
    Sat = "Sat",
    Sun = "Sun"
}

function enumCondition(val: string): boolean {
    return val == Week.Wed;
}

const res = enumCondition('Wed')
console.log(res)

// 3、常量枚举
// 在 enum 前面添加一个 const 即可，它提高了性能
// const enum Week
```



## 泛型

```typescript
// 泛型就像一个占位符一个变量，在使用的时候我们可以将定义好的类型像参数一样传入，原封不动的输出、
function getVal<T>(param: T): T {
    return param
}

// 多个泛型
function getVals<T, U>(param: [T, U]): [T, U] {
    return param
}

console.log(getVal('aaa'))
console.log(getVals(['aaa', 111]))

// 可以使用 interface 来约束 泛型
// 在 T 后面 extends Ilen ，定义 Ilen 里面代码表示，T 必须要有 length 属性

interface ILen {
    readonly length: number
}

function getLen<T extends ILen>(param: T): number {
    return param.length
}

console.log(getLen('abc'))
console.log(getLen([]))
```



## 声明文件与命名空间

> `declare` 和 `namespace`

```typescript
// shims-tsx.d.ts
import Vue, { VNode } from 'vue';

// declare：当使用第三方库时，我们需要引用它的声明文件，才能获得对应的代码补全、接口提示等功能
// shims-tsx.d.ts， 在全局变量 global 中批量命名了数个内部模块。
/*
declare var 声明全局变量
declare function 声明全局方法
declare class 声明全局类
declare enum 声明全局枚举类型
declare global 扩展全局变量
declare module 扩展模块
*/
declare global {
  // namespace：“内部模块”现在称做“命名空间”
  namespace JSX {
    // tslint:disable no-empty-interface
    interface Element extends VNode {}
    // tslint:disable no-empty-interface
    interface ElementClass extends Vue {}
    interface IntrinsicElements {
      [elem: string]: any;
    }
  }
}

// shims-vue.d.ts
// shims-vue.d.ts，意思是告诉 TypeScript *.vue 后缀的文件可以交给 vue 模块来处理
declare module '*.vue' {
  import Vue from 'vue';
  export default Vue;
}
```





## 参考

[Vue3.0 前的 TypeScript 最佳入门实践](https://juejin.cn/post/6844903749501059085)

[TypeScript装饰器（decorators） - 一箭中的 - 博客园 (cnblogs.com)](https://www.cnblogs.com/winfred/p/8216650.html)