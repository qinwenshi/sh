---
layout: post
title: 用 Characterization Testing 来帮助重构遗留代码
category : Coding
tagline: "Supporting tagline"
tags : [代码，重构,遗留代码]
---
# 用 Characterization Testing 来帮助重构遗留代码
---

Michael Feathers 最近又写了[一篇文章](https://michaelfeathers.silvrback.com/characterization-testing)讲述他关于 Characterization Testing 的作用和帮助。最近也在练习在遗留代码中通过增加测试，以让代码可读性提高。

在 Michael 的文章里面，他也澄清了一下“测试”这个让人容易误解的词，从探索式测试、手工测试到单元测试以及其他各种形态的自动化测试。不过有个前提，我们必须得懂我们的那一堆代码到底在做什么。大部分的测试目的是在于告诉我们现在代码的正确度——也就是说，我们测试的目的是为了告诉我们代码是不是真的会按照我们认为的方式运行。而很多时候，问题就在于，我们特么不完全知道代码在做什么。


那么问题来了，如果我们真的就不知道我们的代码到底在做什么怎么办？


我做的练习基本上就是处于这样的状态，看看的代码 [Server.java](https://github.com/qinwenshi/LegacyCode_Server/blob/0c96403e3c48e1c22b383ef01b18456fea43255e/Server.java)


第一眼看到这个代码，大概能猜到它的功能是发送广告邮件，逻辑是遍历给定邮箱的收件箱，逐条回复广告邮件。根据 Michael 的定义，这绝对是名副其实的遗留代码。

Micahel 说先要从写一个叫 x() 的测试开始。先来试试写一个测试，试试能不能把程序运行起来。

~~~ java
@Test
public void x() throws Exception {
	Server.main(new String[]{"a", "b", "c", "d"});;
}
~~~

于是我们看到了一个莫名奇妙的错误

![result_of_x](/image/characterization_testing/result_of_x_case.jpg){:width="450px"}

我们发现原来是因为参数的数量不够，JVM 整个因为这段代码而结束：

![why_exit_1](/image/characterization_testing/why_system_exit_1.jpg){:width="450px"}

得出了第一个知识：如果输入的参数值不够的时候，将会输出一段提示文字，并且系统会退出。我们需要把这条知识记录在测试中。

*讨厌的是，JVM 退出会导致单元测试也提前终止了*

![system_exit_verification](/image/characterization_testing/refactor_extract_method_object.jpg){:width="450px"}

这时我们对系统退出代码做了一次提取方法对象的重构，把新生成的类*Move*到独立的文件中，并改为非static：

~~~ java
	new SystemExit().invoke(1);
~~~

![system_exit_seam](/image/characterization_testing/system_exit_seam.jpg){:width="450px"}

此时再次运行我们的测试x()，确保结果一样。

回到测试中，增加一个 Stub，用于替换 System.exit(1)的逻辑。

![system_exit_stub](/image/characterization_testing/system_exit_stub.jpg){:width="450px"}

再次运行，发现错误跟之前不一样了：

![system_exit_stub_error](/image/characterization_testing/system_exit_stub_error.jpg){:width="450px"}

原来是System.exit(1)逻辑之后，在 if 语句结束后代码会继续运行。这时候，我们的 Stub 其实改变了之前的逻辑了，因此这是一个错误的 Stub。

![fix_system_exit_stub_error](/image/characterization_testing/fix_system_exit_stub.jpg){:width="450px"}

看看结果，貌似跟之前是一样的。这时候我们发现一个问题，我们的单元测试只是检查了 Stub 会丢出异常，而真正的逻辑是系统会退出，并且在错误的流中输出一段文字，我们的这段知识没有在测试中得到体现。同时我们发现 StatusCode 的也需要验证。

	Usage: java Server SMTPHost POP3Host user password EmailListFile CheckPeriodFromName

![result_of_new_stubbing](/image/characterization_testing/result_of_new_stubbing.jpg){:width="450px"}

我们对测试做了进一步的演化：

![system_exit_stubbing_v2](/image/characterization_testing/system_exit_stubbing_v2.jpg){:width="450px"}

代码已经有些长了，进行简单的抽取方法重构：

![system_exit_stubbing_v3](/image/characterization_testing/system_exit_stubbing_v3.jpg){:width="450px"}

此时我们已经对第一点知识比较清楚的了解了：

*我们输入少于6个参数的时候，会收到提示，并且告知系统退出*

因此对测试用例进行一次重命名。

![system_exit_stubbing_v4](/image/characterization_testing/system_exit_stubbing_v4.jpg){:width="450px"}

*第二个测试依然从 x() 开始*


第二个测试用例，我们还是从以 x()命名的探索开始，这时候我们试试传入几个正确的参数看看系统会做点什么事情。为了让它真实运行，我专门注册了一个网易邮箱的小号，看看它到底要搞什么鬼（密码我打上马赛克了:D）。后面的EmailListFile，checkPeriod 和 fromName 暂时还不清楚是做什么用的，先随便填两个值：

![test_with_real_mail_account.jpg](/image/characterization_testing/test_with_real_mail_account.jpg){:width="450px"}

于是得到下面的结果，告诉我们找不到文件 “a”

![test_with_real_mail_account_v1_result.jpg](/image/characterization_testing/test_with_real_mail_account_v1_result.jpg){:width="450px"}

通过错误信息定位到真实代码，发现有一段逻辑，会从给定的这个文件里面读取一个电子邮件列表并且放在一个叫 toList 的成员列表中了。

![reading_email_address_from_file.jpg](/image/characterization_testing/reading_email_address_from_file.jpg){:width="450px"}

为了让测试能够顺利往下运行，我们先给这个文件一个正儿八经的文件名字和一点合理的内容试试：

![add_email_list_file.jpg](/image/characterization_testing/add_email_list_file.jpg){:width="450px"}

同时对测试也做一点微调。

![test_with_real_mail_account_v2.jpg](/image/characterization_testing/test_with_real_mail_account_v2.jpg){:width="450px"}

运行后发现是 SMTP 的一个错误。

![smtp_error.jpg](/image/characterization_testing/smtp_error.jpg){:width="450px"}

我发现这个其实是跟发件服务器的认证机制有关系，网易的 SMTP 服务认证除了用户名密码之外，还要再单开通并设置一个密码，研究了一会之后我发现*这其实跟帮我写通测试关系不是特别大*。

于是我继续回来想办法让测试通过。幸运的是，我找到了一个叫[fakeSMTP](https://nilhcem.github.io/FakeSMTP/)的黑科技小玩意，是 Java 写的，能够模拟出来一个本地运行的 SMTP 服务器，同时会把发出的邮件写入本机的一个目录中。

	sudo java -jar fakeSMTP-2.0.jar
	
运行起来之后是这么个小玩意

![fakeSMTP.jpg](/image/characterization_testing/fakeSMTP.jpg){:width="450px"}

对我们的测试再略做修改：

![test_with_fake_smtp.jpg](/image/characterization_testing/test_with_fake_smtp.jpg){:width="450px"}

运行测试之后再看看，发现这个测试一直在运行，在发件箱里也没有看到什么东西。

![result_of_fake_smtp_v1.jpg](/image/characterization_testing/result_of_fake_smtp_v1.jpg){:width="450px"}

顺着代码往下面看，发现有一个死循环一直在执行。

![infinite_loop_in_code.jpg](/image/characterization_testing/infinite_loop_in_code.jpg){:width="450px"}

还是老办法，先为循环做一个 Seam，便于注入。

![seam_for_infinite_loop.jpg](/image/characterization_testing/seam_for_infinite_loop.jpg){:width="450px"}

Loop 这个类的声明也是通过自动抽取MethodObject 、方法重命名、Move 三次重构完成的。

![looping_object.jpg](/image/characterization_testing/looping_object.jpg){:width="450px"}

这步有点风险，不过因为是编辑器自动完成、编译没有错误，同时运行之前的测试，状态保持一致。*基本上认为本次加入 Seam 的过程是安全的。*

继续在测试中注入一个循环对象：

![inject_looping_object.jpg](/image/characterization_testing/inject_looping_object.jpg){:width="450px"}

运行测试，发现结果依然跟之前一样

![result_of_fake_smtp_v1.jpg](/image/characterization_testing/result_of_fake_smtp_v1.jpg){:width="450px"}

继续找原因，发现在最后部分，有一个让线程休眠的代码，而且一个单位的休眠时间是1分钟。这也是我们的单元测试执行完之后会出现疑似死循环的另一个原因了。

![thread_sleeping.jpg](/image/characterization_testing/thread_sleeping.jpg){:width="450px"}

那么，我们不得不再次为它增加一个 Seam 了。修改后的测试是这样的。

![stop_sleeping_for_long_time.jpg](/image/characterization_testing/stop_sleeping_for_long_time.jpg){:width="450px"}

再次运行之后，发现测试可以通过，而并没有看到有邮件发出去， messages文件夹空空如也。

![smtp_sent.jpg](/image/characterization_testing/smtp_sent.jpg){:width="450px"}

回到的代码，发现有这样一段代码：它检查默认的邮件目录里面是否有邮件，如果没有的话就直接关闭了等候下一轮检查了。

![check_if_any_mail_in_box.jpg](/image/characterization_testing/check_if_any_mail_in_box.jpg){:width="450px"}

为了验证这个理解是否正确，我给自己的邮箱发了一封邮件：

![send_mail_to_self.jpg](/image/characterization_testing/send_mail_to_self.jpg){:width="450px"}

再次运行测试，因为没有进一步校验，测试依然通过，不过我们发现这里多了一封邮件：

![one_mail_sent.jpg](/image/characterization_testing/one_mail_sent.jpg){:width="450px"}

邮件内容是这样的：

![mail_content.jpg](/image/characterization_testing/mail_content.jpg){:width="450px"}

因此我们发现一个新知识 
####当收件箱中有邮件时，程序会发出一封邮件

我们把新发现的知识沉淀到测试中：

![added_new_knowledge_to_verification.jpg](/image/characterization_testing/added_new_knowledge_to_verification.jpg){:width="450px"}

同时我们发现，邮件都是从用户名指定的邮箱发出的，而且是回复给原发件人。这样我们对知识有了进一步了解。

![verify_mail_messages_from_file.jpg](/image/characterization_testing/verify_mail_messages_from_file.jpg){:width="450px"}

我们对第二个测试进行重命名重构

![2nd_test_renaming.jpg](/image/characterization_testing/2nd_test_renaming.jpg){:width="450px"}

有了这几个测试的保护，我们可以开始着手对代码的重构了。

通过代码的逻辑，发现主要部分都是对 JavaMail 的几个类，Session、Folder、Address、Transport 等类型的功能调用。发现隐含着3个 Domain 概念，分别是是 Pop3收件箱，SMTP 发件箱，以及日志的逻辑。


![production_code_refactoring.jpg](/image/characterization_testing/production_code_refactoring.jpg){:width="450px"}


在原来代码中，这样的逻辑重复出现了很多次，因此做了一个类似单例的抽取（因为这里没有多线程，所以没有考虑concurrent的线程安全问题）

~~~ java
if(debugOn){
	System.out.println("xxxxx");
}
~~~

![logger_class.jpg](/image/characterization_testing/logger_class.jpg){:width="450px"}

同时，把打开收件箱、发送邮件的逻辑分别放在Pop3MailBox、SMTPSender进行各自的职责分离。

这个重构基本到此结束。主要学到的：

1. 通过 Michael 提到的Characterization Testing方法，用写测试来学习代码本身复杂的业务知识，并把这些知识在测试中体现出来。
2. 找到代码中的 Seam，如果没有的话想办法用最小的侵入先创建“缝”。这次重构在System.exit(1)、无限循环以及 Thread.sleep 的缝被发现之后就后面的工作开始变得简单了顺理成章了。
3. 不断重复基础的重构，用提取方法、提取类型、重命名、Move 等把疑似相关的代码放在一起，再想办法找到他们对应的 Domain 概念。
4. 先让测试运行起来，在有测试的帮助下，重构的过程中会更安心。

后面还在考虑的是否用真正的单元测试，把准备数据、发送的逻辑隔离。

可以向我打赏

![reward_me_wechat.jpg](/image/characterization_testing/reward_me_wechat.jpg){:width="250px"}
