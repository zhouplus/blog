# git笔记
对于同一个分支，比如master可能你也经常这么做：

1. git pull 拉取你小伙伴写的代码，然后继续写你要写的代码

2. git add <你修改或者增加的文件>

3. git commit

4. 你的习惯可能是尝试push，成功的话就成了，不成功的话再git pull。或者是你根据以往的经验你不知道你的小伙伴在你上次pull之后有没有新的提交并push的远程了，所以你习惯性地使用了git pull 这一步为 了拉取你小伙伴的代码。 使用git pull 或者git pull --rebase

5. 如果第4步你直接使用的是git pull会自动合并代码，但是有冲突的话需要手工合并冲突，然后再commit。如果你第四步使用的是git pull --rebase.则是先把远程的代码下载下来然后本地“重演”你这次所有的提交。重演过程中遇到冲突的话需要手工解决冲突，然后使用git rebase --continue 直至所有提交都成功。

6. push你的代码到远程

## 协同开发

- 代码和环境的一致性


参考技术：
- GitFlow

- GitHub Flow

- GitLabFlow

### 宗旨：对软件架构和服务化的架构的改进 对CI/CD的改进