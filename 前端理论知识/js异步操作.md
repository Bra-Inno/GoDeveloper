`async` 和 `await` 是 JavaScript 中用于处理异步操作的语法糖，它们让异步代码的编写和阅读更加直观，类似于同步代码。

### 1. `async` 函数

- `async` 用于声明一个异步函数。异步函数会自动返回一个 `Promise` 对象。
- 如果函数返回值不是 `Promise`，它会被自动包装成 `Promise`。

```javascript
async function fetchData() {
    return "Data fetched";
}

fetchData().then(data => console.log(data)); // 输出: Data fetched
```

### 2. `await` 表达式

- `await` 只能在 `async` 函数中使用，用于等待一个 `Promise` 完成并返回其结果。
- `await` 会暂停 `async` 函数的执行，直到 `Promise` 完成。

```javascript
async function fetchData() {
    let response = await fetch('https://api.example.com/data');
    let data = await response.json();
    return data;
}

fetchData().then(data => console.log(data));
```

### 3. 错误处理

- 使用 `try...catch` 来捕获 `await` 表达式中可能出现的错误。

```javascript
async function fetchData() {
    try {
        let response = await fetch('https://api.example.com/data');
        let data = await response.json();
        return data;
    } catch (error) {
        console.error('Error:', error);
    }
}

fetchData();
```

### 4. 并行执行

- 如果需要并行执行多个异步操作，可以使用 `Promise.all`。

```javascript
async function fetchMultipleData() {
    let [data1, data2] = await Promise.all([
        fetch('https://api.example.com/data1').then(response => response.json()),
        fetch('https://api.example.com/data2').then(response => response.json())
    ]);
    return { data1, data2 };
}

fetchMultipleData().then(data => console.log(data));
```

### 5. 总结

- `async` 用于声明异步函数，自动返回 `Promise`。
- `await` 用于等待 `Promise` 完成，只能在 `async` 函数中使用。
- 使用 `try...catch` 处理错误。
- 使用 `Promise.all` 实现并行异步操作。

通过 `async` 和 `await`，异步代码的编写和阅读变得更加简洁和直观。





```javascript
// 模拟一个异步的耗时操作
function mockAsyncOperation(time) {
    setTimeout(output=() => {
            console.log(`异步操作完成，耗时${time}ms`);
        }, time);
}
// 方式1: async/await
async function awaitMethod() {
    console.time('await方式');
    try {
        let [result1,result2,result3]=await Promise.all([
            mockAsyncOperation(1000),
            mockAsyncOperation(500),
            mockAsyncOperation(300)
        ]);
        return [result1,result2,result3];
    } catch (error) {
        console.error('await错误:', error);
    }
    console.timeEnd('await方式');
}
// 执行性能测试
async function runPerformanceTest() {
    console.log('开始性能测试...');
    // 多次运行以获得平均值
    const  iterations = 5;
    const awaitTimes = [];
    for(let i = 0; i < iterations; i++) {
        const awaitStart = performance.now();
        await awaitMethod();
        awaitTimes.push(performance.now() - awaitStart);
    }    
    // 计算平均执行时间
    const awaitAvg =await awaitTimes.reduce((a, b) => a + b, 0) / iterations;
    console.log(`\n测试结果(${iterations}次平均):`);
    console.log('await方式平均耗时:', awaitAvg.toFixed(2) + 'ms');
}
// 运行测试
runPerformanceTest();
```



在你的代码中，`console.log` 语句在 `await` 操作之前执行的原因是因为 `await` 是异步的，而 `console.log` 是同步的。JavaScript 会先执行同步代码，然后再处理异步代码。因此，`console.log` 会在 `await` 操作完成之前执行。

为了让 `console.log` 在 `await` 操作完成之后执行，你需要确保 `console.log` 是在 `await` 操作完成之后才被调用。你可以通过将 `console.log` 放在 `await` 操作之后来实现这一点。

以下是修正后的代码：

```javascript
async function runPerformanceTest() {
    console.log('开始性能测试...');
    
    // 多次运行以获得平均值
    const iterations = 5;
    const awaitTimes = [];

    for(let i = 0; i < iterations; i++) {
        const awaitStart = performance.now();
        await awaitMethod(); // 等待异步操作完成
        awaitTimes.push(performance.now() - awaitStart);
    }

    // 计算平均执行时间
    const awaitAvg = awaitTimes.reduce((a, b) => a + b, 0) / iterations;

    // 确保在异步操作完成后才执行
    console.log(`\n测试结果(${iterations}次平均):`);
    console.log('await方式平均耗时:', awaitAvg.toFixed(2) + 'ms');
}

// 假设的异步方法
async function awaitMethod() {
    // 模拟异步操作
    return new Promise(resolve => setTimeout(resolve, 100));
}

// 执行性能测试
runPerformanceTest();
```

### 关键点：

1. **`await` 是异步的**：`await` 会暂停当前异步函数的执行，直到 `Promise` 完成。因此，`await` 之后的代码会在 `Promise` 解决后执行。
2. **同步代码优先执行**：在 `await` 之前的同步代码（如 `console.log`）会立即执行，而不会等待 `await` 完成。

### 修正后的逻辑：
- `await awaitMethod();` 会等待 `awaitMethod` 完成。
- `awaitTimes.push(performance.now() - awaitStart);` 会在 `awaitMethod` 完成后执行。
- 最后，`console.log` 会在所有 `await` 操作完成后执行。

这样，`console.log` 就会在性能测试完成后输出结果，确保结果的准确性。