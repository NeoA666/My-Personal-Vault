GitHub Copilot（模型：GPT-5 mini）

- String：拥有所有权的可增长 UTF‑8 文本。存储在堆上，类型是 owned string（例如：let s = String::from("hello")）。可以修改（push_str、push 等）。
- &str（字符串切片）：对字符串的借用视图，通常指向静态文本或 String 的一部分，不能修改（例如："hello" 或 &s[..]）。
- Vec<String>：一个可增长的动态数组，元素类型是 String。整个 Vec 拥有它里面每个 String 的所有权，可以 push/pop 元素（例如把一堆单词放在一起）。

要点举例：
- 可变性/所有权：String 拥有字符数据；&str 不拥有，只借用。Vec<String> 拥有并管理多个 String。
- 常用方法：
  - String: push_str, push, len, as_str
  - &str: split, contains（只读）
  - Vec<String>: push, pop, len, iter
- 转换：
  - &str -> String: "s".to_string() 或 String::from("s")
  - String -> &str: &s 或 s.as_str()
  - split -> Vec<String>: s.split_whitespace().map(|p| p.to_string()).collect()
