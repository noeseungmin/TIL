# JavaScript [chap.1]

```javascript

```

## 선언
JavaScript의 선언에는 3가지 방법이 있습니다.

### var (ES6 let과 const를 지원하며 현재 사용 X)
var의 세가지 문제점
1. 선언을 중복해서 할 수 있다.
```javascript
var code = 10;
var code = 20;
console.log(`code: ${code}`);  // 'code: 20'
```

* 같은 이름의 변수를 중복해서 선언해도 정상적으로 동작하며, 가장 마지막 값을 저장하고 있는다.


2. scope의 범위가 전역 / 함수로 제한된다.
```javascript
var code = 10;
if (true) {
  var code = 0;
  var codeName = 'Hello JS!';
}
console.log(`code: ${code}`);  // 'code: 0'
console.log(`codeName: ${codeName}`);  // 'codeName: Hello JS!'
```

* if 구문 안에서 새로운 변수를 사용했지만 기존 변수에 데이터가 재할당 되었다. 위에서 본 '1) 선언을 중복해서 할 수 있다'와 동일하게 동작했다.
  이를 통해 if 구문 안의 scope는 전역(Global)과 동일하게 취급받는 것을 알 수 있다.   



3. Hoisting이 가능하다.
```javascript
console.log(`Before declare: ${code}`);  // 'Before declare: undefined'
var code = 10;
console.log(`After declare: ${code}`);  // 'After declare: 10'
```
* JS는 변수의 선언이 실행 시점(Runtime)에 동작할 것 같지만 실행 전에 변수와 함수를 미리 실행한다.
함수만 정의해 놓았는데 자동으로 실행되는 경험이 한 번씩은 있었을 것이다. 이것은 정의된 함수가 실행 시점보다 이전에 동작하기 때문이다.
마찬가지로 변수도 실행 시점 전에 동작한다. 그래서 위 코드처럼 선언되기 전의 변수를 사용하는 코드가 가능하다. 이를 호이스팅이라고 한다. 
- - -
### let, const
1. 선언 중복 X, 때에 따라 재할당도 X
```javascript
let code = 10;
let code = 20;  // SyntaxError: Identifier 'code' has already been declared.
```

* let은 기존의 var를 대체하는 ‘변수’의 역할을 한다.
* let은 중복된 선언이 문법으로 불가능하다. 그래서 전역에서 중복된 이름이 정의된다면 바로 오류가 난다.

* 재할당은 가능하다.
```javascript
let code = 10;
code = 20;
console.log(`code: ${code}`);  // 'code: 20'
```

```javascript
const code;  // SyntaxError: Missing initializer in const declaration.
const code = 10;
code = 0;  // TypeError: Assignment to constant variable.
```
* const는 상수의 역할이므로 선언과 동시에 할당이 되어야 하며 중복 선언, 재할당 모두 불가능하다.


2.  block 단위의 scope를 사용할 수 있다.
```javascript
let code = 10;
if (true) {
  let code = 0;
  let codeName = 'Hello JS!';
}
console.log(`code: ${code}`);  // 'code: 10'
console.log(`codeName: ${codeName}`);  // ReferenceError: codeName is not defined
```

```javascript
const code = 10;
if (true) {
  const code = 0;
  const codeName = 'Hello JS!';
}
console.log(`code: ${code}`);  // 'code: 10'
console.log(`codeName: ${codeName}`);  // ReferenceError: codeName is not defined
```
* var와 const는 블록 단위로 살아있는다. 그래서 codeName을 더 이상 전역에서 사용할 수 없게 된다.


3. Hoisting이 불가능하다
```javascript
console.log(`Before declare: ${code}`);  // ReferenceError: Cannot access 'code' before initialization
let code = 10;
console.log(`After declare: ${code}`);
```
* let과 const는 더 이상 실행 시점에 동작하지 않는다. 이로 인해 변수의 라이프사이클이 상식적으로 되었다.
  당연히 변수를 선언한 시점부터 변수를 사용할 수 있어야 자연스럽지 않을까?
### 

