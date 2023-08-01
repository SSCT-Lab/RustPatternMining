## Coding

### code/get_commit_hash.py 爬取仓库的pull request，从而获取对应的commit hash
```shell
python3 get_commit_hash.py repos.txt
```

使用如上指令，将遍历repos.txt中的所有仓库，获取特定pull request对应的commit hash

获取commit hash流程：

1. 读取仓库名
2. 克隆仓库到本地
3. 使用github cli指令获取所有的closed pull request
4. 获取pull request的title和labels，过滤掉不包含bug相关关键词的pr

```python
keywords = ["fix", "defect", "error", "bug", "issue", "mistake", "incorrect","fault", "flaw"]
```

参考文献：Joshua Garcia, Yang Feng, Junjie Shen, Sumaya Almanee, Yuan Xia, and and Qi Alfred Chen. 2020. A comprehensive study of autonomous vehicle bugs. In Proceedings of the ACM/IEEE 42nd International Conference on Software Engineering (ICSE '20). Association for Computing Machinery, New York, NY, USA, 385–396. https://doi.org/10.1145/3377811.3380397

  5.获取pull request对应的改动commit

  6.将commit hash写入仓库对应的文件

变量root_dir为生成commit hash的根目录。



### code/search.py 根据获取的仓库的commit hash，爬取改动前后的代码

使用如上指令，将遍历repos.txt中的所有仓库

获取仓库改动流程：（使用pydriller工具，该工具可以获取指定仓库的commit，并解析commit以获取其改动的内容，可以细化到method粒度）

1. 读取仓库名
2. 遍历仓库中与后缀名为.rs（即rust源代码文件）的commit
3. 遍历每个commit中改动的.rs文件 （过滤其它类型的文件），并过滤掉改动前/改动后的文件缺失的情况（即文件级别的add/delete的情况）
4. 获取改动前后的method信息（名称、在源文件中的起始行与结束行），并过滤出**在文件改动前后都出现**的method（即过滤掉method被add/delete的情况，仅保留modify）
5. 获取到改动前后的method源代码，并存入文件，文件排布为 仓库——commit——改动文件——method\_改动前、method\_改动后

​	变量root_dir为生成改动代码的根目录。

#### 筛选条件：

- commit改动条数在6行以内
- 针对issue和pull request对应的commit





### code/process.py 生成数据集

对前面获取的每个仓库改动代码文件，

- 使用code/difftastic
  - 进行diff
  
  - 讲diff的结果处理成每个节点的vector形式：每个节点对应一条
  
    ```
    [repo, commit_hash, change type, parent node type, grand parent type]
    ```
  
    写入指定的文件
  
    ```
    python3 process.py vector.csv
    ```



### code/cluster_vectors.py

- 对于process.py生成的内容进行预处理，为每个commit生成对应的vector

  - 每个commit对应多个节点，即多个

    ```
    [change type, parent node type, grand parent type]
    ```

  - 每个commit对应的特征向量具有的维度数：
    $$
    \rm len(Change\ Type) \times len(Parent\ Node\ Type) \times len(Grandparent\ Node\ Type)
    $$

  - 每个维度上的值为commit拥有的对应的节点个数 

- 对所有预处理后的vector进行聚类

  - ​	使用HAC





### code/gen_ast.py 

对前面获取的每个仓库改动代码文件，

使用code/get_tree进行parse，生成对应的AST并以相同的文件排版存储

变量root_dir为生成的AST的根目录。



```shell
code/get_tree/目录下：
cargo build
code/目录下：
python3 gen_ast.py
```



### code/get_tree 生成AST

- tree-sitter::parser::parse()：将源文件转化为tree-sitter::Tree
- tree-sitter::Tree.walk(): 获取TreeCursor用以遍历Tree



#### 生成的AST

**获取的tree-sitter::Tree节点类型：**

- 每个源文件中的内容都是一个function，所以Tree的最顶层的两个节点是
  - source_file
    - function_item
      - fn
      - identifier
      - parameters






### code/gen_diff.py 生成code diff信息

对于获取的每个仓库改动代码文件，使用difft指令生成改动前后代码的diff信息，并重定向写入文件。

```shell
code/目录下：
python3 gen_diff.py
```



### **difftsatic**

```shell
difft --display side-by-side-show-both --context 0 test1.rs test2.rs
```

- --display：显式模式：side-by-side-show-both：显式改动前后的对照

- --context：显式的源代码改动中包含的上下文行数

- 显示的内容：

  - 若前后代码没有语义上的区别，那么会显示：

    ```shell
    test2.rs --- Rust
    No syntactic changes.
    ```

    第一行是输入的后一个的文件路径与源代码所用语言的标识 --- Rust

    第二行是No syntactic changes的提示

  - 若前后代码只有一处改动（TODO：这里判断一处还是多处是由工具中的算法决定的），那么会有形如如下的显示：

    ```shell
    test2.rs --- Rust
    2     let b = 1;                                                     2 
    3                                                                    3 
    4     let s = String::from("23");                                    4     let s = String::from("123");
    .                                                                    5 
    .                                                                    6     let a = 2;
    .                                                                    7 
    5     if (b >= 0){                                                   8     if (b == 0 && b > 1){
    ```

    第一行和前面的一样，后面就是两个文件中的代码不同比较；

    TODO：如何判断是否是同一种变化？

    重定向后写入的文件中 同一行中前后都出现的代码表示前后的改动对应，其他的均为insert/remove
    
  - 若前后代码不止一处改动，显示的内容如下
  
    ```shell
    test1_after.rs --- 1/2 --- Rust
    11         _ => unreachable!("unhandled x500 attr")                                                                                                          11         _ => unreachable!("unhandled x500 attr {:?}", oid)
    
    test1_after.rs --- 2/2 --- Rust
    17         _ => unreachable!("unhandled x500 value type")                                                                                                    17         _ => unreachable!("unhandled x500 value type {:?}", valuety)
    
    ```
  
    比只有一处改动的显示，多了一个当前改动的计数1/n，2/n，...



### difftsatic处理过程

*：仅针对

```shell
difft --display side-by-side-show-both --context 0 test1.rs test2.rs
```

，即比较两Rust的代码源文件

#### 准备工作（略）

- 参数解析
- 检查指令正确性
- 最顶层调用diff_file()获取diff结果
- diff_file()：
  - 读取输入的文件，做检查
  - 调用diff_file_content()
- diff_file_content()
  - 检查
  - 设置语言和config
  - 判断两文件是否相同
  - 检查config
  - 根据config进行parse，调用to_tree_with_limit()获取代码的tree-sitter::Tree
    - 主要输入：源代码
    - 输出：**Tree**
  - 对于获得的两个文件对应的两棵tree，调用to_syntax_with_limit()获取代码的syntax tree，以Vec<&Syntax>
    - 主要输入：两个源代码文件的Tree
    - 输出：**两个源代码文件的Vec<&Syntax>**
    - to_syntax_with_limit()中调用to_syntax()方法，获取Vec<&Syntax>，usize元组，其中usize代表Tree中error的节点个数，这里我们不考虑。
- to_syntax()
  - 检查
  - 调用tree::walk()获取TreeCursor以遍历tree
  - cursor.goto_first_child()方法将cursor指向Tree的root
  - 然后调用all_syntaxes_from_cursor()

- all_syntaxes_from_cursor()
  - 主要输入：cursor
  - 输出：**Vec<&Syntax>**
  - 用一个loop，对空的Vec不断extend()，直到cursor.goto_next_sibling()为空
    - extend中调用用syntax_from_cursor()方法返回一个Option<&Syntax>
- 获取的**Vec<&Syntax>**是DFS前序遍历的结果，并存储了children的关系
  - 继续处理，根据子孙关系设置节点间的前-后关系

- 原本的判断hunk的方式：（格式化后）若改动前或改动后对应的源代码中的两个内容对应的差了一个阈值的行数时，就判定为另一个改动，若在4行以内，则为同一处改动






### 特征向量抽象：

- change type：
  - insert：使用difftsatic获取的的前后改动中 左边没有，右边有
  - delete：使用difftsatic获取的前后改动中 左边有，右边没有
- context：
  - 使用TreeCursor在tree-sitter::Tree中遍历

![1](related work/ref/1.png)

- field_expression：访问对象的field或method：a.foo()和a.foo都是

- arguments：函数调用时的实参

- token_tree：宏中的token

- scoped_identifier： A::b的::

- let_declaration

- block

- non_special_punctuation：标点符号；或是token_tree中的符号（包括::）

- type_arguments：泛型参数/返回值

  ```rust
  fn set_outer_position(&self, pos: PhysicalPosition<i32>)
  
  -> Result<(), MatchAccountOwnerError>
  ```

  上面<>内的i32 Result<>内的()和MatchAccountOwnerError，即尖括号内的内容就是泛型的type_arguments

- tuple_struct_pattern： if let 或 match语句中 

  ```rust
  let v = Some(5);
  
  if let Some(5) = v {
      println!("{}", n);
  }
  ```

  Some(5)就是一个tuple_struct_pattern，说是struct是因为长得像一个struct，

  如果把Some(5)换成(5)，就是一个tuple_pattern，所以说是tuple是因为有()，而如果再去掉括号编程if let 5 = v，就是integer_literal

  而let a = (1, 2); 这种的(1, 2)是一个tuple_expression，如果是let a = Some(5)这种会把Some当作method_call

- match_arm：每个match语句中的一条匹配和其内容

- macro_invocation：整条宏语句，是由一个identifier + 感叹号! + token_tree即括号里()的内容

- binary_expression：二元表达式

- expression_statement：与表达式有关的statement，比如a.foo();  或是一个while for循环，详见其子节点_expression

- function_item： 函数声明

- reference_item： 引用表达式 比如&a，对比reference_type：引用类型，比如&str

- meta_item：

  ```rust
  #[derive(Debug, Display)]
  ```

  整个是一个attribute_item，然后其中[]内算一个meta_item，然后外层的derive是identifier，内层()内的每一个都是一个meta_item

  ()里的test是meta_arguments

- parameters

- parameter

- meta_arguments

- call_expression

- closure_parameters

  ```rust
  grid.clear(|c| c.reset(&template));
  ```

  ||中的内容是closure_parameters

  |c| c.reset(&template)是一个closure_expression

- tuple_pattern



#### 特征向量与hunk间的关系 (一个commit对应一个特征向量时)

- hunk数据结构

  - lhs中涉及到修改的代码**行**  LineNumber的集合

  - rhs中涉及到修改的代码**行** LineNumber的集合

  -  lines：

    ```rust
    Vec<(Option<LineNumber>, Option<LineNumber>)>
    ```

    如果左边是None，就代表右边是新增的；

    如果右边是None，就代表左边被删掉了。

- opposite_to_lhs / opposite_to_rhs 数据结构：

  ```rust
  HashMap<LineNumber, HashSet<LineNumber>>
  ```

  以lhs为例，opposite_to_lhs 是lhs源代码中每一行与rhs源码的行之间的对应关系

  ​	**注意**：只保存了能对应上的，即lhs中的一行能和rhs中的若干行对应上，否则就不会包含这一行 （**e.g.** 左边第3行是删除的，右边没有能对应上的行，那么在HashMap中不会有以第3行为key的内容）；反之同理
  
- MatchedPos数据结构

  ```rust
  pub struct MatchedPos {
      pub kind: MatchKind,
      pub pos: SingleLineSpan,
  }
  pub enum MatchKind {
      UnchangedToken {
          highlight: TokenKind,
          self_pos: Vec<SingleLineSpan>,
          opposite_pos: Vec<SingleLineSpan>,
      },
      Novel {
          highlight: TokenKind,
      },
      NovelLinePart {
          highlight: TokenKind,
          self_pos: SingleLineSpan,
          opposite_pos: Vec<SingleLineSpan>,
      },
      NovelWord {
          highlight: TokenKind,
      },
      Ignored {
          highlight: TokenKind,
      },
  }
  pub struct SingleLineSpan {
      /// All zero-indexed.
      pub line: LineNumber,
      pub start_col: u32,
      pub end_col: u32,
  }
  ```
  
  MatchedPos包含了所有节点的位置信息，以及是否是novel的，但是没有节点名称信息，即Tree中的节点名，也没有Syntax中存节点的字符串，对于MatchKind我们只需要关心UnchangedToken和Novel（未变化的为前者，有变化的为后者）
  
- Change Type	

  - Inserted：遍历hunk中的内容，lines中左边全为None，即新增的全是**行**，显然是Inserted
  - Removed：遍历hunk中的内容，lines中右边全为None，即新删除的的全是**行**，显然是removed
  - Updated：遍历hunk中的内容，lines中左右都不为None，**可能是Updated**
    - TODO：如果某一边部分是None，部分有，怎么判断究竟是哪个？比如删了两行，又加了一行
    - TODO：如果都不为None，是否可以是inserted/removed？ 遍历这一行的MatchedPos，有UnchangedToken，也有Novel的，如何分辨？（比如三个节点，两边都是没变的，中间那个变了，那么显然是Updated）
      - //可以遍历双方MatchedPos，对于UnchangedToken，我们有其对应的内容，对应的内容一定是一样的（因为没变）
      - 查看节点？
  
- 三种数据结构：

  - Tree node
    - 节点名称
    - 字符串
    - 起始行，起始列 （从0开始）
    - 终止行，终止列
    
  - Syntax node （Enum）
    - Syntax info （包括parent、前一个sibling、后一个sibling）
    - 对于List，包含起始字符串（起始行，起始列）、子孙节点、终止字符串
    - 对于Atom，包含（起始行，起始列-终止列）、字符串
    
    ```rust
    pub enum Syntax<'a> {
        List {
            info: SyntaxInfo<'a>,
            open_position: Vec<SingleLineSpan>, // 为什么是Vec？SingleLineSpan只记录一行中的信息，而可能有的节点在源代码中跨行了，所以要存成Vec
            open_content: String,
            children: Vec<&'a Syntax<'a>>,
            close_position: Vec<SingleLineSpan>,
            close_content: String,
            num_descendants: u32,
        },
        Atom {
            info: SyntaxInfo<'a>,
            position: Vec<SingleLineSpan>,
            content: String,
            kind: AtomKind,
        },
    }
    ```
    
  - MatchedPos
    - MatchKind：改动前后完全没变的：UnchangedToken；发生变化的：Novel
    - 所在行、起始列-终止列
  
- 




#### 特征向量与node间的关系

![1](related work/ref/1.png)



- Change Type
  - 输入：所有标记为Novel的节点（可能kind即label不同，可能kind相同但是value不同）
  - 输出：所有标记为Novel节点的change type
    - added
    - deleted
- Type
  - 当前node的父节点

- Context
  - 当前node的祖父节点




- 






## TODO

修复pydriller获取method出错

### 需要进行改动以比较的参数

- commit中修改的代码行数（search.py  LINES_THRESH）
- pull request 对应的commit个数 （fixminer那篇文章中是只考虑了一个commit就修复的bug，并且fixminer中寻找bug fix patch的方法看上去更靠谱一些）
- 将代码改动划分成不同hunk时用于判定不同行的改动是否算成同一处改动的标准
  - 行数：k  （hunks.rs ：MAX_DISTANCE）
    - 比如k = 4时，改动前的第1行对应改动后的2，3行，改动前的第5行对应改动后的4，5行。这样相差了4行，将其判定成两处改动
      反过来 改动前的23对应改动后的1，然后下一块对应改动后的5，它也是判定为两处
  - 更加精确的判断
    - 根据语法树的节点进行判断
- 一个commit对应多少个特征向量
  - 一个commit对应多个hunk，每个hunk对应一个特征向量
  - 一个commit里所有的改动整合成一个向量
- 函数搜索的准确度：直接爬整个文件？






RQ：

- Rust项目中有哪些频繁改动的bug模式？
- 上述模式中哪些是Rust中特有的？







