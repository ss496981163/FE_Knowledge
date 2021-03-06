上篇文章[浅析Vue源码（四）—— $mount中template的编译--parse](https://juejin.im/post/5bb702b6e51d450e46286608)，我们介绍了compile 的 parse部分，至此我们完成了对一个html字符串模板解析成一个AST语法树的过程。下一步就是我们需要通过optimize方法，将AST节点进行静态节点标记。为后面 patch 过程中对比新旧 VNode 树形结构做优化。被标记为 static 的节点在后面的 diff 算法中会被直接忽略，不做详细的比较。

```
src/compiler/optimizer.js
```

```
export function optimize (root: ?ASTElement, options: CompilerOptions) {
  if (!root) return
  // staticKeys 是那些认为不会被更改静态的ast的属性
  isStaticKey = genStaticKeysCached(options.staticKeys || '')
  isPlatformReservedTag = options.isReservedTag || no
  // first pass: mark all non-static nodes.
  // 第一步 标记 AST 所有静态节点
  markStatic(root)
  // second pass: mark static roots.
  // 第二步 标记 AST 所有父节点（即子树根节点）
  markStaticRoots(root, false)
}
```
首先标记所有静态节点：

```
function isStatic (node: ASTNode): boolean {
  if (node.type === 2) { // 表达式
    return false
  }
  if (node.type === 3) { // 文本节点
    return true
  }
  // 处理特殊标记
  return !!(node.pre || ( // v-pre标记的
    !node.hasBindings && // no dynamic bindings
    !node.if && !node.for && // not v-if or v-for or v-else
    !isBuiltInTag(node.tag) && // not a built-in
    isPlatformReservedTag(node.tag) && // not a component
    !isDirectChildOfTemplateFor(node) &&
    Object.keys(node).every(isStaticKey)
  ))
}
```
ASTNode 的 type 字段用于标识节点的类型，可查看上一篇的 AST 节点定义：

type 为 1 表示元素，

type 为 2 表示插值表达式，

type 为 3 表示普通文本。

可以看到，在标记 ASTElement 时会依次检查所有子元素节点的静态标记，从而得出该元素是否为 static。上面 markStatic 函数使用的是树形数据结构的深度优先遍历算法，使用递归实现。 接下来继续标记静态树：

```
function markStaticRoots (node: ASTNode, isInFor: boolean) {
  if (node.type === 1) {
   // 用以标记在v-for内的静态节点。这个属性用以告诉renderStatic(_m)对这个节点生成新的key，避免patch error
    if (node.static || node.once) {
      node.staticInFor = isInFor
    }
    // For a node to qualify as a static root, it should have children that
    // are not just static text. Otherwise the cost of hoisting out will
    // outweigh the benefits and it's better off to just always render it fresh.
    // 一个节点如果想要成为静态根，它的子节点不能单纯只是静态文本。否则，把它单独提取出来还不如重渲染时总是更新它性能高。
    if (node.static && node.children.length && !(
      node.children.length === 1 &&
      node.children[0].type === 3
    )) {
      node.staticRoot = true
      return
    } else {
      node.staticRoot = false
    }
    if (node.children) {
      for (let i = 0, l = node.children.length; i < l; i++) {
        markStaticRoots(node.children[i], isInFor || !!node.for)
      }
    }
    if (node.ifConditions) {
      for (let i = 1, l = node.ifConditions.length; i < l; i++) {
        markStaticRoots(node.ifConditions[i].block, isInFor)
      }
    }
  }
}
```
markStaticRoots 函数里并没有什么特别的地方，仅仅是对静态节点又做了一层筛选。

## 总结
optimizer旨在为语法树的节点标上static和staticRoot属性。 遍历第一轮，标记static属性：

判断node是否为static(有诸多条件)
标记node的children是否为static，若存在non static子节点，父节点更改为static = false
遍历第二轮，标记staticRoot

标记static或节点为staticRoot，这个节点type === 1(一般是含有tag属性的节点)
具有v-once指令的节点同样被标记staticRoot
为了避免过度优化，只有static text为子节点的节点不被标记为staticRoot
标记节点children的staticRoot

要是喜欢的话给我一个start，[github](https://github.com/DIVIBEAR/vue)

感谢[muwoo](https://github.com/muwoo/blogs)提供的思路