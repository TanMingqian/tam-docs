# commit规范

## Angular 规范

![](<../../../.gitbook/assets/image (18) (1).png>)

![](<../../../.gitbook/assets/image (3) (1).png>)

**在 Angular 规范中，Commit Message 包含三个部分，分别是 Header、Body 和 Footer，格式如下：**

```
<type>[optional scope]: <description>
// 空行
[optional body]
// 空行
[optional footer(s)]

```

```
// 示例
fix($compile): couple of unit tests for IE9
# Please enter the Commit Message for your changes. Lines starting
# with '#' will be ignored, and an empty message aborts the commit.
# On branch master
# Changes to be committed:
# ...

Older IEs serialize html uppercased, but IE9 does not...
Would be better to expect case insensitive, unfortunately jasmine does
not allow to user regexps for throw expectations.

Closes #392
Breaks foo.bar api, foo.baz should be used instead
```

### Header

Header 部分只有一行，包括三个字段：type（必选）、scope（可选）和 subject（必选）。

type，它用来说明 commit 的类型。为了方便记忆，我把这些类型做了归纳，它们主要可以归为 Development 和 Production 共两类。它们的含义是：

**Development**：这类修改一般是项目管理类的变更，不会影响最终用户和生产环境的代码，比如 CI 流程、构建方式等的修改。遇到这类修改，通常也意味着可以免测发布。

**Production**：这类修改会影响最终的用户和生产环境的代码。所以对于这种改动，我们一定要慎重，并在提交前做好充分的测试。

![](<../../../.gitbook/assets/image (23) (1).png>)

![](<../../../.gitbook/assets/image (19).png>)

**scope** 是用来说明 commit 的影响范围的，它必须是名词。显然，不同项目会有不同的 scope。

**subject** 是 commit 的简短描述，必须以动词开头、使用现在时。比如，我们可以用 change，却不能用 changed 或 changes，而且这个动词的第一个字母必须是小写。通过这个动词，我们可以明确地知道 commit 所执行的操作。此外我们还要注意，subject 的结尾不能加英文句号。

### Body&#x20;

须要包括修改的动机，以及和跟上一版本相比的改动点。

The body is mandatory for all commits except for those of scope "docs". When the body is required it must be at least 20 characters long.

### Footer&#x20;

主要用来说明本次 commit 导致的后果。在实际应用中，Footer 通常用来说明不兼容的改动和关闭的 Issue 列表

```
BREAKING CHANGE: <breaking change summary>
// 空行
<breaking change description + migration instructions>
// 空行
// 空行
Fixes #<issue number>
```

## 提交频率&#x20;

主要可以分成两种情况。

&#x20;一种情况是，只要对项目进行了修改，一通过测试就立即 commit。比如修复完一个 bug、开发完一个小功能，或者开发完一个完整的功能，测试通过后就提交。

&#x20;另一种情况是，我们规定一个时间，定期提交。这里建议代码下班前固定提交一次，并且要确保本地未提交的代码，延期不超过 1 天。这样，如果本地代码丢失，可以尽可能减少丢失的代码量。

## git rebase&#x20;

git rebase -i 使用 git rebase 命令，-i 参数表示交互（interactive），该命令会进入到一个交互界面中

![](<../../../.gitbook/assets/image (7) (2) (1).png>)

### 修改 Commit Message

```shell
##修改最近一次 commit 的 message
git commit --amend
##修改某次 commit 的 message
git rebase -i：
```

## Commit Message 规范自动化

commitizen-go：使你进入交互模式，并根据提示生成 Commit Message，然后提交。&#x20;

{% embed url="https://github.com/lintingzhen/commitizen-go" %}

commit-msg：githooks，在 commit-msg 中，指定检查的规则，commit-msg 是个脚本，可以根据需要自己写脚本实现。这门课的 commit-msg 调用了 go-gitlint 来进行检查。

&#x20;go-gitlint：检查历史提交的 Commit Message 是否符合 Angular 规范，可以将该工具添加在 CI 流程中，确保 Commit Message 都是符合规范的。&#x20;

{% embed url="https://github.com/llorllale/go-gitlint" %}

gsemver：语义化版本自动生成工具。&#x20;

{% embed url="https://github.com/arnaud-deprez/gsemver" %}

git-chglog：根据 Commit Message 生成 CHANGELOG。

{% embed url="https://github.com/git-chglog/git-chglog" %}
