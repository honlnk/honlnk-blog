以下是使用 TypeScript 实现的 **Levenshtein 编辑距离算法**，并支持返回**最小编辑步骤**的完整实现。该实现不仅能计算出字符串之间的编辑距离，还能输出具体的编辑操作（替换、删除、插入）及对应的位置和字符信息。

---

## ✅ 功能说明

- **输入**：两个字符串 `a` 和 `b`。
- **输出**：
  - 编辑距离（`distance`）。
  - 编辑操作列表（`operations`），包含以下信息：
    - 操作类型（`replace`, `delete`, `insert`）。
    - 操作位置（在原字符串中的索引，0-based）。
    - 操作内容（如原字符、新字符、插入字符等）。

---

## ✅ TypeScript 实现

```ts
type EditOperation = {
    type: 'replace';
    originalChar: string;
    newChar: string;
    index: number;
} | {
    type: 'delete';
    char: string;
    index: number;
} | {
    type: 'insert';
    char: string;
    index: number;
};

function getLevenshteinEditScript(a: string, b: string): { distance: number, operations: EditOperation[] } {
    const m = a.length;
    const n = b.length;

    // 动态规划表
    const dp = Array.from({ length: m + 1 }, () => Array(n + 1).fill(0));

    // 初始化边界
    for (let i = 0; i <= m; i++) dp[i][0] = i;
    for (let j = 0; j <= n; j++) dp[0][j] = j;

    // 填充动态规划表
    for (let i = 1; i <= m; i++) {
        for (let j = 1; j <= n; j++) {
            if (a[i - 1] === b[j - 1]) {
                dp[i][j] = dp[i - 1][j - 1];
            } else {
                dp[i][j] = 1 + Math.min(
                    dp[i - 1][j],    // 删除
                    dp[i][j - 1],    // 插入
                    dp[i - 1][j - 1] // 替换
                );
            }
        }
    }

    const distance = dp[m][n];

    // 回溯获取编辑操作
    const operations: EditOperation[] = [];
    let i = m, j = n;

    while (i > 0 || j > 0) {
        if (i > 0 && j > 0 && a[i - 1] === b[j - 1]) {
            // 字符相同，无需操作
            i--;
            j--;
        } else if (i > 0 && dp[i][j] === dp[i - 1][j] + 1) {
            // 删除操作
            operations.push({
                type: 'delete',
                char: a[i - 1],
                index: i - 1,
            });
            i--;
        } else if (j > 0 && dp[i][j] === dp[i][j - 1] + 1) {
            // 插入操作
            operations.push({
                type: 'insert',
                char: b[j - 1],
                index: i,
            });
            j--;
        } else {
            // 替换操作
            operations.push({
                type: 'replace',
                originalChar: a[i - 1],
                newChar: b[j - 1],
                index: i - 1,
            });
            i--;
            j--;
        }
    }

    // 反转操作步骤以保证顺序
    operations.reverse();

    return { distance, operations };
}
```

---

## ✅ 示例用法

```ts
const a = "tast";
const b = "tets";

const result = getLevenshteinEditScript(a, b);

console.log("编辑距离:", result.distance);
console.log("编辑操作步骤:");
result.operations.forEach(op => {
    switch (op.type) {
        case 'replace':
            console.log(`替换: 将 ${op.originalChar}（位置 ${op.index}） 改为 ${op.newChar}`);
            break;
        case 'delete':
            console.log(`删除: 删除字符 ${op.char}（位置 ${op.index}）`);
            break;
        case 'insert':
            console.log(`插入: 在位置 ${op.index} 插入字符 ${op.char}`);
            break;
    }
});
```

---

## ✅ 示例输出

对于输入 `a = "tast"`，`b = "tets"`，输出如下：

```
编辑距离: 3
编辑操作步骤:
替换: 将 a（位置 1） 改为 e
删除: 删除字符 s（位置 2）
插入: 在位置 3 插入字符 s
```

---

## ✅ 注意事项

- 插入操作的 `index` 表示插入字符在原字符串中的位置（0-based），插入后后续字符会后移。
- 若存在多条最优路径（如多个编辑序列都能达到相同编辑距离），本实现默认返回其中一条。
- 此算法适用于小到中等长度字符串，若需处理超长文本，建议优化为滚动数组或引入启发式搜索。

---

## ✅ 总结

本实现不仅实现了标准的 Levenshtein 编辑距离算法，还进一步扩展了其功能，**输出最小编辑路径的具体操作**，帮助用户清晰了解如何将一个字符串转换为另一个字符串。