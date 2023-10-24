# TypeScript



## 入门

```shell
# 先全局安装，为了使用 tsc 等编译命令
npm install -g typescript
# 再进行项目安装
npm install -D typescript
# 最后安装 ts-node 运行 ts 文件
npm install -D ts-node

# 编译运行时可能会提示未安装 @types/node，直接安装即可
# npm install -D @types/node
```



## 类型

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



### 特殊类型

```ts
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



### 数组

```ts
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



### 元组

```ts
// 元组数据类型需要跟给定的变量类型一致
let my_tuple: [string, boolean, number] = ['aaa', false, 100]
console.log(my_tuple)
```



### 对象

```ts
// 声明一个 ts 对象
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

…

---

## 类型操作



### 字面量

> 定义什么值就只能赋什么值

```typescript
let dog: 'dog'

// dog = 'cat' // error: '"cat"' is not assignable to type '"dog"'.
dog = 'dog' // correct

// 同时赋予多个不同类型的可选值，用"|"分开
let my_var: 'cat'|'dog'|false|100
my_var = 'cat'
my_var = 'dog'
my_var = false
my_var = 100
```



### 联合类型

> 对于一个变量的类型可能是几种类型的时候我们可以使用 any ，但是 any 的范围有点大，不到万不得已不使用；如果知道是其中的哪几种类型的话，我们就可以使用**联合类型**，多个联合类型之间用 `|` 分隔。

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

// querySelector 拿不到 DOM 的时候返回 null
const ele: HTMLElement | null = document.querySelector('.main')
```



### 类型守卫

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



### 类型断言

```typescript
// 类型断言
// 在上面联合类型的变量传入的时候，我们声明了这个类型为 number | string 它不能调用 length 方法
// 这里我们就可以用到 类型断言指定类型为 string
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

> 不要滥用类型断言，只在能够确保代码正确的情况下去使用它。



### 非空断言操作符

> 能确定变量值一定不为空时使用。

```ts
let s = e!.name;  // 断言 e 是非空并访问 name 属性
```

> 与可选参数不同，非空断言操作符不会防止出现 `null` 或 `undefined`。

…



### 类型推论

TypeScript 会根据声明变量时赋值的类型，自动帮推导变量类型。

```ts
// 相当于 msg: string
let msg = 'Hello World'

// 所以要赋值为 number 类型时会报错
msg = 3 // Type 'number' is not assignable to type 'string'
```

TypeScript 会根据 `return` 的结果推导返回值类型。

```ts
// 相当于 getRandomNumber(): number
function getRandomNumber() {
  return Math.round(Math.random() * 10)
}

// 相当于 num: number
const num = getRandomNumber()
```

类型推论的前提是变量在声明时有明确的值，如果一开始没有赋值，那么会被默认为 `any` 类型。

```ts
// 此时相当于 foo: any
let foo

// 所以可以任意改变类型
foo = 1 // 1
foo = true // true
```

…



---

## 函数

> * 要规定函数的*参数类型*和*返回类型*；
> * *可选参数*，参数后带 `?` 表示该参数是一个可选参数。可选参数必须排在必须参数的后面。

```typescript
function func(str: string, num: number): void {}

const func2 = function(): void {}

// 除了上面这种声明式写法还有一种表达式写法
const res = (a: number, b: number): number => a - b
}

interface ISum {
    (a: number, b: number): number
}

let mysum: ISum = res

console.log(mysum)
```

…

---

### 可选参数

```ts
// 也可以为函数添加可选参数 这里用 ? 即可，这样我们就可以调用两个参数或者三个参数不报错
function myFun1 (a: number, b: number, c?: number): number {
    return a + b
}

// 注意：可选参数之后不能再加规定类型的形参 error: A required parameter cannot follow an optional parameter.
// function myFun2 (a: number, b: number, c?: number, d: number): void {}
// 可以把它添加个 ？变为可选参数
function myFun3 (a: number, b: number, c?: number, d?: number): void {}

console.log(myFun1(1, 2))

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

…

---

### 异步函数

对于异步函数，需要用 `Promise<T>` 类型来定义它的返回值，这里的 `T` 是泛型，取决于该函数最终返回一个什么样的值。

```ts
// 注意这里的返回值类型
function queryData(): Promise<string> {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve('Hello World')
    }, 3000)
  })
}

queryData().then((data) => console.log(data))
```

…

---

### function.length

`function.length`，就是**方法中|第一个|具有默认值的参数|之前的|参数个数**。

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

…

---

### 函数重载

```ts
function greet(name: string): string  // TS 类型
function greet(name: string[]): string[]  // TS 类型
// 真正的函数体，函数入参需要把可能涉及到的类型都写出来，用以匹配前两行的类型
// 函数的返回值类型可以省略，因为在第 1 、 2 行里已经定义过返回类型了。
function greet(name: string | string[]) {
  if (Array.isArray(name)) {
    return name.map((n) => `Welcome, ${n}!`)
  }
  return `Welcome, ${name}!`
}

// 单个问候语，此时只有一个类型 string
const greeting = greet('Petter')
console.log(greeting) // Welcome, Petter!

// 多个问候语，此时只有一个类型 string[]
const greetings = greet(['Petter', 'Tom', 'Jimmy'])
console.log(greetings)
// [ 'Welcome, Petter!', 'Welcome, Tom!', 'Welcome, Jimmy!' ]
```

…

---

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





## 新增

### Omit

可以在继承的过程中舍弃某些属性，通过 `Omit` 帮助类型来实现，`Omit` 的类型如下：

```ts
type Omit<T, K extends string | number | symbol>
```

其中 `T` 代表已有的一个对象类型， `K` 代表要删除的属性名，如果只有一个属性就直接是一个字符串，如果有多个属性，用 `|` 来分隔开。

```ts
interface UserItem {
  name: string
  age: number
  enjoyFoods: string[]
  friendList?: UserItem[]
}

// 这里在继承 UserItem 类型的时候，删除了两个多余的属性
interface Admin extends Omit<UserItem, 'enjoyFoods' | 'friendList'> {
  permissionLevel: number
}

// 现在的 admin 就非常精简了
const admin: Admin = {
  name: 'Petter',
  age: 18,
  permissionLevel: 1,
}
```

如果类上面本身有方法存在，接口在继承的时候也要相应的实现，当然也可以借助 `Omit` 帮助类型来去掉这些方法。

```ts
class UserBase {
  name: string
  constructor(userName: string) {
    this.name = userName
  }
  // 这是一个方法
  getName() {
    console.log(this.name)
  }
}

// 接口继承类的时候也可以去掉类上面的方法
interface User extends Omit<UserBase, 'getName'> {
  age: number
}

// 最终只保留数据属性，不带有方法
const petter: User = {
  name: 'Petter',
  age: 18,
}
```

…

---

## 配置详解

先全局安装 TypeScript

```shell
npm install -g typescript
```

再打开根目录进行配置初始化

```shell
tsc --init
```

```shell
Created a new tsconfig.json with:
                                                                                                    TS
  target: es2016
  module: commonjs
  strict: true
  esModuleInterop: true
  skipLibCheck: true
  forceConsistentCasingInFileNames: true
```

每个 `tsc` 命令行选项，都可以使用 `tsconfig.json` 中的一个字段来管理，比如 `--outDir`

```json
{
  "compilerOptions": {
    "target": "es6",
    "module": "es6",
    "outDir": "./dist"
  }
}
```

> 在实际工作中，项目都是通过一些脚手架创建的。`tsconfig.json` 也是脚手架提前配置好通用的选项，只需要在不满足条件的情况下去调整。

…

> 具体可以参考[官方配置](https://www.typescriptlang.org/tsconfig)

> 也可以参考[这个配置](https://jkchao.github.io/typescript-book-chinese/project/compilationContext.html#tsconfig-json)

…

---

## 项目脚手架与配置

项目中的配置如：

* tsconfig
* editor config 编辑器代码格式配置
* prettier 代码格式化
* eslint 语法解析
* …

> 参考：https://vue3.chengpeiquan.com/upgrade.html

…

---

## 参考

* [Vue3.0 前的 TypeScript 最佳入门实践](https://juejin.cn/post/6844903749501059085)
* [TypeScript装饰器（decorators）](https://www.cnblogs.com/winfred/p/8216650.html)
* https://vue3.chengpeiquan.com/typescript.html