## Coding

### code/get_commit_hash.py: get bug-related issue -> pull request -> commit hash
```shell
python3 get_commit_hash.py repos.txt
```

it will traverse all repos listed in repos.txt

steps：

1. traverse the repo names
2. git clone the repos
3. through github api, obtain closed pull request related to bug issues
4. filter out prs according to keywords

```python
keywords = ["fix", "defect", "error", "bug", "issue", "mistake", "incorrect","fault", "flaw"]
```

reference：Joshua Garcia, Yang Feng, Junjie Shen, Sumaya Almanee, Yuan Xia, and and Qi Alfred Chen. 2020. A comprehensive study of autonomous vehicle bugs. In Proceedings of the ACM/IEEE 42nd International Conference on Software Engineering (ICSE '20). Association for Computing Machinery, New York, NY, USA, 385–396. https://doi.org/10.1145/3377811.3380397

5.get corresponding commits of the pr

6.write the commit hashes

root_dir: root directory of commit hash files。



### code/search.py: search for the code changes of the commits

pydriller: get commits from a a repo，and resolve the commits to get the changed methods

1. read the repo names
2. traverse the commits
3. search for changed .rs files （ignore files' add/delete）
4. store the changed codes in the level: repo——commit——file——method\_before、method\__after


#### filtering：

- commit's changes are between 6 statements





### code/process.py: generate corpus


- code/difftastic
  - code diff
  
  - transfer diff results to vector: one node to one vector
  
    ```
    [repo, commit_hash, change type, parent node type, grand parent type]
    ```
  
  
    ```
    python3 process.py vector.csv
    ```



### code/cluster_vectors.py

- preprocess the results of process.py，generate feature vector of commits

  - one commit to multiple nodes

    ```
    [change type, parent node type, grand parent type]
    ```

  - dimensions：
    $$
    \rm len(Change\ Type) \times len(Parent\ Node\ Type) \times len(Grandparent\ Node\ Type)
    $$

- clustering

  - ​	HAC




Feature vectors

- change type：
  - insert
  - delete
- context：
  - use TreeCursor to traverse tree-sitter::Tree


common nodes

- field_expression

- arguments

- token_tree

- scoped_identifier

- let_declaration

- block

- non_special_punctuation

- type_arguments

  ```rust
  fn set_outer_position(&self, pos: PhysicalPosition<i32>)
  
  -> Result<(), MatchAccountOwnerError>
  ```


- tuple_struct_pattern： if let/match

  ```rust
  let v = Some(5);
  
  if let Some(5) = v {
      println!("{}", n);
  }
  ```

- match_arm

- macro_invocation

- binary_expression

- expression_statement

- function_item

- reference_item

- meta_item：

  ```rust
  #[derive(Debug, Display)]
  ```

- parameters

- parameter

- meta_arguments

- call_expression

- closure_parameters

  ```rust
  grid.clear(|c| c.reset(&template));
  ```

- tuple_pattern



  
- MatchedPos

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
  
- 3 data structures in Difftastic：

  - Tree node

  - Syntax node （Enum）

    ```rust
    pub enum Syntax<'a> {
        List {
            info: SyntaxInfo<'a>,
            open_position: Vec<SingleLineSpan>, 
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