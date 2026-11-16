# JSON 格式化、校验与转 TypeScript：几个容易忽略的细节

JSON 看起来是最简单的格式，但实际处理时的问题几乎都在边界上：大文件卡死、大整数精度丢失、从 JSON 推断出的类型是错的。

这篇讲这几个具体问题。

## 格式化 != 只是加缩进

大多数在线工具做的事是 `JSON.parse` 再 `JSON.stringify(obj, null, 2)`。这个流程有三个副作用：

**1. 键顺序可能变**

ES2015 之后，整数键（`"1"`, `"2"`）会被排到前面，字符串键保持插入顺序。如果你的 JSON 里有数字键，格式化后顺序会和原文件不一样。

**2. 精度丢失**

```js
JSON.parse('{"id": 9007199254740993}')
// → { id: 9007199254740992 }   ← 最后一位变了
```

JavaScript 的 Number 是 IEEE 754 双精度，安全整数上限是 `2^53 - 1`（9007199254740991）。超过就丢精度。

**这是真实事故的来源**：Twitter 的 snowflake ID、数据库的 bigint 主键、微信的 openid，都可能超过这个范围。

**解决**：用支持 BigInt 的解析器，或者先当字符串处理：

```js
const text = '{"id": 9007199254740993}';
// 办法 1：预处理，把大整数加引号
const safe = text.replace(/([:[,]\s*)(-?\d{16,})/g, '$1"$2"');
JSON.parse(safe);   // { id: "9007199254740993" }

// 办法 2：用 lossless-json 库
```

**3. 注释和尾随逗号会直接报错**

JSON 标准不允许注释和末尾逗号。JSON5 / JSONC 允许，但 `JSON.parse` 不认。VS Code 的 `settings.json` 就是 JSONC，直接喂给 `JSON.parse` 会炸。

## 大文件：为什么 100MB 的 JSON 会卡死

`JSON.parse` 是**全量解析**——必须把整个字符串读进内存，构建完整对象树。

100MB 的 JSON 文件，解析后占用的内存通常是原文件的 3-5 倍（对象开销 + 字符串拷贝）。浏览器标签页很容易 OOM。

**解决办法**：

**流式解析（streaming）**

不构建完整对象，边读边处理。库：

```
Node      stream-json、@streamparser/json
浏览器     @streamparser/json（支持 Web Streams）
Python    ijson
```

典型用法：

```js
import { parser } from 'stream-json';
import { streamArray } from 'stream-json/streamers/StreamArray';
import { createReadStream } from 'fs';

createReadStream('big.json')
  .pipe(parser())
  .pipe(streamArray())          // 假设顶层是数组
  .on('data', ({ value }) => {
    console.log(value.id);      // 逐条处理，内存恒定
  });
```

**什么时候不需要流式**：文件 < 10MB 就直接 `JSON.parse`，流式解析的复杂度不值得。

## 从 JSON 生成 TypeScript 类型：推断会错的地方

自动生成类型（json2ts 那类）本质是**从样本推断结构**，推断规则有硬伤：

**1. 空数组推不出元素类型**

```json
{ "items": [] }
```
→ `items: never[]` 或者 `any[]`，取决于工具。

**2. null 值的处理**

```json
{ "name": null }
```
→ 生成 `name: null`，但实际运行时可能变成字符串。应该生成 `name: string | null`。

**3. 数字不分 int / float**

```json
{ "count": 1, "price": 9.9 }
```
→ 都是 `number`。但如果你要生成数据库 schema，这个区分很重要。

**4. 同名字段类型不一致**

```json
[{"id": 1}, {"id": "2"}]
```
→ 有的工具报错，有的生成 `number | string`，有的只取第一个。

**5. 日期是字符串**

JSON 没有日期类型。`"2026-11-16T10:00:00Z"` 被推成 `string`，但你实际想用 `Date`。

**正确姿势**：把自动生成的类型当草稿，手工修一遍这些地方。或者用 JSON Schema 作为真相来源，从 schema 生成类型（`json-schema-to-typescript`）。

## 校验：JSON.parse 只检查语法

`JSON.parse` 能通过，不代表数据是对的。缺字段、类型错、值为 null，它都不管。

**结构校验要用 JSON Schema**：

```json
{
  "type": "object",
  "required": ["id", "name"],
  "properties": {
    "id":   { "type": "integer", "minimum": 1 },
    "name": { "type": "string", "minLength": 1 },
    "tags": { "type": "array", "items": { "type": "string" } }
  }
}
```

用 `ajv` 校验：

```js
import Ajv from 'ajv';
const ajv = new Ajv();
const validate = ajv.compile(schema);
if (!validate(data)) console.log(validate.errors);
```

## 格式化时的排版选择

```
2 空格        最常见，Git diff 友好
4 空格        Java / PHP 社区
Tab           可调宽度，但团队里容易吵
不缩进（一行）  适合传输，不适合人读
```

**建议**：配置文件用 2 空格（生态最广），日志/传输用不缩进。

另外：中文内容在 JSON 里默认会被转成 `\uXXXX`（`JSON.stringify` 的行为）。要保留中文用：

```js
JSON.stringify(obj, null, 2).replace(/[\u0080-\uffff]/g,
  c => c);   // 不转义
```

或者干脆在输出时不过 `stringify`（自己拼接）。

## 免费在线工具

浏览器本地格式化、校验、转 TypeScript：

- https://yuantou.one/tools/json/

支持大文件分块处理、树形折叠、一键复制类型定义。

---

## 相关阅读

- [字幕格式互转](https://github.com/cointaboo/subtitle-format-converter)
- [TOTP 二步验证原理](https://github.com/cointaboo/totp-2fa-guide)
