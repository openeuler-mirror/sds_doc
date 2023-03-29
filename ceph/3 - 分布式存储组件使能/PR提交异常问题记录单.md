# PR提交异常问题记录单
@ceph_yang yangxiaoliang贡献

1. 对于ceph上游社区已知问题，需要提供原生patch信息及链接，方便maintainer检视。
   
   ​        通过git format-patch -1 {commit_id}命令可直接导出社区或自己本地已经commit的patch，其中commit_id为你提供的pr中解决问题的commit或自己fork仓库后向自己仓库的commit。
   
   （1） 本地仓库commit后patch制作。
           ![输入图片说明](https://images.gitee.com/uploads/images/2022/0725/111602_15c77831_1665388.png "image-20220719095749487.png")
        查看patch信息   

    （2）社区patch导出并查看patch信息
   
   ![输入图片说明](https://images.gitee.com/uploads/images/2022/0725/111702_93c5a8bf_1665388.png "image-20220719103024797.png")
   
   ​        社区patch导出后需对比与欧拉社区源码代码行数,例如：使用上游社区源码制作patch后，显示行数修改为第1163行，但欧拉社区版本的代码为第1144行。
   
   ![输入图片说明](https://images.gitee.com/uploads/images/2022/0725/111736_0749e44d_1665388.png "image-20220720140941338.png")
   
   ![输入图片说明](https://images.gitee.com/uploads/images/2022/0725/111803_5f11da35_1665388.png "image-20220720141143748.png")

2. ceph.spec文件修改
   
   ​    修改ceph.spec文件时，需要注意关键信息，如果版本有更新，版本号也需要及时更新。
   
   ![img](https://images.gitee.com/uploads/images/2022/0715/164258_52d534df_1665388.png)

3. 同时需要提交多个PR，需要重新创建分支，否则修改内容会同步当前正在审核的PR中。

4. 签署CLA时需要注意，签署CLA邮件需和gitee账户绑定邮箱一致。

5. 合并多次提交为一次

    **git commit --amend方法：**

   （1）修改文件

   （2）git add .

   （3）git commit --amend 修改提交message

   （4）git push origin master --force

   **git rebase -i {commit_id}方法,可以删除多余的commit提交：**

   （1）查找需要合并的commit id,执行git log

   ![输入图片说明](https://images.gitee.com/uploads/images/2022/0725/111904_9053160b_1665388.png "image-20220720153943379.png")

   （1）执行git rebase -i {commit_id}命令，如：git rebase -i  0942506e03855dd478d4ec3220906a2c7ff62c33

   （2）修改对应信息

   ![输入图片说明](https://images.gitee.com/uploads/images/2022/0725/111934_90605a63_1665388.png "image-20220720154141406.png")

   （3）git push origin master --force需要注意，不加--force可能无法删除

   

   
